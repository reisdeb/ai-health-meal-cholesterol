# AI-powered Meal Analysis Cholesterol App

## SECTION A — Executive Summary

- Start with a **narrow, high-value Phase 1 use case**: estimate saturated fat per meal for users managing high cholesterol, rather than attempting broad nutrition analysis on day one.
- Use a **bounded workflow** with distinct stages—input validation, meal inference, output safety review, and state update—because it is easier to evaluate, safer to ship, and simpler for an engineering team new to production AI.
- Treat the image-based estimate as an **uncertainty-aware nutritional estimate**, not as ground truth. The product should explicitly communicate ambiguity, especially for mixed meals, hidden ingredients, and unclear portions.
- Rely on **multimodal image understanding where it adds clear value**—food recognition and portion cues—but pair it with deterministic checks, structured outputs, and schema validation to keep the system predictable.
- Make **safety a first-class requirement** in both prompt design and orchestration: reject unusable or non-food images, strip medical advice, avoid treatment recommendations, and keep responses informational only.
- Design with **evals and monitoring from the start**: measure image validation quality, item identification, sat-fat estimate usefulness, policy compliance, latency, cost, and user correction behavior before and after launch.
- Make the **value proposition legible across three domains**: users get easier meal tracking and faster healthier choices, the business gets habit-forming engagement and an expandable nutrition wedge, and the system gets a safer, more measurable deployment surface.
- Optimize Phase 1 for **clarity, trust, and iteration speed**: a smaller scope yields better user value, faster feedback loops, lower model and policy risk, and cleaner expansion paths into macros, calories, and other dietary cohorts later.

## SECTION B — Product Scope and Assumptions

### Explicit assumptions

| Assumption | Why it matters |
|---|---|
| Users are uploading meal images from a mobile app, one meal at a time. | Keeps the initial workflow and user state model simple. |
| Phase 1 is estimating saturated fat, not exact nutrition truth. | Prevents fake precision and reduces safety risk. |
| The system has access to a user identifier and date context. | Needed to update a daily running total. |
| The product can store meal-level events and user daily totals. | Required for tracking and monitoring. |
| Human-reviewed nutrition labels or user-entered corrections may be added later, but not required in Phase 1. | Keeps first launch narrowly scoped. |
| The engineering team needs a workflow that is teachable and production-ready, not research-heavy. | Drives the choice toward deterministic orchestration and strong schemas. |

### In scope for Phase 1

- Upload a meal image from a mobile client.
- Validate image usability and whether it likely contains food.
- Identify likely food items from the image.
- Estimate meal-level saturated fat in grams using bounded language.
- Optionally include portion-size notes when visually inferable.
- Return confidence and ambiguity notes.
- Update and return the user’s daily saturated fat running total.
- Log outcomes for quality, safety, and operations monitoring.

### Out of scope for Phase 1

- Diagnosis of high cholesterol or any medical condition.
- Treatment recommendations, medication guidance, or disease management advice.
- Personalized diet prescriptions.
- Exact macro, calorie, micronutrient, sodium, fiber, or cholesterol calculations across all foods.
- Ingredient-level decomposition for highly complex meals with high certainty.
- Automatic barcode-based packaged food nutrition lookup as a required dependency.
- Meal planning, grocery recommendations, or coaching interventions.

### Phase 1 vs later expansion

| Area | Phase 1 | Later phases |
|---|---|---|
| Nutrition output | Saturated fat estimate only | Macros, calories, sodium, fiber, cholesterol |
| User segment | People managing high cholesterol | Broader metabolic health and dietary cohorts |
| Interaction mode | Single image upload | Image + text, barcode, meal history, corrections |
| Personalization | Daily sat-fat tally only | Personalized suggestions and trend-based insights |
| Safety posture | Informational, bounded, no advice | Potentially richer coaching with stricter review layers |

## SECTION C — System Architecture

### Logical components

#### 1. Mobile app / client
- **Responsibility:** capture/upload image, show guidance, display result, collect corrections or feedback, and render the daily total.
- **Inputs:** user-selected/captured image, authenticated user context, optional meal notes.
- **Outputs:** API request containing image reference plus user/date context; UI rendering of structured result.
- **Why it exists:** improves image quality upstream and gives the user actionable feedback without exposing internal complexity.
- **Failure modes:** poor image capture UX, missing auth context, duplicate uploads, offline retry issues.

#### 2. API gateway / orchestrator
- **Responsibility:** authenticate requests, coordinate workflow steps, enforce timeouts/idempotency, call model services, validate structured outputs, persist meal events, and return the final response.
- **Inputs:** image reference or upload, user ID, request metadata, meal timestamp.
- **Outputs:** normalized structured response, updated user total, audit/telemetry events.
- **Why it exists:** prevents a fragile “single giant model call” and centralizes policy, observability, and retries.
- **Failure modes:** schema mismatch, partial persistence failure, upstream model timeout, duplicate state updates.

#### 3. Input guardrails
- **Responsibility:** verify the image is processable, likely contains food, is sufficiently clear, and does not violate safety or content policies.
- **Inputs:** image bytes/URL and request metadata.
- **Outputs:** `accept`, `reject`, or `request_better_image` plus reason codes.
- **Why it exists:** avoids wasting model cost on bad inputs and reduces downstream hallucination risk.
- **Failure modes:** false rejections, false accepts on non-food content, inability to assess low-light or cropped images.

