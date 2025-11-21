# Insurance Pricing Logic

## Key Concepts

- **Scale**: The target miniaturization scale (e.g. `0.001`). Scale values are converted into pricing units to size the policy relative to how small the candidate will become.
- **Insurance unit size**: Every `0.01` of scale costs one unit. Units are rounded up with a minimum of one unit, so even tiny scales still carry a base premium.
- **Tier base rates**: Each tier (Basic, Plus, Premium, Ultra) has a base monthly cost per unit defined in `backend/data/settings.json` under `insurance_pricing`.
- **Health bucket multiplier**: The user's health bucket adjusts pricing via the multipliers in `health_bucket_multipliers`. Less healthy users pay higher premiums.
- **Memory points**: Unspent memory tokens award points that can discount the premium when redemption rules allow it.

## Premium Calculation

All premium math happens inside `_calculate_insurance_pricing` (see `backend/app/services.py`). The steps are:

1. **Translate scale to units**
   ```text
   units = ceil((1 - scale) / 0.01)
   ```
   Example: a request with scale `0.087` yields `92` units, while a larger scale such as `0.45` produces `55` units. Smaller target bodies therefore carry higher premiums.

2. **Apply tier base rate**
   ```text
   monthly_before_multiplier = units * tier_base_rate
   ```
   Base rates (defaults from `settings.json`):
   - Basic: `20` USD per unit
   - Plus: `30` USD per unit
   - Premium: `60` USD per unit
   - Ultra: `80` USD per unit

3. **Adjust for health bucket**
   ```text
   monthly_premium = monthly_before_multiplier * bucket_multiplier
   ```
   Multipliers (defaults):
   - Good: `1.0`
   - Normal: `1.2`
   - Unhealthy: `1.7`
   - Extremely Unhealthy: `2.4`

4. **Redeem memory points (optional)**
   - Every full `points_per_discount_unit` (default `10000` points) unlocks a `discount_per_unit` (default `10` USD) reduction.
   - Points redemption is capped by both points on hand and the premium amount so charges never go negative.
   - During scheduling scenarios (upgrades while another policy is active) the preview still shows hypothetical points spent, but actual redemption occurs only when the new policy becomes active.

5. **Final premium**
   ```text
   final_premium = max(0, monthly_premium - discount_amount)
   ```

Returned metadata also tracks points spent, available points, and the bucket multiplier so the frontend can display the breakdown.

## Activation Behaviour

- **Initial signup**: The signup flow captures a preferred tier (defaults to Basic). Once the user's miniaturization request completes, `_auto_activate_initial_insurance` immediately issues that tier with an effect1ive date matching the completion time.
- **Subsequent upgrades/downgrades**: When a user changes tiers via `/account/insurance`, the system schedules the new policy to take effect on the next billing date if an active policy is present. Otherwise the activation is immediate.
- **Scheduled swaps**: If a scheduled policy already exists for the request, the newer selection cancels the previous scheduled record before writing the replacement.

## Configuring Pricing

Adjust the following keys in `backend/data/settings.json` to tune insurance behaviour:

- `insurance_pricing`: Per-tier base rate per unit.
- `health_bucket_multipliers`: Multipliers by health bucket value.
- `points_discount.points_per_discount_unit`: Points needed for each discount step.
- `points_discount.discount_per_unit`: Dollar amount reduced per discount step.
