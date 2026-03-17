# What Makes a Recipe Great? An Analysis of Food.com Recipes and Ratings
**By Alice Guo**

This project investigates the relationship between recipe complexity, nutrition, and user ratings using data from Food.com.

---

## Introduction

This dataset contains recipes and user ratings scraped from Food.com. The central question this project explores is: **do simpler recipes receive higher ratings than more complex ones?** This question matters because it touches on a fundamental tension in cooking — whether users prefer accessible, straightforward recipes or reward the effort that goes into more elaborate ones.

The dataset contains **83,782 rows**, where each row represents a unique recipe. The relevant columns for this analysis are:

| Column | Description |
|---|---|
| `name` | Name of the recipe |
| `n_steps` | Number of steps in the recipe |
| `n_ingredients` | Number of ingredients required |
| `minutes` | Total time to prepare the recipe |
| `calories` | Total calories (from nutrition info) |
| `total_fat` | Total fat as % daily value |
| `protein` | Protein as % daily value |
| `carbohydrates` | Carbohydrates as % daily value |
| `avg_rating` | Average user rating (1–5 stars) |

---

## Data Cleaning and Exploratory Data Analysis

The following cleaning steps were performed:

1. **Merged ratings with recipes**: The interactions dataset was merged onto the recipes dataset using `recipe_id`. Ratings of 0 were replaced with `NaN` since a 0 rating indicates no rating was given, not a true score of 0.
2. **Computed average rating**: After replacing 0s, the mean rating per recipe was computed and merged back as `avg_rating`.
3. **Parsed nutrition column**: The `nutrition` column was stored as a string representation of a list. It was split and expanded into individual numeric columns: `calories`, `total_fat`, `sugar`, `sodium`, `protein`, `saturated_fat`, and `carbohydrates`.

Here is the head of the cleaned DataFrame:

| name | n_steps | n_ingredients | minutes | calories | total_fat | protein | carbohydrates | avg_rating |
|---|---|---|---|---|---|---|---|---|
| 1 brownies in the world best ever | 10 | 9 | 40 | 138.4 | 10.0 | 3.0 | 6.0 | 4.0 |
| 1 in canada chocolate chip cookies | 12 | 11 | 45 | 595.1 | 46.0 | 13.0 | 26.0 | 5.0 |
| 412 broccoli casserole | 6 | 9 | 40 | 194.8 | 20.0 | 22.0 | 3.0 | 5.0 |
| millionaire pound cake | 7 | 7 | 120 | 878.3 | 63.0 | 20.0 | 39.0 | 5.0 |
| 2000 meatloaf | 17 | 13 | 90 | 267.0 | 30.0 | 29.0 | 2.0 | 5.0 |

### Univariate Analysis

<iframe src="assets/cal_distribution.html" width="800" height="500" frameborder="0"></iframe>

The distribution of calories (capped at 2000 kcal) is right-skewed, with most recipes falling below 500 calories. This suggests that the majority of Food.com recipes are everyday meals rather than indulgent or high-calorie dishes.

<iframe src="assets/rating_by_steps.html" width="800" height="500" frameborder="0"></iframe>

Average ratings are remarkably consistent across recipe complexity bins, hovering between 4.6 and 4.7 regardless of the number of steps. This hints that users may not systematically penalize or reward complexity in their ratings.

### Bivariate Analysis & Interesting Aggregates

The table below shows the mean rating for each step complexity bin, aggregated across all recipes:

| Number of Steps | Mean Rating |
|---|---|
| 1–5 | ~4.68 |
| 6–10 | ~4.67 |
| 11–15 | ~4.66 |
| 16–20 | ~4.65 |
| 21–25 | ~4.64 |
| 26+ | ~4.63 |

While mean ratings decline very slightly as recipes become more complex, the differences are small, suggesting recipe complexity alone is not a strong driver of user satisfaction.

---

## Assessment of Missingness

**MNAR Discussion**: The `description` column in this dataset is potentially MNAR (Missing Not At Random). Recipes without descriptions are likely submitted by less engaged contributors who may also receive fewer ratings — meaning the missingness is tied to an unobserved variable (contributor engagement) that is not captured in the data itself. To confirm this and make the missingness MAR, we would ideally want additional data such as contributor activity level or submission history.

**Missingness Dependency — `avg_rating` vs. `n_steps`** (MAR):

A permutation test was run to test whether missingness of `avg_rating` depends on `n_steps`. The observed difference in mean `n_steps` between recipes with and without ratings was significant (p-value = 0.0000), meaning we reject the null hypothesis. The missingness of `avg_rating` **does** depend on `n_steps` — more complex recipes are less likely to have been rated.

<iframe src="assets/missingness_nsteps.html" width="800" height="500" frameborder="0"></iframe>