#### 4. Meal inference service
- **Responsibility:** identify likely food items, infer rough portions when possible, estimate saturated fat, and generate structured ambiguity notes.
- **Inputs:** validated image, optional meal notes, prompt version, schema definition.
- **Outputs:** structured meal inference JSON.
- **Why it exists:** this is the core multimodal reasoning step where image understanding adds value.
- **Failure modes:** uncertain meal composition, hidden ingredients, poor portion estimation, schema noncompliance.

#### 5. Output guardrails
- **Responsibility:** ensure the response is nutritional/informational only, bounded, uncertainty-aware, and free of diagnosis/treatment language.
- **Inputs:** meal inference JSON and safe messaging templates.
- **Outputs:** approved response or safe fallback response.
- **Why it exists:** health-tech context requires strict control over wording and claims.
- **Failure modes:** overconfident wording, accidental advice, unsupported claims, missing disclaimer fields.

#### 6. User state / daily saturated fat tracker
- **Responsibility:** persist meal events, aggregate a user’s daily saturated fat total, support idempotent updates, and expose totals to the app.
- **Inputs:** approved meal estimate, user ID, date key, request ID.
- **Outputs:** updated daily total and audit log entry.
- **Why it exists:** the product value is not just per-meal estimation, but daily tracking behavior over time.
- **Failure modes:** duplicate increments, timezone/date-boundary bugs, missing user context, state drift from corrected meals.

#### 7. Logging, monitoring, and evaluation layer
- **Responsibility:** capture structured telemetry, prompts/schema versions, latency, cost, confidence levels, corrections, rejections, and safety incidents.
- **Inputs:** events from every workflow step.
- **Outputs:** dashboards, alerts, eval datasets, failure-mode slices, release gates.
- **Why it exists:** production AI systems improve through measured iteration, not intuition.
- **Failure modes:** missing event coverage, PII over-logging, inability to tie issues to prompt/model versions.

### Simple text architecture diagram

```text
[Mobile App]
    |
    v
[API Gateway / Orchestrator]
    |
    +--> [Input Guardrails]
    |         |
    |         +--> reject / request better image
    |
    +--> [Meal Inference Service]
    |
    +--> [Output Guardrails]
    |
    +--> [User State / Daily Sat-Fat Tracker]
    |
    +--> [Logging + Monitoring + Eval Layer]
    |
    v
[Safe Structured Response to Client]
```

### End-to-end request flow

1. User captures a meal photo in the mobile app.
2. Client sends image plus authenticated user context and timestamp to the API gateway.
3. Orchestrator runs deterministic checks first: file type, size, upload integrity, dedupe key.
4. Input guardrails assess whether the image is food-related, usable, and allowed.
5. If accepted, orchestrator calls the meal inference service with a versioned prompt and structured output schema.
6. Inference service returns food items, estimated sat-fat grams, portion notes, and ambiguity flags.
7. Output guardrails review the structured result and generate safe end-user language.
8. User state service updates the daily running total idempotently.
9. Logging layer records latency, cost, confidence, safety outcome, prompt/schema version, and user feedback hooks.
10. Client receives a bounded response with estimate, uncertainty notes, total-to-date, and any next-step prompt such as “retake image.”

### Fallback paths

| Scenario | Fallback behavior |
|---|---|
| Non-food image | Return a rejection with a reason code such as `no_food_detected` and ask for a meal photo. |
| Low quality image | Return `better_image_required` with capture guidance such as brighter lighting or top-down framing. |
| Uncertain meal inference | Return a lower-confidence estimate, ambiguity flags, and an explicit note that the estimate may vary due to hidden ingredients or unclear portions. |
| Unsafe / policy-violating response | Block unsafe language and replace with a safe informational fallback message plus disclaimers. |
| Missing user context | Return an estimate without updating daily totals, set a state warning, and ask the client to retry with authenticated context. |

## SECTION D — Model Strategy and Tradeoffs

### Recommended tiered model architecture

| Step | Recommended approach | Why |
|---|---|---|
| Input safety moderation | Moderations API (`omni-moderation-latest`) | Purpose-built safety screening before inference. |
| Input guardrails | Deterministic file checks + lightweight vision/text classification step | Cheap, fast, and good enough for accept/reject routing. |
| Meal inference | Stronger multimodal model with structured output | Best suited for mixed visual reasoning and ambiguity handling. |
| Output safety / formatting | Deterministic templates first, optional lightweight review step second | Safer and more predictable than re-generating the whole answer. |

### Do not use one model for everything

A single large end-to-end call can work in demos, but it is a poor production default for this use case because it:
- couples image validation, nutrition reasoning, safe wording, and state updates into one opaque step;
- makes it difficult to measure where failures occur;
- increases cost when low-value requests should have been rejected earlier;
- creates a harder safety review surface in a health context.

### Tradeoffs across accuracy, latency, cost, and safety

| Design choice | Accuracy | Latency | Cost | Safety |
|---|---|---|---|---|
| Use strong multimodal model only for accepted images | High on relevant requests | Moderate | Controlled | Better than full-pipeline large-model usage |
| Use lightweight classifier for input screening | Moderate but sufficient | Low | Low | Good at blocking obvious bad inputs |
| Deterministic output templating | Stable | Low | Low | Very strong for bounded messaging |
| Strong model for all steps | Potentially high | Higher | Higher | Harder to audit and constrain |

### When to use a stronger multimodal model

Use the stronger multimodal step when:
- the image passed basic quality checks;
- the system must infer multiple food items or ambiguous composition;
- portion estimation is needed;
- the meal is mixed or restaurant-style rather than a single obvious item;
- you want detailed ambiguity notes with structured fields.

### When lighter / cheaper steps are enough

Use lighter logic when:
- verifying upload metadata and image validity;
- checking for obvious non-food images;
- mapping confidence bands to user-facing wording;
- validating JSON schema compliance;
- applying final response templates and disclaimers.

