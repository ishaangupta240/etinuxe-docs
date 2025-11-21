# Computational Factors Reference

This note catalogs the core formulas that drive Etinuxe's onboarding, pricing, and wellness calculations. Symbols are documented alongside, and any constants are referenced with their defaults from `backend/app/config.py`.

---

## Miniaturization Pricing

**Purpose:** Compute the one-time charge for an individual miniaturization request.

- Inputs:
  - `scale` — desired final scale (0 < scale ≤ 1)
  - `scale_step` — permitted reduction increment (`settings.scale_step`)
  - `pricing_per_step` — base cost per reduction step (`settings.pricing_per_step`)
- Derived values:
  - `reduction = max(0, 1 - scale)`
  - `steps = ceil(reduction / scale_step)`
  - `cost_usd = steps * pricing_per_step`

A larger reduction (smaller target scale) increases `steps` and therefore the total price.

---

## Insurance Premiums & Discounts

**Purpose:** Quote or settle subscription insurance tied to a miniaturization request.

- Inputs:
  - `scale` — request scale (same as above)
  - `tier` — selected insurance tier (`basic`, `plus`, `premium`, `ultra`)
  - `settings.insurance_pricing.<tier>` — base rate per 1% reduction unit (USD)
  - `settings.health_bucket_multipliers.<bucket>` — multiplier per health bucket (`good`, `normal`, `unhealthy`, `extremely_unhealthy`)
  - `settings.points_discount` — loyalty redemption rules (`points_per_discount_unit`, `discount_per_unit`)
  - `available_points` — unspent memory points for the user
- Derived values:
  - `units = max(1, ceil((1 - scale) / 0.01))`
  - `base_rate = insurance_pricing.<tier>`
  - `bucket_multiplier = health_bucket_multipliers.<user.health_bucket>`
  - `monthly_before_multiplier = units * base_rate`
  - `monthly_premium = monthly_before_multiplier * bucket_multiplier`
  - Points redemption:
    - `affordable_units = floor(available_points / points_per_discount_unit)`
    - `max_units_by_cost = floor(monthly_premium / discount_per_unit)`
    - `redemption_units = min(affordable_units, max_units_by_cost)`
    - `points_spent = redemption_units * points_per_discount_unit`
    - `discount_amount = redemption_units * discount_per_unit`
  - `final_premium = max(0, monthly_premium - discount_amount)`

A scheduled policy defers activation until the next billing moment of an active plan; otherwise it activates immediately and may also log a payment equal to `final_premium`.

---

## Health Score & Bucket

**Purpose:** Evaluate baseline health from the intake survey.

Starting baseline: `score = 40`

Adjustments are additive, with caps enforced at 0–100 before rounding.

- Sleep:
  - Optimal (7–9 h) → `+15`
  - Near-optimal (6–7 h or 9–10 h) → `+10`
  - Slightly off (5–6 h or 10–11 h) → `+5`
  - Otherwise → risk `sleep_deficit`
- Activity (`exercise_minutes_per_week`): add `18 * min(1, minutes / 210)`; flag `low_activity` when below 105 minutes.
- Diet quality (1–5 scale): add `16 * ((diet_quality - 1) / 4)`; low values set `dietary_risk`.
- Stress (1–5 scale): add `12 * ((6 - stress_level) / 5)`; high values set `elevated_stress`.
- Chronic condition: `-10` if present, `+2` otherwise.
- Alcohol (`units_per_week`):
  - ≤7 → `+4`
  - ≤14 → `+1` & hint message
  - >14 → `-6` & risk `alcohol_load`
- Smoking: `-12` & risk `tobacco_exposure` if true, else `+3`.
- Mindfulness (`meditation_minutes_per_week`): add `6 * min(1, minutes / 180)`.
- Hydration (`liters_per_day`):
  - ≥2.5 → `+6`
  - ≥1.5 → `+3`
  - <1.5 → risk `low_hydration`

After adjustments:

- `final_score = clamp(round(score), 0, 100)`
- Bucket assignment:
  - `<20` → `extremely_unhealthy`
  - `<60` → `unhealthy`
  - `<80` → `normal`
  - `≥80` → `good`

The function also returns rich textual feedback (summary and risk codes) stored alongside the DNA profile.

---

## Dream & Organism Metrics (Highlights)

The organism state engine recalculates internal metrics from surveys, memory logs, and sleep data. Core relationships include:

- **Hunger target**
  - `hours_since_feed = hours(now - last_feed)` (default 24)
  - `hunger_target = clamp(hours_since_feed * (1.1 + energy_demand * 0.9 + respiration_ratio * 0.4)
    + health_penalty * 45 + avg_memory_strength * 8 - avg_sleep_quality * 12,
    0, HUNGER_MAX)`
  - `energy_demand = avg_energy_consumption`
  - `respiration_ratio = avg_respiration / 12`
  - `health_penalty = max(0, 1 - avg_health_score / 100)`
- **Metabolism target**
  - `metabolism_target = clamp(avg_respiration * 5.5 + energy_demand * 18 + (1 - health_penalty) * 14 + avg_memory_strength * 20,
    10, METABOLISM_MAX)`
- **Dream energy target**
  - `dream_energy_target = clamp(2.5 + dream_tolerance * 4 + dna_energy_total * 0.05
    + positive_emotion * 1.5 + avg_sleep_quality * 3
    - avg_memory_toxicity * 2 - dream_debt * 0.6,
    0, MAX_DREAM_ENERGY)`

The live state blends current values toward these targets using exponential smoothing (blend factors 0.25–0.4) and records telemetry snapshots when deviations are significant.

---

## OTP Regeneration & Verification

- Initial signup: `OTP = six-digit random (uuid4 integer % 1_000_000)` with expiry `now + OTP_EXPIRY_MINUTES`.
- Resend request or login of unverified account:
  - Mark older OTPs consumed.
  - Generate new code and expiry following same formula.
  - Persist `otp_id` on the user record and dispatch via emailer.

Verification succeeds when a matching, unconsumed code with future expiry is found; on success the user transitions to `status = verified` and `current_stage = verified`.

---

## Miniaturization Stage Progression

Stage transitions are triggered by service calls:

1. Signup → `signup`
2. OTP verified → `verified`
3. Health intake update → (stage unchanged, but health score updated)
4. Miniaturization request submission → `request_submitted`
5. Payment capture → `payment_captured`
6. Personality assessment → `assessment_complete`
7. Token issuance → `awaiting_procedure`
8. Administrative approval (outside onboarding) → `miniaturized`

Each transition updates `user.updated_at`, ensuring the frontend can resume onboarding from the latest step.

---

## Loyalty Points Spending

- Each memory log awards tokens; redeemable balance sums unspent token amounts.
- Spending (e.g., insurance discount) consumes tokens FIFO by creation time.
- Partial consumption splits tokens, leaving residual balances available for future use.

---
