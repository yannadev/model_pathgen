# How `agent_csv.py` builds the OpenPath agent datasets

The script converts Junyi's recorded encounters into examples for predicting whether a student will struggle on a candidate question. It reads the student's earlier answers, calculates features, attaches the candidate's outcome as the target, and writes three CSVs. The training rows come from real Junyi encounters. Difficulty, recent behavior, and mastery are calculated from those records.

This explanation refers to the [implemented `agent_csv.py` in `Downloads/roror`](../../roror/agent_csv.py). The [agent training schema](agent_training.md) explains each exported column, and the [split plan](agent_trainging_testing.md) explains the row quotas.

## 1. Inputs and outputs

| Input | Why the script uses it |
| --- | --- |
| [Original Junyi problem log](../../pathgen0.4/junyi_ProblemLog_original.csv) | Supplies `user_id`, `time_done`, `exercise`, `problem_type`, `correct`, `count_attempts`, `hint_used`, and `time_taken`. |
| [Skill map](../../pathgen0.4/csv/skill_map.csv) | Maps an exact `exercise` and `problem_type` pair to one of the ten modeled skills. |
| [BKT fitting CSV used by the script](../../roror/bkt_fitting.csv) | Supplies the training histories used to refit BKT for each of the five student folds. |
| [Fitted BKT parameters](../../roror/bkt_parameters.json) | Supplies the frozen model for validation and test students, plus the expected hash of the fitting CSV. |

| Output | Rows per skill | Total rows |
| --- | ---: | ---: |
| [agent_training.csv](../../roror/agent_training.csv) | 10,000 | 100,000 |
| [agent_validation.csv](../../roror/agent_validation.csv) | 2,500 | 25,000 |
| [agent_testing.csv](../../roror/agent_testing.csv) | 3,000 | 30,000 |

Each CSV has **12 model features, one target, and two tracking columns**. `user_id` and `candidate_time_done` help verify splits and chronological order. During agent training, feed only the 12 features to the model. The candidate's completion timestamp is available in the historical log after the encounter.

## 2. How the script reads and splits the data

| Function | What it does and why |
| --- | --- |
| `hash64()` | Converts text to a deterministic number using SHA-256. Given the same student ID and seed, the script assigns that student the same split or fold each time. |
| `student_assignment()` | Applies the existing BKT split rules. It checks the test assignment first, then reserves validation students from the development group. Remaining students go to training and are assigned to one of five folds. Splitting by student keeps a learner's history together. |
| `load_skill_map()` | Reads the mapping and checks that every `exercise` and `problem_type` pair is unique. It assigns internal numeric codes for templates and skills so later calculations can use arrays efficiently. The exported skill ID comes from `modeled_skill_id`. For example, `PLUS` and `MINUS` under the same Junyi exercise can map to different skills. |
| `load_mapped_encounters()` | Reads the log in chunks of 500,000 rows, keeps the matched encounters, and sorts them by student and `time_done`. It checks types and rejects missing values or tied timestamps for a student. This gives each learner's history a clear order and avoids holding the entire raw log in memory at once. |
| `load_frozen_bkt()` | Loads the fitted parameters and checks that the fitting CSV matches the hash recorded in the JSON. It checks parameter ranges, skill coverage, and that the fitting students all belong to training. This catches accidental use of parameters fitted on a different file. |

## 3. How it prepares the BKT models

| Function | What it does and why |
| --- | --- |
| `likelihood_counts()` | Precomputes the correct and incorrect responses before and after every possible learning point in a student's skill history. With forgetting fixed at zero, the learner can move from unmastered to mastered at most once. Precomputing these counts avoids recalculating them at every optimizer step. |
| `bkt_objective()` | Computes the negative average log likelihood and its gradient. The likelihood includes the possible learning points and cases where the student starts mastered or stays unmastered throughout the observed sequence. The gradient tells the optimizer how to adjust the parameters. |
| `fit_bkt_skill()` | Fits initial mastery, learning, guessing, and slipping for one skill. It tries the number of starting points recorded in the fitted JSON, eight for this run, and keeps the converged fit with the lowest training log loss. The model requires a higher probability of correctness in the mastered state. |
| `cross_fit_bkt()` | Fits ten skill models for each of the five training folds. When it fits for one fold, it excludes every student in that fold from parameter fitting. Those students still contribute their own earlier answers when the script later computes their mastery. Validation and test students use the frozen fit. |