### Recommended Phase 1 model mapping

| Workflow stage | Recommended model/capability | Notes |
|---|---|---|
| Unsafe/disallowed-content screen | `omni-moderation-latest` | Run early on image input to block disallowed content. |
| Food/usability screening | `gpt-4.1-mini` (image input) | Lower-cost routing pass for food/non-food and quality checks. |
| Meal inference | `gpt-4.1` (image input + structured output) | Main reasoning step for food identification, sat-fat estimate, and ambiguity flags. |
| Output review (optional) | `gpt-4.1-mini` (text) | Enforce bounded informational wording and policy-safe phrasing. |
| Offline eval triage (optional) | `gpt-5-mini` (text) | Use for analysis of traces/failures outside user-facing request path. |

### Why structured outputs matter here

Structured outputs are critical because they:
- make downstream state updates deterministic;
- reduce parsing failures in production;
- support eval automation by field rather than whole-text judgment;
- separate machine-readable facts from user-facing messaging;
- allow confidence, ambiguity, and safety metadata to be explicit rather than buried in prose.

### Why confidence / ambiguity handling matters

This system will often face hidden sauces, cooking oils, cheese, mixed dishes, and unclear portions. Confidence handling is not a nice-to-have; it is central to product trust. The app should be comfortable saying:
- “likely grilled chicken salad with cheese, but dressing amount is unclear,”
- “estimate is low confidence due to image angle,” or
- “unable to estimate reliably from this image alone.”

That behavior is safer than overconfident specificity.

## SECTION E — Structured Output Schema

