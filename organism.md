# Organism Systems Reference

This document outlines how Marova's organism simulation works across the backend. It describes each state field, the primary systems that mutate them, and the configuration knobs that tune behaviours. Use this as the canonical guide when adjusting gameplay rules.

## Core Data Structures

### Organism State (`app/models.py::OrganismState`)

| Field | Meaning | Primary Influences |
| ----- | ------- | ------------------ |
| `hunger` | Stored hunger level (0–140). | `feed_organism`, `_rebuild_organism_state`, dream impacts. |
| `metabolism` | Metabolic output (10–140). | Same as above, plus assessments. |
| `mood` | Enum of emotional state. | `_resolve_mood` after feeds, assessments, dreams, state rebuild. |
| `last_feed` | UTC timestamp of last consumption. | `feed_organism`, assessments. |
| `dream_energy` | Energy available for dreams (0–12). | `_rebuild_organism_state`, sleep cycles, dreams, memory logs. |
| `dream_debt` | Pressure to dream (0–10). | `_rebuild_organism_state`, dreams, sleep cycles. |
| `toxicity_level` | Accumulated corruption (0–100). | `_rebuild_organism_state`, memory logs, dreams, sleep cycles. |
| `sleep_phase` | Qualitative sleep phase string. | `_rebuild_organism_state`, `run_sleep_cycle`, `generate_dream`. |
| `last_sleep` | Timestamp of last completed sleep cycle. | `run_sleep_cycle`. |
| `sensitivity_threshold` | Reactivity to inputs (0–1). | `_apply_sensitivity_to_state`, `_rebuild_organism_state`. |
| `toxicity_resistance` | Shield against toxicity (0–1). | Assessments, `_rebuild_organism_state`. |
| `dream_tolerance` | Comfort with intense dreams (0–1). | Assessments, `_rebuild_organism_state`. |
| `auto_sleep_enabled` | Toggles nightly auto-sleep routine. | Configuration, `_ensure_auto_sleep_state`. |
| `sleep_schedule_hour` | Scheduled bedtime hour (UTC, 0–23). | Config defaults, admin tuning. |
| `wake_schedule_hour` | Scheduled wake hour (UTC, 0–23). | Derived from schedule + duration. |
| `sleep_duration_hours` | Target nightly sleep span. | Config defaults, admin tuning. |
| `sleep_session_started_at` | Timestamp when the current auto-sleep began. | `_ensure_auto_sleep_state`. |
| `sleep_session_ends_at` | Scheduled wake timestamp for the active session. | `_ensure_auto_sleep_state`. |

### Supporting Models

- `MemoryLog`, `MemoryNode`, `MemoryToken`: represent recorded memories, their graph, and reward economy.
- `SleepCycleRecord`: historical sleep sessions, used to calculate recovery averages.
- `DNAToken`: external energy reserves for dream generation.
- `PersonalityAssessment`, `DNAProfile`: human data used to steer the organism.

## Configuration Anchors (`app/config.py`)

- `MOOD_THRESHOLDS`: Bins for energy, toxicity, hunger, and sleep debt when resolving moods.
- `DREAM_RULES`: Base dream success probability, toxicity bands, and per-category stat deltas.
- `MEMORY_RULES`: Coefficients governing toxicity changes and energy gains when logging memories.
- `AUTO_SLEEP_RULES`: Default bedtime, duration, and quality for the nightly auto-sleep cycle.
- Clamp constants: `HUNGER_MAX`, `METABOLISM_MAX`, `TOXICITY_MAX`, `DREAM_DEBT_MAX`, `MAX_DREAM_ENERGY`, `MAX_DNA_TOKEN_ENERGY` (in `services.py`).

Tweaking any of these values allows balancing without direct code edits.

## Systems Overview

### State Reconstruction (`_rebuild_organism_state`)

This is the heart of the simulation. Every time the organism is persisted, the function recalculates target values from historical data and eases current readings toward them.

Key inputs:

- **DNA metrics** (`_aggregate_dna_metrics`): respiration, energy consumption, health scores.
- **Personality metrics** (`_aggregate_personality_metrics`): emotional scores, intensity, and behavioural thresholds.
- **Memory metrics** (`_aggregate_memory_metrics`): last 30 memory logs, giving average valence/strength/toxicity and a decaying "recent toxicity" value.
- **Sleep metrics** (`_aggregate_sleep_metrics`): rolling averages of duration/quality plus hours since last cycle and last sleep.
- **DNA energy metrics** (`_aggregate_dna_energy`): current token reserves.

Targets and effects:

- `hunger_target`: combines hours since feed, DNA energy demand, respiration ratio, health penalty, memory strength, and sleep quality.
- `metabolism_target`: blends respiration, energy demand, health, and memory strength.
- `toxicity_target`: scales memory toxicity by resistance, then adds health penalty and DNA condition.
- `dream_energy_target`: rewards dream tolerance, DNA energy, positive emotion, sleep quality; penalises memory toxicity and existing dream debt.
- `dream_debt_target`: grows with poor sleep, negative emotion, recent toxicity; shrinks with DNA energy.
- `sleep_phase`: derived from sleep history but honours the "sleeping" flag for a short window after sleep cycles.
- **Mood resolution**: `_resolve_mood` inspects current energy, toxicity, hunger, dream debt, and hours since sleep to pick from `happy`, `excited`, `relaxed`, `sad`, `angry`, `depressed`, `calm`, `agitated`, `distressed`, or `neutral`.

