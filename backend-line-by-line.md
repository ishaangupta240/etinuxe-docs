# Backend Companion Guide

A quick walkthrough of the backend modules and the most important code paths that drive Marova. Instead of line numbers, this guide focuses on intent and key snippets you can lift straight into a presentation.

## Module Map

### `app/main.py`
- Enables postponed annotations (`from __future__ import annotations`).
- Instantiates the FastAPI app, adds permissive CORS, and mounts routers for auth, admin, dreams, memories, organism, and support.
- Exposes a lightweight `/` health check returning `{"status": "ok"}`.

### `app/config.py`
- Declares tuning knobs such as `MOOD_THRESHOLDS`, `DREAM_RULES`, `MEMORY_RULES`, and `AUTO_SLEEP_RULES`.
- Collects override-ready dictionaries so behaviour can be tweaked without touching code.

### `app/models.py`
- Pydantic schemas for everything: users, admins, DNA profiles, personality assessments, memory logs, tokens, dreams, sleep cycles, insurance policies, organism state, and CLI payloads.
- Scalar aliases (`PositiveFloat`, `UnitIntervalFloat`, etc.) keep validation readable.

### `app/services.py`
- Home of every simulation rule. Utility helpers (_clamp/_average/_blend), aggregators, state rebuild pipeline, dream/memory flows, insurance helpers, and CLI support.
- Reads and writes JSON storage via `storage.py`, always recalculating organism state before persisting.

### `app/routers/*`
- Thin FastAPI routers that deserialize requests, call the matching service function, and re-serialize responses with `model_dump`.
- Separation keeps HTTP details away from the simulation logic.

### `app/cli.py`
- Typer-powered console for admins: inspect organism vitals (`code-state`), view telemetry (`code-stats`), and toggle auto-sleep (`code-auto-sleep`).

### `storage.py`
- JSON persistence shim. `read_db()` loads all datasets, `write_db()` writes them back atomically, and helpers patch individual collections safely.

### Supporting Modules
- `emailer.py` mocks outbound email (OTP, reset links) and can be swapped for a real provider.
- `timeutils.py` centralises IST conversions to keep timestamps consistent.
- `config.py`, `models.py`, and `services.py` form the configuration → schema → behaviour triangle.

## Key Implementation Snippets

### Password Hashing and OTP Generation
```python
def _generate_otp() -> str:
	return f"{uuid.uuid4().int % 1_000_000:06d}"

def _hash_password(secret: str) -> str:
	return hashlib.sha256(secret.encode()).hexdigest()

def _password_matches(expected_hash: str, candidate: str) -> bool:
	return bool(expected_hash) and hmac.compare_digest(expected_hash, _hash_password(candidate))
```
Six-digit OTP codes come from UUID entropy. Passwords are hashed with SHA-256, and comparisons use `hmac.compare_digest` to avoid timing leaks.

### OTP Issued During Signup
```python
otp = OTPRecord(
	user_id=user.id,
	code=_generate_otp(),
	expires_at=now + timedelta(minutes=config.OTP_EXPIRY_MINUTES),
)
user.otp_id = otp.id

emailer.send_signup_otp(user, otp)

db.setdefault("users", []).append(user.model_dump(mode="json"))
db.setdefault("otp_store", []).append(otp.model_dump(mode="json"))
```
Creating a user immediately mints an OTP, emails it, and records it in the JSON store for later verification.

### Evaluating Health Surveys and DNA Profiles
```python
def _evaluate_health_survey(survey: HealthSurvey) -> Tuple[int, HealthBucket, str, List[str]]:
	score = 40.0
	insights: List[str] = []
	risks: List[str] = []

	if 7.0 <= survey.sleep_hours <= 9.0:
		score += 15.0
		insights.append("Sleep duration sits within the optimal 7-9 hour band.")
	else:
		risks.append("sleep_deficit")
		insights.append("Significant sleep disruption detected; prioritise restorative rest.")

	activity_ratio = min(1.0, survey.exercise_minutes_per_week / 210.0)
	score += 18.0 * activity_ratio
	if activity_ratio < 0.5:
		risks.append("low_activity")

	# ...diet quality, stress, chronic conditions, alcohol, smoking, mindfulness, hydration...

	score = _clamp(score, 0.0, 100.0)
	final_score = int(round(score))
	bucket = _health_bucket(final_score)
	summary = " ".join(insights)
	return final_score, bucket, summary, sorted(set(risks))
```
Survey inputs become a 0–100 health score, qualitative insights, and risk tags. When a survey exists, the new DNA profile snapshot is written with these results so the organism can consume them.

```python
def _aggregate_dna_metrics(profiles: List[DNAProfile]) -> Dict[str, float]:
	latest_profiles = _latest_by_user(profiles, lambda profile: profile.user_id, lambda profile: profile.updated_at)
	avg_respiration = _average([profile.respiration_rate for profile in latest_profiles], 12.0)
	avg_energy = _average([profile.energy_consumption for profile in latest_profiles], 1.0)
	avg_health = _average([float(profile.health_score) for profile in latest_profiles], 60.0)
	medical_notes = sum(1 for profile in latest_profiles if profile.medical_history)
	condition_ratio = medical_notes / len(latest_profiles) if latest_profiles else 0.0
	return {
		"avg_respiration": avg_respiration,
		"avg_energy_consumption": avg_energy,
		"avg_health_score": avg_health,
		"profile_count": float(len(latest_profiles)),
		"condition_ratio": condition_ratio,
	}
```
These metrics flow into the state rebuild loop to adjust hunger, metabolism, and toxicity targets.