### Main inference response schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "MealAnalysisResponse",
  "type": "object",
  "required": [
    "request_id",
    "status",
    "food_items",
    "estimated_portion_notes",
    "estimated_saturated_fat_grams",
    "confidence_level",
    "ambiguity_flags",
    "user_daily_saturated_fat_total",
    "safe_user_message",
    "disclaimers",
    "escalation_flag"
  ],
  "properties": {
    "request_id": { "type": "string" },
    "status": {
      "type": "string",
      "enum": ["accepted", "uncertain", "rejected"]
    },
    "food_items": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["name", "confidence", "evidence"],
        "properties": {
          "name": { "type": "string" },
          "confidence": {
            "type": "string",
            "enum": ["low", "medium", "high"]
          },
          "evidence": { "type": "string" },
          "estimated_portion": { "type": ["string", "null"] },
          "estimated_saturated_fat_grams": { "type": ["number", "null"] }
        },
        "additionalProperties": false
      }
    },
    "estimated_portion_notes": { "type": "string" },
    "estimated_saturated_fat_grams": {
      "type": ["number", "null"],
      "minimum": 0
    },
    "confidence_level": {
      "type": "string",
      "enum": ["low", "medium", "high"]
    },
    "ambiguity_flags": {
      "type": "array",
      "items": {
        "type": "string",
        "enum": [
          "hidden_ingredients",
          "unclear_portion_size",
          "mixed_dish",
          "packaged_food_label_not_visible",
          "low_image_quality",
          "multiple_possible_foods",
          "missing_user_context",
          "non_food_image",
          "unsafe_content"
        ]
      }
    },
    "user_daily_saturated_fat_total": {
      "type": "object",
      "required": ["date", "total_grams", "updated"],
      "properties": {
        "date": { "type": ["string", "null"], "format": "date" },
        "total_grams": { "type": ["number", "null"], "minimum": 0 },
        "updated": { "type": "boolean" }
      },
      "additionalProperties": false
    },
    "safe_user_message": { "type": "string" },
    "disclaimers": {
      "type": "array",
      "items": { "type": "string" },
      "minItems": 1
    },
    "escalation_flag": {
      "type": "string",
      "enum": [
        "none",
        "retake_image",
        "manual_review_candidate",
        "client_context_required",
        "policy_block"
      ]
    }
  },
  "additionalProperties": false
}
```

### Sample JSON response — valid meal

```json
{
  "request_id": "req_01HZXM_VALID_001",
  "status": "accepted",
  "food_items": [
    {
      "name": "cheeseburger",
      "confidence": "high",
      "evidence": "bun, beef patty, and melted cheese are clearly visible",
      "estimated_portion": "1 burger",
      "estimated_saturated_fat_grams": 8.5
    },
    {
      "name": "french fries",
      "confidence": "medium",
      "evidence": "side portion of fries visible next to burger",
      "estimated_portion": "small side",
      "estimated_saturated_fat_grams": 1.5
    }
  ],
  "estimated_portion_notes": "Portions appear to be a single burger with a small side of fries.",
  "estimated_saturated_fat_grams": 10.0,
  "confidence_level": "medium",
  "ambiguity_flags": ["unclear_portion_size"],
  "user_daily_saturated_fat_total": {
    "date": "2026-03-23",
    "total_grams": 15.5,
    "updated": true
  },
  "safe_user_message": "Estimated saturated fat for this meal is about 10 grams. This is a meal-image estimate and may vary based on portion size and preparation.",
  "disclaimers": [
    "This estimate is informational and not medical advice.",
    "Image-based nutrition estimates can vary based on ingredients and preparation."
  ],
  "escalation_flag": "none"
}
```

### Sample JSON response — uncertain case

```json
{
  "request_id": "req_01HZXM_UNCERTAIN_002",
  "status": "uncertain",
  "food_items": [
    {
      "name": "creamy pasta dish",
      "confidence": "medium",
      "evidence": "bowl of pasta with white sauce is visible",
      "estimated_portion": "1 bowl",
      "estimated_saturated_fat_grams": null
    }
  ],
  "estimated_portion_notes": "Portion appears moderate, but sauce amount and added cheese are difficult to assess from the image.",
  "estimated_saturated_fat_grams": null,
  "confidence_level": "low",
  "ambiguity_flags": ["hidden_ingredients", "unclear_portion_size", "mixed_dish"],
  "user_daily_saturated_fat_total": {
    "date": "2026-03-23",
    "total_grams": 5.5,
    "updated": false
  },
  "safe_user_message": "I can identify this as a likely creamy pasta dish, but I cannot estimate saturated fat reliably from this image alone. A clearer image or meal details may help.",
  "disclaimers": [
    "This estimate is informational and not medical advice.",
    "For mixed dishes with hidden ingredients, image-only estimates may be unreliable."
  ],
  "escalation_flag": "retake_image"
}
```

### Sample JSON response — rejected image

```json
{
  "request_id": "req_01HZXM_REJECTED_003",
  "status": "rejected",
  "food_items": [],
  "estimated_portion_notes": "No usable meal content detected.",
  "estimated_saturated_fat_grams": null,
  "confidence_level": "low",
  "ambiguity_flags": ["non_food_image", "low_image_quality"],
  "user_daily_saturated_fat_total": {
    "date": "2026-03-23",
    "total_grams": 5.5,
    "updated": false
  },
  "safe_user_message": "I could not confirm a usable food image. Please upload a clearer photo with the meal centered and visible.",
  "disclaimers": [
    "This system only provides informational meal estimates.",
    "No daily total was updated because the image could not be analyzed."
  ],
  "escalation_flag": "retake_image"
}
```

## SECTION F — Guardrails and Safety Design

### Input guardrails

#### Objectives
- Confirm the image likely contains food.
- Reject unusable images before expensive inference.
- Reject unsafe or disallowed content.
- Ask for a better image when a retry is more useful than a hard failure.

#### Recommended checks

| Check type | Method | Action |
|---|---|---|
| File integrity | MIME/type/size checks, corruption detection | Reject invalid uploads |
| Basic visual quality | Blur/brightness/framing heuristics + lightweight model check | Ask for retake if too low quality |
| Food presence | Lightweight classifier or multimodal prompt with strict label output | Reject non-food images |
| Disallowed content | Safety moderation layer for unsafe or irrelevant content | Policy block |
| Dedupe/idempotency | Request hash and image fingerprint | Avoid double counting |

### Output guardrails

#### Objectives
- Keep the product informational and nutrition-focused.
- Prevent diagnosis, treatment, or prescriptive medical language.
- Force bounded, uncertainty-aware wording.
- Maintain stable schema and disclaimer behavior.

#### Recommended controls

| Guardrail | Implementation |
|---|---|
| No diagnosis | Prompt and policy rules forbid statements about disease status or test interpretation. |
| No treatment advice | Block language about medications, prescriptions, or what a user “should” do medically. |
| Bounded wording | Use response templates such as “estimated,” “likely,” and “may vary.” |
| Low-confidence handling | Require ambiguity flags and safer language when confidence is low. |
| Schema validation | Reject or repair malformed responses before returning to client. |

### Health-tech policy stance

This system should be explicit that:
- it **does not diagnose** high cholesterol or any other condition;
- it **does not prescribe treatment** or recommend medication changes;
- it **does not replace a clinician or registered dietitian**;
- it provides **meal-level nutritional estimates for tracking purposes only**.

That policy stance should appear in prompts, templates, product copy, QA criteria, and incident review.

## SECTION G — Evaluation Strategy

### Value proposition and how to measure success

| Domain | Value proposition | Primary success measures |
|---|---|---|
| Users | Make saturated-fat tracking easier from meal photos, help users understand daily consumption versus target, and support faster healthier meal decisions. | `% of days within sat-fat target`, `reduction in average sat-fat intake over time`, thumbs up/down feedback on whether the app helps users understand sat-fat intake and make faster healthier decisions, `% of users logging meals daily`. |
| Business | Build a habit-forming daily tracking workflow with strong retention, higher engagement, and a clean path to broader nutrition products. | daily logging rate, 7/30-day retention, repeat meal-analysis usage, user satisfaction target such as `>= 60%`, pilot conversion to broader nutrition features. |
| System | Deliver a safe, bounded, and measurable production workflow that is reliable enough to operate and improve. | `% valid food detections`, estimate quality target such as `> 85%` on agreed eval slices, hallucination rate, safety-violation rate, uptime target, latency target such as `< 3 sec`, cost per request target such as `< $0.005`, rejection/correction/low-confidence rates. |

### Eval philosophy

Treat evaluation as a release gate, not a cleanup task. The first launch should succeed because the workflow is narrow enough to measure.

### Before launch: offline eval dataset

Build a Phase 1 dataset with labeled examples across:
- clear single-item meals;
- mixed meals with sauces, cheese, and cooking fats;
- restaurant plating vs home meals;
- packaged foods where labels are visible and not visible;
- non-food images;
- low-light, blurry, cropped, and partially occluded images;
- difficult portion-size scenarios;
- culturally diverse meal types.

### Offline eval metrics

| Eval area | Example metrics |
|---|---|
| 1. Input validation | food/non-food precision-recall, low-quality rejection precision, false reject rate |
| 2. Meal item identification | top-k item match rate, mixed-meal identification score, reviewer agreement |
| 3. Saturated fat estimate quality | within-range acceptance rate, median absolute error band vs reference estimate, high-uncertainty abstention rate |
| 4. Schema correctness | valid JSON rate, required-field compliance, enum/value compliance |
| 5. Policy/safety compliance | medical advice violation rate, overclaim rate, disclaimer presence rate |

### After launch: monitoring metrics

Track production metrics continuously:
- end-to-end latency, with a Phase 1 working target of `< 3 seconds` for the common path;
- model latency by step;
- cost per request, with a target such as `< $0.005` for sustainable scaling;
- accept / reject / retake rates;
- low-confidence rate, with an initial target of keeping it around or below `~25%` while preserving safe abstention behavior;
- daily-total update success rate;
- user correction rate;
- user feedback or thumbs-down rate;
- safety incident count and severity, with a launch goal of no material safety violations;
- schema failure and fallback rate;
- service uptime, with a clearly defined pilot SLO rather than an informal availability guess.

### Detecting new failure modes

Use three feedback loops:
1. **Automated slices:** cluster failures by cuisine, lighting, mixed dishes, packaging, device type, and prompt/schema version.
2. **User corrections:** treat edits or repeated retakes as weak supervision signals.
3. **Human review queue:** sample low-confidence and high-impact cases weekly for policy and quality review.

### Logs feeding improvement

Logs should make it easy to answer:
- Which prompt version caused a spike in retakes?
- Which model version increased low-confidence responses?
- Which meal categories drive user corrections?
- Are rejections appropriately concentrated in bad inputs, or are we blocking good meals?

Those insights should feed prompt revisions, guardrail tuning, schema changes, and routing logic updates.

### Three-layer evaluation framework

| Layer | Goal | Example measures |
|---|---|---|
| User outcomes | Is the product helping people build a useful sat-fat tracking habit? | `% of users logging meals daily`, `% of days within sat-fat target`, reduction in average sat-fat intake over time, thumbs up/down sentiment, self-reported usefulness for understanding sat-fat consumption and making faster healthier decisions |
| Business outcomes | Is the product engaging, retainable, and expandable? | daily active logging rate, 7/30-day retention, repeat analyses per week, `>= 60%` user satisfaction target, pilot conversion to adjacent nutrition experiences |
| System outcomes | Is the system accurate, safe, fast, and affordable enough to operate? | `% valid food detections`, estimate quality on eval set with target such as `> 85%`, hallucination rate, safety-violation rate, p95 latency `< 3 sec`, cost/request `< $0.005`, rejection/correction/low-confidence rates, uptime/SLO attainment |

## SECTION H — Developer Workflow and Codex Best Practices

### How the team should use Codex productively

Codex should accelerate implementation tasks that are repetitive, scaffold-heavy, or schema-sensitive, including:
- prototyping prompts for input validation and meal inference;
- generating typed API clients from OpenAPI or JSON schema;
- scaffolding backend routes and DTOs;
- writing eval harnesses and fixture loaders;
- debugging schema validation failures;
- creating synthetic test fixtures and edge-case cases;
- iterating on orchestration code with small diffs;
- drafting architecture and runbook documentation.

### Practical workflow guidance

| Task | Good use of Codex | Required human review |
|---|---|---|
| Prompt prototyping | Draft structured prompts and revise ambiguity instructions | Safety and product review |
| API scaffolding | Generate route handlers, request/response types, and validators | Backend review and tests |
| Eval scripts | Create dataset loaders, scorers, and dashboards | Metric design review |
| Schema debugging | Find malformed outputs and suggest tighter schema constraints | Final contract ownership by engineers |
| Docs | Produce first-draft architecture docs and runbooks | Technical accuracy review |

### Best practices
- Codex speeds iteration; it does **not** replace engineering judgment.
- Evals must exist before scaling traffic or broadening scope.
- Code review and safety review remain mandatory for every production change.
- Work in **small, testable increments** rather than giant prompt rewrites.
- Keep prompts, schemas, thresholds, and routing logic **versioned**.
- Log prompt version, model version, and schema version on every request.
- Prefer deterministic checks when they can do the job; use LLMs where reasoning genuinely adds value.

## SECTION I — Implementation Roadmap

### Phase 0 — design and eval setup
- **Goals:** finalize scope, schema, prompt contracts, and eval plan.
- **Key deliverables:** architecture decision record, structured schema, baseline prompts, offline dataset plan, telemetry plan.
- **Risks:** starting implementation before agreeing on evaluation criteria.
- **Go / no-go criteria:** schema approved, safety principles documented, eval dataset plan funded and staffed.

### Phase 1 — happy path prototype for sat-fat estimation
- **Goals:** ship end-to-end flow for accepted, clear food images.
- **Key deliverables:** mobile upload, orchestrator, input checks, meal inference, daily total updates, basic dashboards.
- **Risks:** overestimating portion quality and underinvesting in fallback handling.
- **Go / no-go criteria:** stable schema rate, acceptable latency/cost, strong performance on clear-image eval slice.

### Phase 2 — guardrails + eval hardening
- **Goals:** strengthen rejection logic, uncertainty handling, output safety, and observability.
- **Key deliverables:** expanded eval set, policy tests, alerting, low-confidence UX, correction capture.
- **Risks:** false rejections that reduce usefulness.
- **Go / no-go criteria:** safety compliance targets met, retake UX acceptable, major failure modes instrumented.

### Phase 3 — pilot launch
- **Goals:** validate real-world usefulness with a limited cohort.
- **Key deliverables:** pilot dashboards, incident runbook, review cadence, model/prompt versioning discipline.
- **Risks:** real-world image diversity exceeds offline dataset assumptions.
- **Go / no-go criteria:** user correction rates manageable, no major safety incidents, operating cost within target.

### Phase 4 — expansion to more nutrition dimensions / cohorts
- **Goals:** add additional nutrient estimates, text input, barcode support, and cohort-specific experiences.
- **Key deliverables:** expanded schema, richer prompts, new eval slices, cohort-specific product policy.
- **Risks:** scope creep outpaces evaluability.
- **Go / no-go criteria:** Phase 1 metrics stable over time and expansion metrics defined before build-out.

## SECTION J — Presentation Draft

### Slide 1 — Title
- **Bullets:**
  - AI-powered Meal Analysis Cholesterol App
  - Phase 1 focus: meal-level saturated fat estimation
  - Production-ready architecture for safe, measurable launch
- **Speaker notes:** Introduce the narrow scope and explain that the goal is not generic nutrition AI, but a bounded system that can actually be deployed responsibly.

### Slide 2 — Value proposition
- **Bullets:**
  - User value: easier sat-fat tracking, better visibility into daily consumption, faster healthier decisions
  - Business value: daily engagement, habit formation, and a strong wedge into broader nutrition
  - System value: bounded workflow with measurable grams/day outcomes and safer production deployment
- **Speaker notes:** Emphasize that the product is useful because it is specific. We are solving one meaningful tracking problem first, and we can measure success clearly for users, the business, and the system.

### Slide 3 — Why start narrow
- **Bullets:**
  - Easier to evaluate than full nutrition estimation
  - Lower safety and product risk
  - Faster path to trust, feedback, and iteration
- **Speaker notes:** A narrow target user and metric create a tractable system. This is the right first production choice, not a limitation.

### Slide 4 — Process flow
- **Bullets:**
  - Validate image → infer meal → estimate sat fat → apply guardrails → update daily total
  - Explicit fallback for non-food, low-quality, and uncertain cases
  - Structured output enables deterministic product behavior
- **Speaker notes:** Walk through the bounded workflow and note that every step is measurable.

### Slide 5 — High-level architecture
- **Bullets:**
  - Mobile client, orchestrator, input guardrails, meal inference, output guardrails, state store, monitoring
  - Separate components improve auditability and reliability
  - Logging and eval are part of the architecture, not an afterthought
- **Speaker notes:** Explain why the orchestrator pattern is better than a single large model call in production.

### Slide 6 — Production example
- **Bullets:**
  - Example: burger + fries → accepted image, medium-confidence estimate, daily total updated
  - Example: creamy pasta → uncertain result, no total update, retake/details requested
  - Example: non-food image → rejected early
- **Speaker notes:** Use the examples to illustrate trust-preserving behavior under clarity and ambiguity.

### Slide 7 — Features & limitations
- **Bullets:**
  - Supports food detection, sat-fat estimate, ambiguity notes, and daily totals
  - Does not diagnose, prescribe treatment, or provide exact nutrition truth
  - Uses bounded language and disclaimers by design
- **Speaker notes:** Clear limitations improve safety and set the right expectations.

### Slide 8 — Key challenges & mitigations
- **Bullets:**
  - Hidden ingredients and portion ambiguity
  - Bad images and non-food uploads
  - Overconfident or unsafe language in health contexts
- **Speaker notes:** Map each challenge to a mitigation: guardrails, confidence scoring, and deterministic templates.

### Slide 9 — Model strategy
- **Bullets:**
  - Lightweight screening for input guardrails
  - Stronger multimodal model for accepted meal images
  - Deterministic or lightweight post-processing for safe outputs
- **Speaker notes:** Reinforce that different tasks deserve different model and non-model tools.

### Slide 10 — Evaluation
- **Bullets:**
  - User success: logging frequency, days within sat-fat target, reduction in average sat-fat intake, thumbs up/down usefulness
  - Business success: retention, daily engagement, satisfaction, expansion readiness
  - System success: food-detection validity, `> 85%` estimate quality on eval slices, latency `< 3 sec`, cost `< $0.005`, low-confidence / correction / safety rates
- **Speaker notes:** Explain that success must be measured across users, business, and system operations. That makes iteration disciplined rather than subjective.

### Slide 11 — Codex workflow
- **Bullets:**
  - Use Codex for scaffolding, schemas, eval scripts, and documentation
  - Keep prompts and schemas versioned
  - Require code review and safety review for production changes
- **Speaker notes:** Position Codex as a multiplier for engineering productivity, not a replacement for rigor.

### Slide 12 — Takeaways
- **Bullets:**
  - Narrow scope makes the product safer, clearer, and more deployable
  - Bounded workflow + structured outputs = production readiness
  - Evals, monitoring, and guardrails are core to launch quality
- **Speaker notes:** Close by framing the plan as intentionally practical: useful first product, strong learning loop, clean expansion path.

## SECTION K — Loom Script (Polished, <10 min)

**Target length:** ~8–9 minutes

### 0:00–0:45 — Opening and framing
Hi everyone — in this walkthrough I’ll present a production-ready design for our AI-powered Meal Analysis Cholesterol App.

This is intentionally a **narrow Phase 1 launch**: we are helping users who manage high cholesterol estimate **saturated fat per meal** from an image and track their daily total.

We are not trying to solve all nutrition on day one. We are choosing a scope that is safer, measurable, and deployable.

### 0:45–1:45 — Value proposition in three domains
The value proposition is clear across three domains:

1. **User value:** easier meal logging, better understanding of daily saturated-fat consumption, and faster healthier decisions.
2. **Business value:** stronger daily engagement, habit formation, and a credible expansion path into broader nutrition offerings.
3. **System value:** a bounded workflow with explicit guardrails and measurable outputs, which is far safer than an unconstrained AI feature.

### 1:45–3:05 — End-to-end workflow
The workflow has seven explicit stages:

- user uploads image,
- deterministic upload checks,
- input guardrails,
- meal inference,
- output guardrails,
- daily total state update,
- logging and evaluation.

If an image is non-food, low quality, or unsafe, we do not force a guess. We return a clear fallback: reject, retake, or policy block.

That behavior is critical in health-adjacent products: reliable abstention is better than fake precision.

### 3:05–4:35 — Architecture and component boundaries
Architecturally, we use an **orchestrator-first pattern**:

- Mobile client captures image and displays bounded results.
- API orchestrator coordinates every step, handles idempotency/timeouts, and assembles the final response.
- Input guardrails filter low-quality and non-food images before expensive inference.
- Meal inference service performs multimodal reasoning and returns structured outputs.
- Output guardrails enforce non-medical, informational wording.
- State service updates daily saturated-fat totals idempotently.
- Logging/eval layer captures metrics, versions, and failure modes.

This gives us auditability: we can diagnose failures by stage, not guess from one giant model call.

### 4:35–5:45 — Model strategy and OpenAI best practices
Model routing is tiered:

- **Moderation first** with `omni-moderation-latest`.
- **Low-cost image screening** with `gpt-4.1-mini` for food/usability routing.
- **Primary multimodal inference** with `gpt-4.1` for food items, sat-fat estimate, and ambiguity.
- Optional lightweight output review with `gpt-4.1-mini`.

Key best practices:

- structured outputs with strict schema validation,
- deterministic checks before model calls,
- uncertainty as an explicit field,
- versioned prompts/schemas,
- eval-first launch gates.

### 5:45–7:00 — Safety, uncertainty, and limitations
We are explicit about boundaries:

- this system does **not diagnose**,
- does **not prescribe treatment**,
- and does **not replace clinicians or dietitians**.

Outputs must stay informational and bounded.

When confidence is low — for example mixed dishes, hidden oils, unclear portions — we return uncertainty flags or abstain rather than over-claim.

That is how we preserve trust.

### 7:00–8:10 — Success criteria and operating metrics
We measure success in three domains:

- **User:** daily logging, days within sat-fat target, and reduction in average sat-fat intake over time.
- **Business:** retention, repeat usage, and user satisfaction.
- **System:** valid food detection, estimate quality, safety-violation rate, latency, cost per request, rejection/correction/low-confidence rates.

These metrics are not reporting-only; they are release gates and iteration signals.

### 8:10–9:00 — Close and next steps
Main takeaway: this is a practical production plan — narrow scope, strong guardrails, structured outputs, and measurable outcomes.

If we execute this Phase 1 well, we earn the right to expand into calories, macros, personalization, and broader cohorts with much lower risk.

Immediate next steps are:

1. finalize prompt/schema versions,
2. implement the happy-path orchestrator,
3. build and run the offline eval dataset,
4. launch a limited pilot with monitoring and safety review cadence.

That gives us a safe and credible path from prototype to production.

## SECTION L — Implementation Scaffold

### Suggested starter stack

- **Mobile client:** React Native for image capture, session management, and results display.
- **Backend:** Python FastAPI or TypeScript Node service for orchestration. For a team new to production AI, FastAPI plus Pydantic is a strong fit because schemas, validation, and eval scripting are straightforward.
- **AI integration:** dedicated OpenAI client wrapper with prompt versioning, retries, and telemetry hooks.
- **Storage:** relational store for meal events and daily totals; object storage for images if server-side upload persistence is needed.
- **Observability:** OpenTelemetry-compatible tracing, structured logs, metrics dashboards, and eval exports.

### Suggested folder structure

```text
.
├── README.md
├── .env.example
├── docs/
│   └── meal-analysis-cholesterol-app.md
├── apps/
│   ├── mobile/
│   │   └── README.md
│   └── api/
│       ├── README.md
│       └── src/
│           ├── routes/
│           │   └── meal-analysis.md
│           └── schemas/
│               └── meal-analysis-response.schema.json
├── prompts/
│   └── meal_inference/
│       └── v1/
│           └── system.md
├── evals/
│   ├── README.md
│   └── datasets/
│       └── phase1_satfat/
│           └── manifest.json
└── tests/
    └── fixtures/
        └── README.md
