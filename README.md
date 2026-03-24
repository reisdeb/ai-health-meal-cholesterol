# AI-powered Meal Analysis Cholesterol App

This repository contains a production-oriented technical enablement asset and starter scaffold for a Phase 1 meal analysis system focused on estimating **saturated fat per meal** for people managing high cholesterol.

## Contents
- `docs/meal-analysis-cholesterol-app.md` — customer-ready technical enablement document covering architecture, guardrails, evaluation, roadmap, slide outline, and Loom script.
- `apps/api/src/schemas/meal-analysis-response.schema.json` — starter JSON Schema for structured meal analysis responses.
- `apps/api/README.md` — backend/orchestration scaffold guidance.
- `apps/mobile/README.md` — mobile client responsibilities and integration notes.
- `prompts/meal_inference/v1/system.md` — versioned starter prompt for multimodal meal inference.
- `evals/README.md` — evaluation dataset and workflow guidance.
- `.env.example` — example environment variables.

## Phase 1 focus
Phase 1 is intentionally narrow:
- Validate whether an uploaded image contains a usable food photo.
- Identify likely food items.
- Estimate saturated fat with bounded, uncertainty-aware language.
- Update the user’s running daily saturated fat total.
- Avoid medical advice, diagnosis, and treatment recommendations.

## Recommended next steps
1. Review the technical enablement document.
2. Align on the Phase 1 API contract and schema.
3. Stand up the happy-path backend orchestration flow.
4. Build the offline eval set before broadening scope.
