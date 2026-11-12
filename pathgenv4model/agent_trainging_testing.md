# OpenPath agent training, validation, and testing

## 1. What is the planned split per skill?

OpenPath will use the same row quota for each of its ten skills. The training file will contain **100,000 rows**, and the testing file will contain **30,000 rows**. A separate validation set will contain **25,000 rows**.

| Split | Rows per skill | Total rows across ten skills |
| --- | ---: | ---: |
| Training | 10,000 | 100,000 |
| Validation | 2,500 | 25,000 |
| Testing | 3,000 | 30,000 |
| **Total** | **15,500** | **155,000** |

Assign each student to one split using the existing BKT student assignments. Then build the history features from that student's earlier mapped encounters before sampling the finished examples. Sample real rows without replacement and keep the original proportion of struggle and success within each skill as closely as rounding allows. [Agent training schema](agent_training.md)

## 2. Why balance the skills instead of training on every available row?

One agent will learn across all ten skills. With equal weight per row, addition's **138,463** available training examples would contribute nearly **14 times** as much to the training objective as the **10,131** available for opposites. A pooled score could also hide weak predictions on a skill with fewer examples. Equal skill quotas give each skill the same representation and reduce the training workload.

Skill balance and target balance are separate. A skill may naturally have more successful encounters than struggle encounters. Preserving that mix lets us evaluate whether the agent's predicted struggle probabilities match the rates observed for each skill. Equal skill counts also leave differences in which parts of the Grade 7 skills Junyi covers.

## 3. How many rows are available in each skill's split?

These counts cover the existing Junyi skill map and the established student splits, before sampling. An encounter is eligible when the student has at least one earlier mapped encounter. The quotas in Section 1 apply to each skill below. [Junyi problem log](../../pathgen0.4/junyi_ProblemLog_original.csv) · [Skill map](../../pathgen0.4/csv/skill_map.csv)

| OpenPath skill | Available for training | Available for validation | Available for testing |
| --- | ---: | ---: | ---: |
| 1. Integers on a number line | 134,371 | 35,648 | 42,000 |
| 2. Opposites | 10,131 | 2,518 | 3,373 |
| 3. Absolute-value comparison | 47,903 | 12,322 | 13,872 |
| 4. Compare and order integers | 88,094 | 21,902 | 28,140 |
| 5. Add integers | 138,463 | 34,288 | 44,572 |
| 6. Subtract integers | 57,047 | 14,098 | 18,620 |
| 7. Multiply integers | 27,083 | 6,219 | 7,912 |
| 8. Divide integers | 26,958 | 6,161 | 7,861 |
| 9. Order of operations | 16,855 | 3,624 | 5,280 |
| 10. Signed quantities and changes | 50,621 | 11,958 | 15,434 |
| **Total** | **597,526** | **148,738** | **187,064** |

Opposites sets the limit in all three splits. Its **10,131 training examples** and **3,373 test examples** come from different students. Each pool can supply its planned quota independently.

## 4. Why have separate training, validation, and testing sets?

| Split | What it is for |
| --- | --- |
| **Training** | Fit the agent so it learns how the available features relate to next-question struggle. |
| **Validation** | Compare models and feature choices, tune settings, and check the quality of the predicted probabilities on students whose examples were excluded from fitting. |
| **Testing** | Measure the final model on a separate group of students after model and feature choices are fixed. |

Validation lets us improve the model without using the final test results to decide what to change. Keeping whole students in separate splits gives us a clearer measure of performance on new learners. For evaluation, report log loss for each skill, the equal-weight average across skills, and the comparison with BKT alone.

## 5. Is a smaller balanced dataset better than a larger unbalanced one?

It depends on the goal. **The smaller balanced sample is a good first choice for OpenPath** because all ten skills matter and 100,000 training rows fit the chosen budget. Its tradeoff is that downsampling can discard useful examples of unusual student behavior or less common question variants in the larger skills.

A larger uneven sample can retain more of that variety. With equal row weights, high-volume skills still have more influence during fitting. If we later have the compute budget, we can compare the balanced run with a larger run that weights each skill equally in the training objective. Judge both on the same validation students using per-skill log loss and the equal-weight average. That comparison will show which approach gives OpenPath better predictions.
