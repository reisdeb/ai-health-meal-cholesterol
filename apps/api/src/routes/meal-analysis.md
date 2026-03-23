# `/v1/meal-analysis` route contract

## Request body
- `user_id`: authenticated user identifier
- `meal_timestamp`: ISO-8601 timestamp from client
- `image_url` or uploaded file reference
- `client_request_id`: optional idempotency key
- `meal_note`: optional short text hint from user

## Response body
Return the shared `MealAnalysisResponse` schema defined in `../schemas/meal-analysis-response.schema.json`.

## Route behavior
1. Validate auth and request shape.
2. Run upload integrity checks.
3. Run input guardrails.
4. If accepted, call meal inference with versioned prompt.
5. Validate structured output.
6. Apply output guardrails / safe user messaging.
7. Update daily totals idempotently if safe and allowed.
8. Emit telemetry and return response.
