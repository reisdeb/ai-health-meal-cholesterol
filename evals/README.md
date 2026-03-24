# Evaluation Scaffold

## Phase 1 dataset goals
Build an offline dataset that includes:
- clear food images
- mixed meals
- restaurant meals
- packaged foods
- low-light or blurry images
- non-food images
- difficult portion-size cases
- culturally diverse meals

## Suggested dataset manifest fields
- `example_id`
- `image_path` or storage URI
- `slice_tags`
- `food_present_label`
- `quality_label`
- `reference_food_items`
- `reference_sat_fat_range`
- `policy_notes`

## Release-gate eval categories
1. Input validation quality
2. Food item identification quality
3. Saturated fat estimate usefulness and abstention behavior
4. Structured schema compliance
5. Safety and policy compliance