**Missingness Dependency — `avg_rating` vs. `sodium`** (MCAR):

A second permutation test found no significant relationship between missingness of `avg_rating` and sodium content (p-value = 0.9000). We fail to reject the null hypothesis — the missingness of `avg_rating` does **not** depend on sodium content, consistent with MCAR for this variable.

<iframe src="assets/missingness_sodium.html" width="800" height="500" frameborder="0"></iframe>

---

## Hypothesis Testing

**Question**: Do simpler recipes (at or below the median number of steps) receive higher average ratings than more complex ones?

- **Null Hypothesis**: Simple and complex recipes have the same average rating; any observed difference is due to random chance.
- **Alternative Hypothesis**: Simple recipes have a higher average rating than complex recipes.
- **Test Statistic**: Difference in mean ratings (simple − complex). This is appropriate because we are comparing a numerical outcome across two groups.
- **Significance Level**: 0.05

**Result**: p-value = 0.2270. We fail to reject the null hypothesis. There is no statistically significant evidence that simpler recipes receive higher ratings than complex ones.

<iframe src="assets/hypothesis_test.html" width="800" height="500" frameborder="0"></iframe>

---

## Framing a Prediction Problem

**Prediction Problem**: Can we predict the number of **calories** in a recipe based on its other attributes?

This is a **regression** problem. The response variable is `calories`, chosen because it is a concrete, nutritionally meaningful quantity that users and recipe platforms would benefit from being able to estimate. We evaluate our model using **RMSE (Root Mean Squared Error)** because it penalizes large errors more heavily than MAE, which is desirable when large calorie mispredictions would be most harmful. All features used (steps, ingredients, fat, protein, carbohydrates, etc.) are known at the time a recipe is submitted, so there is no data leakage.

---

## Baseline Model

The baseline model is a **Linear Regression** trained on two quantitative features:

- `n_steps` (quantitative): number of steps in the recipe
- `n_ingredients` (quantitative): number of ingredients

No encoding was needed since both features are numeric. The model was implemented in a single `sklearn` Pipeline.

**Performance:**
| Split | RMSE | R² |
|---|---|---|
| Train | 423.86 | 0.0353 |
| Test | 428.30 | 0.0393 |

The baseline model performs poorly — an R² near 0 means it explains almost none of the variance in calories. This makes sense because the number of steps and ingredients alone carry little information about caloric content. A recipe with 5 steps could be a salad or a cheesecake.

---

## Final Model

The final model uses a **Random Forest Regressor** with the following features added on top of the baseline:

- `total_fat` (quantitative): fat content is directly tied to caloric density
- `protein` (quantitative): protein contributes calories and improves prediction
- `carbohydrates` (quantitative): the primary macronutrient driver of calorie count
- `sugar` (quantitative): high sugar recipes tend to be calorie-dense
- `log_minutes` (quantitative): log-transformed cook time, since raw minutes are heavily skewed; longer recipes may involve more ingredients and higher calories

These features were chosen because macronutrients have a direct, known relationship to caloric content (fat = 9 kcal/g, carbs and protein = 4 kcal/g each). Including them transforms the problem from one requiring guesswork to one grounded in nutritional science.

A `StandardScaler` was applied before the Random Forest. Hyperparameter tuning was performed using `GridSearchCV` (3-fold CV) over:
- `n_estimators`: [50, 100]
- `max_depth`: [10, 20, None]

**Best parameters**: `max_depth=20`, `n_estimators=100`

**Performance:**
| Split | RMSE | R² |
|---|---|---|
| Train | 19.93 | 0.9979 |
| Test | 44.34 | 0.9897 |

The final model dramatically outperforms the baseline — Test RMSE dropped from 428.30 to 44.34 and R² improved from 0.039 to 0.990. This confirms that macronutrient information is highly predictive of caloric content.

---

## Fairness Analysis

**Groups**:
- **Group X**: Low-calorie recipes (at or below median calories in the test set)
- **Group Y**: High-calorie recipes (above median calories)

**Evaluation Metric**: RMSE

**Null Hypothesis**: The model is fair. Its RMSE for low-calorie and high-calorie recipes is roughly the same, and any difference is due to random chance.

**Alternative Hypothesis**: The model is unfair. Its RMSE is higher for high-calorie recipes than for low-calorie ones.

**Test Statistic**: Difference in RMSE (high − low). **Significance Level**: 0.05.

**Result**: p-value = 0.0000. We reject the null hypothesis. The model performs significantly worse on high-calorie recipes, suggesting it has a systematic bias — it struggles more with extreme calorie values, which are inherently harder to predict accurately.

<iframe src="assets/fairness_test.html" width="800" height="500" frameborder="0"></iframe>