```

### Key service responsibilities

| Service | Responsibility |
|---|---|
| `mobile` | Capture meal images, surface capture guidance, render bounded results, collect corrections |
| `api/orchestrator` | Authenticate requests, route to guardrails and inference, validate schema, persist totals |
| `schemas` | Define machine-readable contracts and shared types |
| `prompts` | Store versioned prompts and policy instructions |
| `evals` | Hold offline datasets, scoring scripts, and release-gate logic |
| `tests/fixtures` | Keep representative request/response and edge-case assets |

### Example environment variables

```dotenv
OPENAI_API_KEY=
OPENAI_MODEL_MODERATION=omni-moderation-latest
OPENAI_MODEL_MEAL_INFERENCE=gpt-4.1
OPENAI_MODEL_INPUT_GUARDRAILS=gpt-4.1-mini
OPENAI_MODEL_OUTPUT_REVIEW=gpt-4.1-mini
DATABASE_URL=
OBJECT_STORAGE_BUCKET=
OTEL_EXPORTER_OTLP_ENDPOINT=
APP_ENV=development
MEAL_ANALYSIS_PROMPT_VERSION=v1
MEAL_ANALYSIS_SCHEMA_VERSION=2026-03-23
MAX_IMAGE_MB=10
REQUEST_TIMEOUT_MS=12000
```

### Where to store prompts
- Store prompts in `prompts/<workflow>/<version>/`.
- Keep system prompts, reviewer prompts, and prompt change notes versioned together.
- Reference prompt versions in logs and persisted meal events.

### Where to store eval datasets
- Store eval manifests in `evals/datasets/<dataset_name>/`.
- Keep labels, metadata, and slices explicitly versioned.
- Separate gold/reference labels from generated outputs.

### Example API endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/v1/meal-analysis` | Submit image and user context for analysis |
| `GET` | `/v1/daily-totals/{user_id}?date=YYYY-MM-DD` | Fetch user daily saturated fat total |
| `POST` | `/v1/meal-analysis/{request_id}/feedback` | Capture correction or quality feedback |
| `POST` | `/v1/meal-analysis/{request_id}/retry` | Retry analysis for improved image |

