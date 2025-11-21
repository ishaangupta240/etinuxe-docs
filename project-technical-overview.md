# EtinuxE Technical Deep Dive

## System Overview
- **Backend (`backend/app`)**: FastAPI service that exposes REST routers for admin, auth, dreams, memories, organism state, payments, and support. Domain logic lives in `services.py`, while `models.py` defines all Pydantic schemas shared between routers, background jobs, and the CLI.
- **Frontend (`frontend/src`)**: React + Vite dashboard for admins and internal operators. Pages under `pages/` call the API via `src/api.ts` to visualise organism metrics, manage users, and trigger simulation events.
- **Data layer (`backend/data/*.json`)**: Lightweight JSON stores that emulate a database. `storage.py` provides a thin persistence layer around these files so service functions can read/write atomically.
- **CLI (`backend/app/cli.py`, `cli/`)**: Operator tooling for scripted tasks (e.g. reseeding data, forcing cycles) and a remote CLI that mirrors admin capabilities.

## Marova: Digital Organism Lifecycle
- **State container**: `models.OrganismState` holds numerical gauges for hunger, metabolism, dream energy, dream debt, toxicity, and sleep scheduling metadata alongside the current `OrganismMood`.
- **Rebuild loop**: Every mutation funnels through `services._rebuild_organism_state`. It aggregates fresh metrics from DNA profiles, personality assessments, memory logs, sleep cycles, and DNA tokens, then blends them into the stored state (`services._blend`). This keeps the simulation consistent even if inputs arrive out of order.
- **Event handlers**: High-level actions such as `feed_organism`, `record_memory_log`, `generate_dream`, `run_sleep_cycle`, and `_ensure_auto_sleep_state` live in `services.py`. Each handler updates the organism, persists telemetry, and re-evaluates mood before responding to the API caller or CLI.

## Mood Classification Pipeline
- **Enumerations**: `models.OrganismMood` defines the ten moods surfaced to the UI: `neutral`, `happy`, `excited`, `relaxed`, `sad`, `angry`, `depressed`, `calm`, `agitated`, and `distressed`.
- **Resolver**: `services._resolve_mood` compares the organism's hunger, dream energy, dream debt, toxicity, and hours since sleep against `config.MOOD_THRESHOLDS`. Threshold bins decide the dominant affect; e.g. high dream energy and low toxicity bias toward `happy/excited`, while elevated toxicity + dream debt pushes `distressed`.
- **Telemetry feedback**: Resolved moods are appended to the telemetry log (`services._append_telemetry_entry`) so dashboards can chart shifts over time.

## DNA, Memory, and Personality Interplay
- **DNA profiles (`models.DNAProfile`)** capture medical baselines like respiration and health score. `_aggregate_dna_metrics` influences target hunger, metabolism, and toxicity; frail profiles introduce penalties that make Marova needier and more fragile.
- **DNA tokens (`models.DNAToken`)** act as external dream-energy reserves. `_aggregate_dna_energy` feeds their remaining energy into the rebuild loop, and `generate_dream` draws from them whenever `state.dream_energy` is insufficient.
- **Personality assessments (`models.PersonalityAssessment`)** store per-user emotional vectors plus narrative context. `_aggregate_personality_metrics` and `_derive_assessment_traits` translate those vectors into sensitivity, toxicity resistance, and dream tolerance that are applied to the organism via `_apply_sensitivity_to_state`.
- **Memory logs (`models.MemoryLog`)** include valence, strength, toxicity, and the issuing user. `_aggregate_memory_metrics` distils recent entries into decay-weighted averages that nudge dream energy, toxicity, and dream debt. Positive valence reduces toxicity and restores energy; negative, intense memories do the reverse.

## Personality Assessment Workflow
1. **Ingestion**: Admins or surveys submit `PersonalityAssessment` payloads via `routers/admin.py` or `routers/users.py`.
2. **Trait derivation**: `_derive_assessment_traits` normalises the emotional profile, adjusts for narrative bonus, clamps values, and computes sensitivity, toxicity resistance, and dream tolerance.
3. **State update**: `_update_organism_from_assessment` blends the derived traits into the organism, tweaks hunger/metabolism based on cognitive load, and re-runs `_resolve_mood`.
4. **Persistence + audit**: The updated assessment and organism state are persisted through `storage.write_db`, and telemetry notes the mood impact for UI review.

## DNA Profiling: Tokens and Nodes
- **Profiles** represent static medical traits per user. Only the latest record per user is considered to avoid stale baselines.
- **Tokens** wrap encrypted DNA payloads and track `remaining_energy`; they recharge during sleep cycles (`run_sleep_cycle`) and deplete in dreams.
- **Miniaturization tokens (`models.MiniaturizationToken`)** tie DNA tokens to miniaturization requests. They ensure the organism can borrow energy from a micro-scale clone during recovery events.

## Memory Fabric: Tokens, Nodes, Neighbours
- **Memory tokens (`models.MemoryToken`)** form the reward economy. When `record_memory_log` runs, it mints a token associated with the memory log and applies `_memory_reward` (currently a placeholder) so future tuning can incentivise positive recollection cadence.
- **Memory nodes (`models.MemoryNode`)** organise memories into a graph. Each node stores semantic keywords and up to four neighbours (`services.MEMORY_NEIGHBOR_LIMIT`). Neighbouring links create associative clusters that dreams can traverse.
- **Neighbour maintenance**: When a new memory arrives, `services._update_memory_graph` scores existing nodes for similarity (keyword overlap and recency) and updates forward/backward links to keep the graph balanced.
- **Graph influence**: Dream generation pulls seeds from the memory graph, weighting towards interconnected nodes. Dense positive clusters raise success chance; isolated toxic nodes increase nightmare risk.

## Sleep and Dream Mechanics
- **Auto-sleep**: `_ensure_auto_sleep_state` enforces nightly windows based on `config.AUTO_SLEEP_RULES`. It can trigger `run_sleep_cycle` automatically once the scheduled duration elapses.
- **Dream generation**: `generate_dream` checks sleep phase, energy cost, and memory availability before selecting a dream category. Outcomes invoke `_apply_dream_impacts`, which adjusts hunger, energy, toxicity, and dream debt differently per category.
- **Dream debt**: Tracks overexertion; high values make nightmares likelier and degrade mood until relieved by quality sleep.

## Project Surfaces
- **APIs**: Routers under `backend/app/routers/` expose organism controls, dream requests, memory logging, DNA submissions, personality assessments, insurance, and support workflows.
- **Admin UI**: Components in `frontend/src/pages/Admin*.tsx` reflect the backend data stores, allowing operators to review Marova's health, inject memories, administer tokens, and handle user onboarding.
- **Operator CLI**: `python -m app.cli` offers scripted access to the same endpoints for testing or batch operations.

Use this document with the existing module-specific references (e.g. `docs/organism.md`) to navigate the codebase and reason about how each subsystem shapes Marova's behaviour.