Blending uses `_blend` for smooth transitions.

### Feeding (`feed_organism`)

- Reduces `hunger` by up to 10 based on volume and intensity.
- Increases `metabolism` through sensory intensity and motion.
- Updates `last_feed` and recomputes mood via `_resolve_mood` with current sleep metrics.

### Memory Logging (`record_memory_log`)

1. Validates cadence (minimum 1 hour per user) and normalises timestamps.
2. Records `MemoryLog` and associated `MemoryToken` (reward via `_memory_reward` placeholder) and `MemoryNode` with neighbours.
3. Applies toxicity change:
   - `base_push = toxicity * base_push_coeff`.
   - `affect = valence * strength`; positive affect yields relief, negative affect adds a penalty.
   - Resistance reduces intake via `intake_scale = 1 - resistance * weight`.
   - Net delta updates `state.toxicity_level`, clamped.
4. Increases `dream_energy` based on positive valence and strength using `MEMORY_RULES['energy_gain']`.
5. Persists the organism and memory data.

Result: happy, strong memories cleanse toxicity; bitter memories worsen it.

### Automatic Sleep Schedule (`_ensure_auto_sleep_state`)

- Nightly sleep is scheduled for 23:00 UTC with a fixed seven-hour window (configurable via `AUTO_SLEEP_RULES`).
- When the current time crosses into the window, the organism enters the `sleeping` phase, and `sleep_session_started_at` / `sleep_session_ends_at` are populated.
- At or after the scheduled wake timestamp, a sleep cycle is executed automatically (`SleepCycleRequest`) to regenerate energy, detox toxicity, and log the session before returning the organism to `awake`.
- `wake_schedule_hour` is derived from the bedtime and duration to support UI display of the sleep plan.
- The helper runs ahead of every major state mutation, so sleep initiates or ends as soon as any backend interaction occurs after the threshold.

### Dream Generation (`generate_dream`)

- Requires active memory logs/nodes and tokens (unless forced).
- Computes combined energy (organism + DNA) vs base cost.
- Success probability: `base_probability` (0.6 by default) tweaked by average seed valence/toxicity, clamped 5–95%. Force requests set it to 1.0.
- On success: `_select_dream_category` picks `happy`, `neutral`, or `nightmare` based on current `state.toxicity_level` and the configured bands (unless a preferred category inside the allowed set is supplied).
- Effects applied by `_apply_dream_impacts`:
  - **Nightmare**: drains 2.13 energy, adds 4 toxicity, increases hunger, bumps dream debt, and may trigger an abrupt wake if toxicity overshoots the high band.
  - **Happy**: boosts energy, reduces toxicity by 3.5, drops metabolism, cuts dream debt.
  - **Neutral**: small energy gain, stable toxicity, mild metabolism cooldown, light dream debt relief.
- Failure raises dream debt slightly and marks the organism "restless".
- Dreams now require the organism to be inside an active sleep window (`sleep_phase == "sleeping"`); attempts outside of sleep are rejected.
- Energy is consumed from organism reserves first, then DNA tokens.
- Dream records include glyphs/effects (vary by category).

### Sleep Cycles (`run_sleep_cycle`)

- Marks the organism as `sleeping`, restores energy (`duration_hours * (0.4 + quality)`) and detoxifies toxicity proportionally.
- Abrupt wakes reduce benefits and increase dream debt.
- Detox also seeps into memory nodes, lowering their toxicity.
- DNA tokens regenerate energy in parallel.
- A `SleepCycleRecord` is appended, metrics are recomputed, and the organism persists with fresh mood via `_resolve_mood` during the next rebuild.

### Personality Assessments (`_update_organism_from_assessment`)

- Converts assessment emotional profile into `emotion_score` and `intensity`.
- Adjusts metabolism/hunger for cognitive load and intensity.
- Updates threshold fields (`sensitivity_threshold`, `toxicity_resistance`, `dream_tolerance`) through `_apply_sensitivity_to_state`.
- Re-evaluates mood using current sleep metrics, then persists.

### DNA Tokens (Energy Reservoirs)

- `DNAToken.remaining_energy` supplements dream energy when organism reserves are insufficient.
- Regenerated during sleep cycles; decreased during dream generation.
- `_aggregate_dna_energy` feeds their metrics into state rebuild for dream energy targeting.

## Factor Summary

| Factor | Source | Influences |
| ------- | ------ | ---------- |
| Memory valence/strength/toxicity | User memory logs | Toxicity level, dream energy, dream success probability, dream category. |
| Sleep quality & duration | Sleep cycles history | Dream debt, dream energy, hunger target, sleep phase, mood. |
| DNA profiles (respiration, energy, health) | User DNA data | Hunger/metabolism targets, health penalties, dream energy target. |
| Personality assessments | Emotional and threshold scores | Mood resolver inputs, sensitivity/toxicity resistance/dream tolerance. |
| DNA tokens | Miniaturization system | Supplemental dream energy, offsets dream debt targets. |
| Dream category outcomes | Dream generation | Energy, toxicity, hunger, metabolism, dream debt, sleep phase. |

## Operational Flow

1. **User interacts** (feed, memory, dream, sleep) → corresponding service mutates immediate state.
2. `_persist_organism_state` always rebuilds full organism state before writing to disk.
3. The frontend and admin dashboards query `/organism/state`, `/dreams`, `/memories`, etc., to display current metrics.
4. Tunable rules live in `config.py`; change them here to rebalance behaviours without hunting through logic.