### Practical implementation notes

- Start with a single happy-path route and make every workflow stage observable.
- Use schema validation at every boundary: model output, API response, and state-update event.
- Persist request IDs and idempotency keys to avoid double counting meal totals.
- Treat user corrections as future eval data.
- Keep the first mobile UX simple: upload, result, retake, optional correction.
- Expand scope only after the Phase 1 eval and pilot metrics are stable.

## SECTION M — OpenAI Best Practices Applicable to This Project

### 1) Best practices applicable (and where)

| Best practice | Where it applies in this project | Practical implementation note |
|---|---|---|
| Use a bounded workflow, not one giant model call | API orchestrator + route contract | Keep explicit stages: input checks → guardrails → inference → output guardrails → state update. |
| Use multimodal models only where image reasoning adds value | Meal inference service | Reserve stronger multimodal inference for accepted images only to control cost and latency. |
| Prefer deterministic checks first | Upload integrity and orchestration layer | Enforce MIME/size/corruption/idempotency checks before calling any model. |
| Use structured outputs with strict schema validation | `MealAnalysisResponse` contract + API response layer | Validate model output against schema before state updates and user response assembly. |
| Separate model reasoning from user-safe messaging | Output guardrails + templating | Generate core inference fields first, then produce bounded user language with disclaimers. |
| Build evals before scaling | `evals/` and launch gates | Maintain offline labeled slices and block broad rollout until target thresholds are met. |
| Instrument every stage for monitoring and iteration | Logging/telemetry layer | Log model/prompt/schema versions, confidence, fallback reasons, latency, and cost. |
| Explicit uncertainty handling and abstention | Inference + response contract | Use confidence levels and ambiguity flags; return uncertain/rejected when reliability is low. |
| Health-context safety boundaries | Prompt policy + output guardrails | Enforce no diagnosis, no treatment recommendations, and informational-only language. |
| Version prompts and contracts | `prompts/` + schema + env vars | Persist prompt/schema versions in request logs to support safe rollback and analysis. |
| Keep agentic behavior offline-first | Ops/eval workflows | Use optional agents for trace review/eval triage, not as the synchronous user-path orchestrator. |

