# Pathgen agent training and evaluation

The final agent reached **80.11% accuracy** on **30,000 held-out Junyi examples from 3,169 students**. BKT alone scored **80.27%** on the same rows. The agent detected more struggle cases and assigned better probabilities, with **50.64% struggle recall** versus **40.48%** for BKT and **0.4162 log loss** versus **0.4342**. These scores describe the Junyi benchmark.

The six executed code cells and their outputs are in [agent_training.ipynb](../../roror/agent_training.ipynb). The fitted model and its test results are in [agent_model.pkl](../../roror/agent_model.pkl).

## 1. What the agent learned

The model estimates the **probability that a learner will struggle on a candidate next question**. Each example comes from a real mapped Junyi encounter. `struggle_on_next_question` is `1` when the candidate has `correct=false`, more than one attempt, or a hint. Otherwise it is `0`. Junyi's `correct` flag includes its hint override. The agent's probability can be used when ranking eligible questions in a teacher's pool.

The final model uses these 12 features:

| Feature group | Columns |
| --- | --- |
| Candidate | `candidate_skill_id`, `candidate_difficulty` |
| Last completed encounter | `last_correct`, `last_attempt_count`, `last_hint_used`, `last_duration_sec` |
| Up to five earlier encounters | `recent_correct_rate`, `recent_hint_rate`, `recent_mean_attempts`, `recent_median_duration_sec` |
| Earlier practice in the candidate skill | `candidate_skill_prior_encounters`, `candidate_skill_bkt_mastery` |

`user_id` and `candidate_time_done` are tracking fields. The candidate's own correctness, attempts, and hints supply the target after its features have been calculated. [Agent dataset schema](agent_training.md) · [How the CSV generator works](agent_csv_explain.md)

## 2. Data splits and leakage checks

The notebook used the balanced Junyi exports below. The generator assigns students to disjoint splits and samples real encounters within each skill. Within each skill and split, the original proportions of struggle and success were retained to the nearest whole row.

| Dataset | Rows | Students | Rows per skill |
| --- | ---: | ---: | ---: |
| [Training CSV](../../roror/agent_training.csv) | 100,000 | 10,146 | 10,000 |
| [Validation CSV](../../roror/agent_validation.csv) | 25,000 | 2,568 | 2,500 |
| [Test CSV](../../roror/agent_testing.csv) | 30,000 | 3,169 | 3,000 |

The 25,000 validation rows were divided by **student** into three groups before any model selection:

| Validation role | Rows | Students | Purpose |
| --- | ---: | ---: | --- |
| Model selection | 12,170 | 1,263 | Choose which fitted model to keep. |
| Calibration | 6,623 | 653 | Fit the optional probability calibrator while holding the chosen model fixed. |
| Threshold tuning | 6,207 | 652 | Decide whether to keep calibration and choose the classification threshold. |

The notebook checked the schema, skill quotas, numeric ranges, duplicate student/timestamp keys, and assignment of each student to the correct split. The earlier CSV generation used five training student folds for the difficulty and BKT estimates. For a training row, those estimates exclude its student's fold when fitting shared parameters. The student's mastery state then uses their own earlier answers. Validation and test use frozen, training-derived estimates. The model received only the 12 feature columns, and the notebook first read the test CSV after the model, calibration, and thresholds were fixed.

## 3. What happened in each notebook cell

| Cell | What it did |
| --- | --- |
| 1 | Set the feature list, target, input paths, BKT baseline, and evaluation settings. |
| 2 | Checked the training and validation files and assigned validation students to their three roles. |
| 3 | Built the preprocessing pipelines, fitted four candidate models on the training data, and selected by validation log loss. |
| 4 | Fitted a sigmoid calibrator on its reserved learners, compared calibrated and raw probabilities, and selected the thresholds on the tuning learners. |
| 5 | Opened the held-out test file, verified student separation, and scored the final model, baselines, and per-skill results. |
| 6 | Saved the pickle, loaded it again, and checked that it reproduced the test probabilities on the verification sample. A separate fresh-process check reproduced test accuracy, log loss, and the confusion matrix. |

