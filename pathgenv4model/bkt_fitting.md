# OpenPath BKT fitting data

## 1. Why create `bkt_fitting.csv` from Junyi?

Junyi provides historical practice records. Bayesian Knowledge Tracing (BKT) needs those records organized as **a sequence of correct and incorrect responses for each student and skill**. The fitting CSV puts the data in that format so we can estimate starting mastery, the learning rate, and the probabilities of guessing correctly or making a mistake for each modeled skill.

Validation students were reserved before sampling. Each selected student's complete history for that skill was kept in chronological order. The **30,000-row cap per skill** lets you load and fit one skill at a time on your laptop.

**Every row represents a real Junyi encounter.** Sampling selected existing records without generating answers, filling in missing responses, or duplicating rows to balance the skills.

## 2. Which Junyi columns were used?

Five columns from the original problem log feed the fitting CSV: **`user_id`, `time_done`, `exercise`, `problem_type`, and `correct`**. The local `skill_map.csv` maps Junyi's exercise and problem type to a modeled skill. The exercise metadata table helped review coverage. The CSV builder gets its skill assignments from the map. [Junyi problem log](../../pathgen0.4/junyi_ProblemLog_original.csv) · [Junyi exercise metadata](../../pathgen0.4/junyi_Exercise_table.csv) · [Skill map](../../pathgen0.4/csv/skill_map.csv)

### How `skill_map.csv` works

The map uses `exercise` and `problem_type` together to match a Junyi record to one `modeled_skill_id`. This lets OpenPath separate addition and subtraction even when Junyi records them under the same exercise.

| Junyi `exercise` | Junyi `problem_type` | Fitting `modeled_skill_id` |
| --- | --- | --- |
| `adding_and_subtracting_negative_numbers` | `PLUS` | `INT_ADD_NEG` |
| `adding_and_subtracting_negative_numbers` | `MINUS` | `INT_SUB_NEG` |

`template_id` identifies the mapping rule. `approved_skill_id` comes from an earlier grouping and should not be read as the OpenPath skill number in Section 3. The fitting file uses `modeled_skill_id` to identify the skill.

## 3. How many rows are in the CSV?

The capped CSV in the OpenPath `model` folder contains **263,813 data rows**, excluding the header. [Fitting CSV](bkt_fitting.csv)

| OpenPath skill | Skill ID in the CSV | Rows |
| --- | --- | ---: |
| 1. Integers on a number line | `INT_NUMBER_LINE` | 29,976 |
| 2. Opposites | `INT_OPPOSITE_POS4` | 11,863 |
| 3. Absolute value, represented by comparison tasks | `INT_ABS_COMPARE` | 29,983 |
| 4. Compare and order integers | `INT_ORDER_NEG` | 29,990 |
| 5. Add integers | `INT_ADD_NEG` | 29,999 |
| 6. Subtract integers | `INT_SUB_NEG` | 29,989 |
| 7. Multiply integers | `INT_MULT_NEG` | 27,442 |
| 8. Divide integers | `INT_DIV_NEG` | 27,536 |
| 9. Order of operations | `INT_EXPR_NEG` | 17,038 |
| 10. Signed quantities and changes | `INT_CONTEXT` | 29,997 |
| **Total** | | **263,813** |

Four skills used all eligible training histories and still fell below the cap. The other six stopped near 30,000 at a student boundary so that each selected history stayed intact.

## 4. What does each fitting column do?

| Fitting column | Where it comes from | Purpose |
| --- | --- | --- |
| `user_id` | Copied from Junyi's `user_id`. | Groups responses belonging to the same student and keeps that student in one data split. |
| `order_id` | Derived by sorting each student's records by Junyi's `time_done`. After sampling, retained rows are numbered again from `1` for each student. | Gives BKT the order in which to read the student's responses. |
| `modeled_skill_id` | Derived by looking up `exercise` and `problem_type` together in `skill_map.csv`. | Tells BKT which skill model receives the response. |
| `correct` | Copied from Junyi's `correct` flag and encoded as `1` for true and `0` for false. | Supplies the binary observation BKT uses to update mastery. |

**Junyi's `correct` flag represents first-attempt correctness with a hint override.** Junyi records `false` when a hint is requested. The capped CSV preserves those labels, so its `0` values include hint use as well as incorrect responses. [Junyi release field definitions](https://github.com/swu-qjw-lab/code/blob/main/RMCD/data/junyi/README.md)