### 2) Key challenges

| Challenge | Why it is hard | Mitigation in this architecture |
|---|---|---|
| Hidden ingredients (oils, butter, sauces) | Often not visually observable | Force ambiguity flags and lower confidence instead of fake precision. |
| Portion-size ambiguity from one image | No depth scale or prep metadata | Return portion notes as estimates and use uncertainty-aware language. |
| Mixed dishes with multiple plausible interpretations | Visual overlap and occlusion | Use stronger multimodal model only for accepted images and allow abstention. |
| Non-food / low-quality uploads | Common in real mobile usage | Deterministic checks + low-cost guardrail classification + retake UX. |
| Safety drift in generated wording | Health-adjacent context raises risk | Output guardrails and approved templates for bounded, non-medical responses. |
| Cost and latency pressure | Multimodal calls can be expensive | Early rejection and tiered model routing to keep p95 and cost in bounds. |
| State consistency | Duplicate or retried requests can double-count totals | Idempotency keys and request IDs on write path. |

### 3) Known limitations (Phase 1)

- This is an **estimate-first** system, not exact nutrition truth from images.
- It does not diagnose, prescribe treatment, or replace clinicians/dietitians.
- Confidence can be low for complex, mixed, low-light, or heavily occluded meals.
- Portion estimates are approximate and may vary by angle/container.
- A single image may be insufficient without user-provided context in some cases.
- Early launch focuses on saturated fat only, not complete nutrition analysis.