### Personality Assessment → DNA Token
```python
payload, _ = _derive_assessment_traits(payload)

checksum_source = f"{payload.user_id}{payload.emotional_profile}{payload.narrative}{_now().isoformat()}"
checksum = hashlib.sha256(checksum_source.encode()).hexdigest()

emotional_profile = payload.emotional_profile or {}
emotional_intensity = _average([abs(value) for value in emotional_profile.values()], 0.0)
dynamic_energy = 4.0 + emotional_intensity * 3.0
dynamic_energy += (payload.dream_tolerance or 0.5) * 3.0
dynamic_energy += (payload.toxicity_resistance or 0.5) * 1.5

dna_token = DNAToken(
	id=uuid.uuid4(),
	user_id=payload.user_id,
	payload_checksum=checksum,
	encrypted_blob=hashlib.sha256(checksum.encode()).hexdigest(),
	created_at=_now(),
	remaining_energy=round(_clamp(dynamic_energy, 3.0, MAX_DNA_TOKEN_ENERGY), 3),
)

db.setdefault("personality_assessments", []).append(payload.model_dump(mode="json"))
db.setdefault("dna_tokens", []).append(dna_token.model_dump(mode="json"))
```
Every assessment produces a checksumed DNA token whose energy reserve scales with emotional intensity, dream tolerance, and toxicity resistance.

### Personality Traits Applied to the Organism
```python
def _update_organism_from_assessment(db: Dict[str, Any], assessment: PersonalityAssessment) -> None:
	state = _ensure_auto_sleep_state(db)
	emotion_values = list(assessment.emotional_profile.values())
	intensity = sum(abs(value) for value in emotion_values) / len(emotion_values) if emotion_values else 0.0

	state.metabolism = min(100.0, state.metabolism + max(1.0, intensity * 5))
	state.hunger = max(0.0, state.hunger - max(2.0, intensity * 4))
	state.last_feed = _now()
	state = _apply_sensitivity_to_state(state, assessment)

	sleep_cycles = [SleepCycleRecord.model_validate(item) for item in db.get("sleep_cycles", [])]
	sleep_snapshot = _aggregate_sleep_metrics(sleep_cycles, state.last_sleep)
	state.mood = _resolve_mood(state, sleep_snapshot)
	_persist_organism_state(db, state)
```
Assessments immediately retune hunger/metabolism, update trait sliders, and trigger a fresh mood resolution.

### Memory Logging, Tokens, and Toxicity Feedback
```python
reward = _memory_reward(payload.valence, payload.strength, payload.toxicity)
log = MemoryLog(
	id=uuid.uuid4(),
	user_id=user_id,
	timestamp=timestamp,
	valence=payload.valence,
	strength=payload.strength,
	toxicity=payload.toxicity,
	embedding=payload.embedding,
	memory_text=body,
	tokens_awarded=reward,
)

token = MemoryToken(
	id=uuid.uuid4(),
	user_id=user_id,
	log_id=log.id,
	amount=reward,
)

base_push = payload.toxicity * config.MEMORY_RULES["toxicity"]["base_push"]
affect = payload.valence * payload.strength
relief = max(0.0, affect) * config.MEMORY_RULES["toxicity"]["positive_relief"]
penalty = max(0.0, -affect) * config.MEMORY_RULES["toxicity"]["negative_penalty"]
intake_scale = max(0.0, 1.0 - state.toxicity_resistance * config.MEMORY_RULES["toxicity"]["resistance_weight"])
toxicity_delta = (base_push + penalty) * intake_scale - relief
state.toxicity_level = _clamp(state.toxicity_level + toxicity_delta, 0.0, TOXICITY_MAX)

energy_gain = max(0.0, payload.valence) * config.MEMORY_RULES["energy_gain"]["positive_valence"]
energy_gain += payload.strength * config.MEMORY_RULES["energy_gain"]["strength"]
state.dream_energy = _clamp(state.dream_energy + energy_gain, 0.0, MAX_DREAM_ENERGY)
```
Every memory uplink mints a spendable token, grows the memory graph, and nudges toxicity/dream energy according to the emotional payload.

### Dream Generation and DNA Energy Drawdown
```python
combined_energy = state.dream_energy + sum(token.remaining_energy for token in active_dna_tokens)
base_energy_cost = _clamp(0.6 + avg_strength + avg_toxicity * 0.5, 0.3, 4.0)

state_energy_used = min(state.dream_energy, total_energy_cost)
state.dream_energy = _clamp(state.dream_energy - state_energy_used, 0.0, MAX_DREAM_ENERGY)

dna_energy_used = 0.0
for token in active_dna_tokens:
	available = max(0.0, token.remaining_energy)
	draw = min(available, remaining_cost)
	token.remaining_energy = round(max(0.0, available - draw), 6)
	dna_energy_used += draw
	remaining_cost -= draw
	if remaining_cost <= 0:
		break

if success:
	_apply_dream_impacts(state, category)
else:
	state.dream_debt = _clamp(state.dream_debt + 0.3, 0.0, DREAM_DEBT_MAX)

db["dna_tokens"] = [token.model_dump(mode="json") for token in dna_tokens]
_persist_organism_state(db, state)
```
Dreams tap organism energy first, then draw from DNA tokens. Outcomes feed back into mood, energy, toxicity, and dream debt.

### Admin & Support Utilities
- Payments, insurance policies, and support sessions reuse the same storage helpers while applying domain-specific validations (e.g., payment must cover quoted cost; support updates respect session status transitions).
- `list_dna_tokens`, `list_memory_logs`, and similar helpers power both API responses and CLI commands.

## Using This Guide
- Pair the module map with the code snippets to explain behaviour during demos.
- Cross-reference `project-technical-overview.md` for broader context, then drill into the snippets here when you want to showcase exact logic.
- Because persistence is pure JSON, everything shown above can be traced directly in `backend/data/*.json` when stepping through scenarios.