The fold-specific fits make the training features more like those available for new students. A training student's answers can update that student's mastery as their history unfolds. Their outcomes have no role in estimating the shared parameters used for their fold.

## 4. How it creates a prediction row

| Function | What it does and why |
| --- | --- |
| `prior_history()` | Finds the boundaries between learners, identifies encounters with prior history, and summarizes up to five earlier mapped questions. It calculates recent correctness, hint rate, mean attempts, and median duration. Negative durations are treated as missing. The median gives unusually long durations less influence than a mean would. |
| `prior_mastery()` | Walks each learner's history in order and tracks mastery and prior encounter counts separately for each skill. It saves the candidate skill's mastery and practice count **before** updating them with that encounter. This matches what OpenPath will know when it chooses a question. |
| `template_difficulty()` | Calculates the observed fraction of training encounters labeled as struggle for each Junyi template. For a training row it excludes that learner's fold. For validation and test rows it uses the training pool. The resulting number measures historical Junyi difficulty on a scale from 0 to 1. OpenPath's teacher difficulty tags still need to be aligned with that scale. |

For example, when the candidate is question 5, `last_attempt_count` comes from question 4. The recent summaries use questions 1–4, and mastery reflects only earlier responses on question 5's skill. Question 5's outcome supplies the target:

```text
struggle_on_next_question = 1 if correct is false,
                              count_attempts > 1, or hint_used is true
                           0 otherwise
```

This timing prevents **target leakage**, which would occur if the model received question 5's outcome as an input to predicting question 5. Junyi's recorded `correct` flag includes its hint override, and the script uses that flag as supplied.

## 5. How it balances and writes the CSVs

| Function | What it does and why |
| --- | --- |
| `choose_examples()` | Samples after the history features have been calculated. For each split and skill, it chooses the agreed number of real encounters without replacement. It rounds the number of struggle examples to the nearest whole row so each sampled skill stays close to its observed outcome rate. Sampling after feature creation lets earlier encounters contribute to history even if they are outside the final sample. |
| `export()` | Runs the full pipeline, constructs the 15 columns, and checks the skill quotas, unique encounter keys, valid feature ranges, and student separation. It stages the files in a temporary folder and moves them into the output folder after all three pass its checks. |
| `main()` | Sets up the command-line options. By default it finds the fitting files beside the script, looks for the Junyi inputs in the sibling `pathgen0.4` folder, and writes the agent CSVs beside the script. The `if __name__ == '__main__'` guard starts generation when the file is run directly. |

To regenerate the CSVs from the current project folder, use Python with NumPy, pandas, and SciPy installed:

```powershell
Set-Location -LiteralPath 'C:\Users\jhadz\Downloads\roror'
python agent_csv.py
```

Use `python agent_csv.py --help` to see how to pass different input paths or save a new run to a separate output directory.

## 6. What was verified

- All **155,000 target labels** and their skill assignments matched the corresponding original Junyi encounters.
- The files met the exact per-skill quotas and kept the struggle rates within rounding distance of the eligible data for each skill and split. No learner appeared in multiple exported splits.
- An independent check reconstructed the earlier behavior features and difficulty for **90 sampled rows**. For the sampled validation and test rows, it also independently checked mastery against the frozen BKT parameters and earlier responses.

These checks cover dataset construction. After agent training, use the validation set to choose the model and the test set to measure its final prediction performance against BKT alone.