Each cell was executed and checked before the next one. The run used scikit-learn **1.9.1**, pandas **3.0.6**, and NumPy **2.5.3**. [Scikit-learn guidance on training-only preprocessing](https://scikit-learn.org/stable/common_pitfalls.html)

### Preprocessing and model selection

- **Encoding.** One-hot encode `candidate_skill_id` so the skill IDs have no artificial numeric order.
- **Missing values.** Impute missing numeric features with their training medians. Add missing-value indicators for columns with missing values in training.
- **Scaling.** Standardize numeric inputs using the training means and scales. The fitted preprocessing pipeline is saved with the estimator, so predictions use the same transformations.

Models were ranked by **mean log loss across the ten skills**, with equal weight per skill. Lower is better. The validation comparison was:

| Model candidate | Selection log loss | ROC AUC |
| --- | ---: | ---: |
| **Histogram gradient boosting, 31 leaves** | **0.42315** | **0.77582** |
| Histogram gradient boosting, 15 leaves | 0.42383 | 0.77497 |
| Histogram gradient boosting, 31 leaves, without template difficulty | 0.42738 | 0.77095 |
| Logistic regression | 0.43469 | 0.76169 |

The selected `HistGradientBoostingClassifier` used a maximum of **31 leaves per tree**, **220 boosting iterations**, a **0.07 learning rate**, **115 minimum rows per leaf**, and **2.5 L2 regularization**. The version using template difficulty performed better than the version omitting it on the model-selection group. BKT alone had **0.4436** mean log loss on those same learners.

### Calibration and threshold

Sigmoid calibration was fitted with the selected model frozen. On the threshold-tuning learners, mean skill log loss went from **0.4112** with raw probabilities to **0.4110** after calibration, so the calibrated version was retained. [Scikit-learn probability calibration](https://scikit-learn.org/stable/modules/calibration.html)

The threshold rule chose the highest F1 among cutoffs that achieved at least **80% validation accuracy** and **25% validation struggle recall**. If none met both, the planned fallback was the highest F1 overall. The agent met both constraints at **0.315**, with **80.47% validation accuracy** and **50.75% validation struggle recall**. The BKT comparison used the same selection rule and got a threshold of **0.350**. Both cutoffs were fixed before testing.

## 4. Held-out test results

On the final test set, **6,203 of 30,000 encounters (20.68%)** were labeled as struggle. The **BKT-only** comparison uses one minus BKT's predicted probability of correctness. BKT calculates correctness from the student's current mastery and the fitted guess and slip probabilities. This is a proxy for the struggle label. The agent also sees recent behavior and candidate difficulty.

| Test metric | Pathgen agent | BKT alone | Skill training rate | Always predict success |
| --- | ---: | ---: | ---: | ---: |
| Accuracy | 80.11% | **80.27%** | 79.32% | 79.32% |
| Balanced accuracy | **69.21%** | 65.56% | 50.00% | 50.00% |
| Precision for struggle | 51.95% | **53.01%** | 0.00% | 0.00% |
| Recall for struggle | **50.64%** | 40.48% | 0.00% | 0.00% |
| F1 for struggle | **0.5129** | 0.4590 | 0.0000 | 0.0000 |
| ROC AUC | **0.7781** | 0.7539 | 0.6790 | — |
| Average precision | **0.5419** | 0.4938 | 0.3330 | — |
| Log loss ↓ | **0.4162** | 0.4342 | 0.4755 | — |
| Brier score ↓ | **0.1302** | 0.1367 | 0.1524 | — |

The skill training-rate baseline assigns the same probability to every row of a given skill, using that skill's training struggle rate. Precision is the share of struggle alerts that were correct. Recall is the share of actual struggles that were detected. F1 combines precision and recall. Balanced accuracy gives success and struggle equal weight. Log loss and Brier score evaluate the probability estimates.

### Confusion matrix at the saved 0.315 threshold

| Actual outcome | Predicted success | Predicted struggle |
| --- | ---: | ---: |
| Success | 20,892 | 2,905 |
| Struggle | 3,062 | 3,141 |

The agent met the **80% accuracy target** on this benchmark. BKT's accuracy was higher by **0.16 percentage points**. The agent detected **10.16 percentage points more** of the actual struggles, with a small decrease in precision. An always-success rule scored 79.32% accuracy because success was common, while detecting zero struggles.

### Results by skill

Each skill has 3,000 held-out examples. The same global threshold was used across all skills.

| Skill | Struggle rate | Accuracy | Struggle recall | F1 | Agent log loss ↓ | BKT log loss ↓ |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Integers on a number line | 24.87% | 73.77% | 46.78% | 0.4700 | 0.5049 | 0.5214 |
| Opposites | 15.83% | 83.20% | 32.84% | 0.3824 | 0.3946 | 0.4057 |
| Absolute-value comparison | 12.20% | 86.13% | 12.84% | 0.1843 | 0.3402 | 0.3395 |
| Compare and order integers | 7.93% | 91.97% | 18.07% | 0.2630 | 0.2487 | 0.2612 |
| Add integers | 16.63% | 80.17% | 36.47% | 0.3796 | 0.3981 | 0.4175 |
| Subtract integers | 35.97% | 66.07% | 79.24% | 0.6268 | 0.5625 | 0.5815 |
| Multiply integers | 10.87% | 88.87% | 9.20% | 0.1523 | 0.3196 | 0.3310 |
| Divide integers | 14.13% | 84.53% | 17.45% | 0.2418 | 0.3796 | 0.3894 |
| Order of operations | 42.70% | 72.03% | 79.63% | 0.7086 | 0.5150 | 0.5660 |
| Signed quantities and changes | 25.63% | 74.37% | 50.07% | 0.5003 | 0.4988 | 0.5291 |

The agent had lower log loss than BKT in **nine of ten skills**. BKT had a slight edge on absolute-value comparison. Multiplication had **88.87% accuracy** and only **9.20% struggle recall** at the global cutoff, an example of why the per-skill results and the probabilities matter when choosing how to use the model.

## 5. Using the saved model in Pathgen

`agent_model.pkl` includes the fitted pipeline, the sigmoid calibrator, the exact feature list, the **0.315** cutoff, test metrics, and hashes of the data used. Use the positive-class probability for question ranking. To reproduce the reported binary flags, compare that probability with `decision_threshold` stored in the pickle.

The model is ready for integration testing. The 80.11% result measures predictions on **held-out Junyi encounters**. Before evaluating live accuracy, align teacher difficulty tags with the Junyi template difficulty scale and collect outcomes from Pathgen's own questions. A separate evaluation is needed to measure whether using the predictions to choose questions improves student learning.