### 4) Success criteria (three domains)

| Domain | Success criteria (examples) | Why it matters |
|---|---|---|
| Users | `% users logging meals daily`, `% days within sat-fat target`, reduction in average sat-fat intake over time, thumbs up/down usefulness signal | Confirms the feature creates real behavior change and perceived value. |
| Business | day-7/day-30 retention, repeat analyses per week, satisfaction target (e.g., `>=60%`), expansion readiness to additional nutrition features | Confirms this narrow wedge can support durable product growth. |
| System | `% valid food detections`, estimate quality on agreed eval slices (e.g., `>85%`), hallucination rate, safety-violation rate, p95 latency target (e.g., `<3s`), cost/request target (e.g., `<$0.005`), low-confidence/rejection/correction rates | Confirms the system is safe, reliable, and economically operable. |

### 5) Main takeaways

- The right Phase 1 pattern is **orchestrator-first, bounded, and measurable**.
- **Model tiering** (cheap guardrails + stronger inference + bounded output review) is the best tradeoff for quality, safety, latency, and cost.
- **Structured outputs + strict validation** are mandatory for dependable state updates.
- **Uncertainty is a feature**, not a failure: abstain when confidence is low.
- **Evals and monitoring are product features** in production AI, not post-launch extras.
- Start narrow, ship safely, learn quickly, then expand.
