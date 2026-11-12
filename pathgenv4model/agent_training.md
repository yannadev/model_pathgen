# OpenPath agent training data

This note describes the proposed training dataset for the agent that predicts whether a student will struggle on a candidate next question. Each training row represents a real Junyi encounter and uses the history available before that encounter.

## 1–2. How many original Junyi columns do we need, and which ones?

**Eight columns** from the original Junyi problem log are enough for the first version. [Junyi problem log](../../pathgen0.4/junyi_ProblemLog_original.csv)

| Original Junyi columns | What we use them for |
| --- | --- |
| `user_id`, `time_done` | Group encounters by student, order their history, and assign each student to training, validation, or test. |
| `exercise`, `problem_type` | Look up the question's OpenPath skill and identify the Junyi template used to estimate difficulty. |
| `correct`, `count_attempts`, `hint_used` | Describe behavior on earlier questions and calculate the candidate question's outcome label once its result is known. |
| `time_taken` | Measure how long the student spent on earlier questions. |

## 3–5. What are the complete agent columns, what do they do, and where do they come from?

The proposed CSV has **15 columns**, made up of **12 features**, **one target**, and **two tracking columns**. Each row describes a decision point and the next mapped question that Junyi recorded for that student.

| Agent CSV column | Source or derivation | Purpose |
| --- | --- | --- |
| `user_id` | Junyi `user_id`. | Tracking. Groups the student's history and keeps the student in one split. |
| `candidate_time_done` | Junyi `time_done` for the candidate encounter. | Tracking. Verifies chronological order. This completion timestamp is excluded from the model inputs. |
| `candidate_skill_id` | Look up the candidate's `exercise` and `problem_type` in `skill_map.csv` and use `modeled_skill_id`. | Identifies the skill the candidate question tests. |
| `candidate_difficulty` | Estimate difficulty from training students' outcomes on the same Junyi template. | Represents how challenging the candidate is. The estimate needs calibration against OpenPath's teacher-assigned levels. |
| `last_correct` | `correct` from the most recent completed mapped encounter. | Shows the student's last result under Junyi's scoring rule. |
| `last_attempt_count` | `count_attempts` from that encounter. | Shows how many submissions the student made. |
| `last_hint_used` | `hint_used` from that encounter. | Shows whether the student requested a hint. |
| `last_duration_sec` | Cleaned `time_taken` from that encounter. | Measures time spent on the last question. |
| `recent_correct_rate` | Share of `correct=true` over the previous five mapped encounters, or all available if fewer than five. | Measures recent success. |
| `recent_hint_rate` | Share of `hint_used=true` over that same window. | Measures how often the student requests help. |
| `recent_mean_attempts` | Mean `count_attempts` over that window. | Measures how many submissions the student typically makes. |
| `recent_median_duration_sec` | Median valid `time_taken` over that window. | Measures typical response time with less influence from unusually long encounters. |
| `candidate_skill_prior_encounters` | Count earlier mapped encounters with the candidate's skill. | Measures prior practice on that skill. |
| `candidate_skill_bkt_mastery` | Use fitted BKT parameters and the student's earlier responses on the candidate skill. | Estimates how well the student currently knows that skill. |
| `struggle_on_next_question` | Apply the target rule to the candidate encounter's `correct`, `count_attempts`, and `hint_used`. | Target. The outcome the agent learns to predict. |

The skill map matches `exercise` and `problem_type` together. For example, `PLUS` and `MINUS` under `adding_and_subtracting_negative_numbers` map to separate OpenPath skills. Use the map's `modeled_skill_id` for this lookup. Every training example comes from a recorded encounter. Features such as difficulty and mastery are estimates calculated from real responses. [Skill map](../../pathgen0.4/csv/skill_map.csv)

## 6. How do we derive the agent CSV?

1. **Map and sort the encounters.** Match `exercise` and `problem_type` to a skill. Sort the mapped encounters by `time_done` within each student. A next mapped encounter can follow intervening Junyi questions outside the selected skill map.
2. **Split by student.** Reuse the BKT student assignments for training, validation, and test. Read behavior from the original problem log. The four-column fitting file contains only the data needed for BKT. [BKT fitting note](bkt_fitting.md)
3. **Calculate features using earlier encounters.** Take the last completed mapped encounter, summarize up to five previous mapped encounters, and count prior practice in the candidate's skill. Treat negative durations as missing. For training rows, estimate template difficulty and fit the BKT parameters using other training students. Then update each student's BKT state using their own earlier responses. Use training students only to estimate difficulty and fit BKT parameters for validation and test predictions.
4. **Attach the candidate outcome as the label.** Use the candidate's recorded correctness, attempts, and hint use to calculate the target after the feature values have been assembled.

For example, when predicting question 5, question 4's two attempts and hint can be inputs. If the student solves question 5 on the first attempt without a hint, the target for question 5 is `0`.

## 7. What are the features and the target?

**Features.** The 12 columns from `candidate_skill_id` through `candidate_skill_bkt_mastery` in the table. The learner ID and the candidate's completion timestamp are tracking columns.

**Target.** `struggle_on_next_question`.

```text
1 if the candidate has correct=false, count_attempts > 1, or hint_used=true
0 otherwise
```

The agent outputs a **probability of struggle for a candidate question**. The selection rule uses that probability to choose from the teacher's pool. The mastery feature can be populated after BKT fitting. The difficulty feature still needs calibration against OpenPath's teacher tags.

## 8. Why do we need a new CSV for agent training?

The agent needs one row per prediction example. That row brings together the candidate's skill and difficulty, the student's earlier behavior, their current BKT mastery, and the outcome used as the target. Those values come from multiple Junyi rows, the skill map, and the fitted BKT model.

A prepared CSV lets us inspect the features and target, check that students stay in their assigned split, and reuse the same examples when comparing models. The BKT fitting CSV serves a different purpose. It contains the ordered binary responses used to fit mastery parameters and omits the behavior fields needed here.

## 9. Why can't we just use the original Junyi CSV directly for training?

The original Junyi CSV records what happened during individual encounters. The agent predicts an outcome **before** the candidate is answered. If we feed the candidate's own `correct`, `count_attempts`, or `hint_used` values into the model, we give it the information that defines the answer. That is target leakage.

The raw log also needs chronological ordering, OpenPath skill mapping, history summaries, and a student-level split. A training program could perform those transformations as it reads the raw file. Saving the prepared CSV makes the transformation easier to verify and avoids repeating it for each training run.
