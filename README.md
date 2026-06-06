# Meal Macros: Predicting Recipe Calories for Diet Planning

**By Elizabeth Kao**

## Introduction

This project uses the **Recipes and Ratings** dataset from food.com, which contains two tables: `RAW_recipes.csv` (recipes submitted since 2008, including ingredients, nutrition, prep time, and steps) and `RAW_interactions.csv` (recipe reviews and ratings). The two datasets are merged to compute an average rating per recipe. After merging and cleaning, the dataset contains **82,944 rows**.

The central question driving this project is:

> **Can we accurately predict a recipe's calorie content based on its nutritional macros?**

A follow-up question explored through hypothesis testing is:

> **Are "healthy" high-protein, low-carb recipes actually rated lower, suggesting people prefer taste over nutrition?**

These questions are motivated by weight-loss diet planning. Many people starting a diet set a daily calorie ceiling or restrict certain macros, keto, for instance. But most people don't know how caloric a recipe is until the ingredients are already assembled. A model that estimates calories from features like fat, protein, and carbs could help people make better food choices before they start cooking.

The most relevant columns are:

| Column | Description |
|---|---|
| `calories` | Total calories in the recipe |
| `protein` | Protein content as % of daily value (PDV) |
| `total_fat` | Total fat as % PDV |
| `carbs` | Carbohydrates as % PDV |
| `sugar` | Sugar as % PDV |
| `sodium` | Sodium as % PDV |
| `saturated_fat` | Saturated fat as % PDV |
| `n_steps` | Number of steps in the recipe |
| `n_ingredients` | Number of ingredients |
| `minutes` | Estimated prep and cook time |
| `avg_rating` | Mean user rating (computed from interactions) |


## Data Cleaning and Exploratory Data Analysis

### Cleaning Steps

1. Left-merged `recipes` with `interactions` on recipe ID so every recipe is retained even if no reviews were left.
2. Replaced ratings of 0 with `NaN`. The food.com rating scale runs 1-5, so a rating of 0 means the user did not leave a numeric score. Treating 0 as an actual rating would deflate true averages.
3. Computed `avg_rating` per recipe by taking the mean of all non-NaN ratings for each recipe ID, then joined it onto the recipes dataframe.
4. Parsed the `nutrition` column from a string that looks like a list into seven individual numeric columns: `calories`, `total_fat`, `sugar`, `sodium`, `protein`, `saturated_fat`, `carbs`.
5. Removed extreme calorie outliers above the 99th percentile (recipes exceeding 3,200 calories), assuming data entry errors that would distort model training.
6. Parsed `submitted` as a datetime and extracted `year`.

The resulting cleaned dataset has **82,944 rows**.

### Cleaned DataFrame (first 5 rows)

| name                                 |   minutes |   n_steps |   n_ingredients |   calories |   protein |   carbs |   avg_rating |
|:-------------------------------------|----------:|----------:|----------------:|-----------:|----------:|--------:|-------------:|
| 1 brownies in the world    best ever |        40 |        10 |               9 |      138.4 |         3 |       6 |            4 |
| 1 in canada chocolate chip cookies   |        45 |        12 |              11 |      595.1 |        13 |      26 |            5 |
| 412 broccoli casserole               |        40 |         6 |               9 |      194.8 |        22 |       3 |            5 |
| millionaire pound cake               |       120 |         7 |               7 |      878.3 |        20 |      39 |            5 |
| 2000 meatloaf                        |        90 |        17 |              13 |      267.0 |        29 |       2 |            5 |

### Univariate Analysis

The distribution of calories is right-skewed, with most recipes falling between 100 and 600 calories. The mode is around 200-300 calories which makes sense for most normal meals. The long right tail supports our decision to cap at the 99th percentile, as extreme values are likely data entry errors rather than real recipes.

<iframe src="assets/calories_dist.html" width="800" height="500" frameborder="0"></iframe>

Protein content is also right-skewed. Most recipes provide less than 30% of the daily protein value, showing that food.com skews toward desserts, baked goods, and side dishes rather than protein-heavy meals. This is relevant context for our hypothesis test since high-protein recipes are a minority in this dataset.

<iframe src="assets/protein_dist.html" width="800" height="500" frameborder="0"></iframe>

### Bivariate Analysis

There is a clear positive relationship between protein content and calorie count. High-protein recipes tend to involve more meat and dairy, which add calories alongside protein. This relationship is one of the strongest signals in the prediction model.

<iframe src="assets/protein_vs_calories.html" width="800" height="500" frameborder="0"></iframe>

Recipes binned into quartiles by protein content show little variation in average rating. All tiers cluster around 4.6 to 5, suggesting that users on food.com do not rate high-protein diet recipes differently from low-protein ones. This is explored further in the hypothesis test.

<iframe src="assets/rating_by_protein_tier.html" width="800" height="500" frameborder="0"></iframe>

### Interesting Aggregates

The table below shows median calories, protein, carbs, and average rating grouped by recipe complexity (number of steps). Recipes with more steps tend to have higher median calories and protein -- complex recipes are more likely to involve protein-heavy ingredients like meat or layered dishes like lasagna. This motivates using `n_steps` as a feature in our calorie prediction model.

| steps   |   calories |   protein |   carbs |   avg_rating |
|:--------|-----------:|----------:|--------:|-------------:|
| 1-5     |     218.75 |         9 |       6 |            5 |
| 6-10    |     293.00 |        19 |       8 |            5 |
| 11-15   |     346.45 |        23 |      10 |            5 |
| 16-20   |     382.60 |        26 |      11 |            5 |
| 21+     |     435.70 |        26 |      13 |            5 |


## Assessment of Missingness

### NMAR Analysis

The `description` column is likely **NMAR** (Not Missing at Random). When a recipe contributor does not write a description, it is probably because they consider the recipe self-explanatory. The content and simplicity of the recipe itself drives the decision not to write a description -- not any other observable column in the dataset. Nothing about `n_steps`, `calories`, or `n_ingredients` fully explains why a contributor would skip a description, since that decision comes down to personal judgment.

To make this MAR, we would want additional data such as a contributor-level experience score or whether this was their first submission.

### Missingness Dependency

We analyzed whether the missingness of `avg_rating` depends on other columns.

**Test 1: Does `avg_rating` missingness depend on `calories`?**

We expected missingness to be related to calories as very high-calorie or unusual recipes might attract fewer raters. A permutation test using the difference in mean calories between missing and non-missing groups gave a p-value of 0.0000, well below alpha = 0.05. We reject the null. `avg_rating` missingness does depend on calories.

<iframe src="assets/missingness_calories.html" width="800" height="500" frameborder="0"></iframe>

**Test 2: Does `avg_rating` missingness depend on `sodium`?**

Sodium is a nutritional fact with no logical connection to whether a user chooses to rate a recipe. Whether a recipe gets rated is driven by popularity and user engagement, not salt content. The permutation test gave a p-value of 0.6970, well above alpha = 0.05. We fail to reject the null. `avg_rating` missingness does not depend on sodium, consistent with our expectation.


## Hypothesis Testing

### Do high-protein recipes get rated differently than low-protein recipes?

If users systematically rate high-protein recipes lower, it suggests palatability matters more than nutrition.

- Null Hypothesis (H0): High-protein and low-protein recipes have the same average rating in the population. Any observed difference is due to random chance.
- Alternative Hypothesis (H1): High-protein recipes have a different average rating than low-protein recipes (two-sided).
- Test Statistic: Difference in mean ratings (high-protein quartile minus low-protein quartile).
- Significance Level: alpha = 0.05

We used a permutation test because it makes no distributional assumptions and is valid for any sample where observations are exchangeable under the null. Since this is a regression problem, classification metrics like accuracy or F1-score are not applicable here.

<iframe src="assets/protein_boxplot.html" width="800" height="500" frameborder="0"></iframe>

Visually, the two groups look almost identical. Both distributions are concentrated between 4 and 5 stars with medians near 5. This suggests that protein content has little practical effect on how users rate recipes. We use the permutation test below to determine whether any observed difference is statistically significant.

<iframe src="assets/hypothesis_test.html" width="800" height="500" frameborder="0"></iframe>

Result: p-value = **0.0000**. We reject the null hypothesis at alpha = 0.05. High-protein recipes (mean rating 4.607) are rated statistically differently from low-protein recipes (mean rating 4.651). However, the magnitude of the difference (around 0.04 stars) is negligible in practical terms. Food.com users seem to rate recipes highly regardless of protein content suggesting taste and ease of preparation drive ratings more than nutritional profile.


## Framing a Prediction Problem

**Prediction task:** Predict the calorie content of a recipe given its nutritional macros and structural metadata.

- Type: Regression
- Response variable: `calories` -- a continuous numeric value
- Why calories: Calorie content is the most actionable number for weight-loss diet planning. Unlike `avg_rating`, which is only known after users interact with a recipe, calorie content can be estimated from data available at the time a recipe is posted.
- Evaluation metric: RMSE because it measures error in the same units as calories and penalizes large mistakes heavily, which matters for diet planning (a 500-calorie mistake is detrimental compared to a 20-calorie mistake). Since this is a regression problem, classification metrics like accuracy or F1-score are not applicable. We also report R-squared as a secondary interpretability metric.

We only use features available at the time of prediction: `n_steps`, `n_ingredients`, `total_fat`, `protein`, `carbs`, `sugar`. We intentionally exclude `avg_rating` since it requires post-submission user interactions.


## Baseline Model

Our baseline is a **Linear Regression** model in a `sklearn Pipeline` with `StandardScaler`.

Features (4 quantitative, 0 categorical, 0 ordinal):
- `n_steps`, `n_ingredients`: proxies for recipe complexity
- `total_fat`, `protein`: the two macros most carefully tracked in low-carb diets like keto

Carbohydrates and sugar were intentionally left out. This mirrors the nutritional focus of keto dieters, who carefully track fat and protein while restricting carbs. A calorie tool for keto users would naturally start with just these two macros, making the baseline a realistic representation of that limited information rather than just an arbitrary modeling choice. Carbs and sugar are reserved for the final model to show what is gained by tracking them too.

**Performance:**

| Metric | Value |
|---|---|
| Train RMSE | 170.10 cal |
| Test RMSE | 159.16 cal |
| Test R-squared | 0.7721 |

The baseline explains roughly 77% of caloric variance but has a high RMSE. Carbohydrates, a primary food and fuel source at around 4 cal/g are excluded, which is the primary driver of prediction error. The final model addresses this directly.


## Final Model

### Engineered Features

Three features were added on top of the baseline:

1. **`carbs`**: Carbohydrates contribute around 4 cal/g and are the single largest caloric source in most recipes. Their absence was the primary reason for the baseline's high RMSE.
2. **`sugar`**: Sugar is a caloric subset of carbohydrates. A cake and a bread loaf may have the same total carb PDV but very different caloric profiles. Separating sugar from total carbs gives the model finer resolution.
3. **`total_fat x protein` interaction term**: High-fat and high-protein foods (ribeye steak, cheese, eggs) are more calorie-dense than either macro predicts independently. A `PolynomialFeatures(degree=2, interaction_only=True)` term captures this synergy that a purely additive model cannot represent.

### Modeling Algorithm and Feature Search

We used **Linear Regression** because the calorie calculation is linear because there is a fixed cal/g per macro. We confirmed this by testing Random Forest with GridSearchCV, which gave a test RMSE of 31.47 which is worse than even the baseline because trees must approximate a linear relationship with splits, introducing unnecessary variance.

Since Linear Regression has no meaningful hyperparameters, we performed a manual feature search comparing RMSE across combinations of features (baseline only, plus carbs, plus carbs and sugar, plus carbs and sugar and interaction term) and  went with the combination that had the lowest test RMSE.

### Performance

| Model | Test RMSE | Test R-squared |
|---|---|---|
| Baseline (Linear Regression, 4 features) | 159.16 cal | 0.7721 |
| Final (Linear Regression, 6 features + interaction) | 29.45 cal | 0.9922 |

The final model improved test RMSE from 159.16 to 29.45, a reduction of ~130 calories. Adding `carbs`, `sugar`, and the fat-protein interaction term filled in the gaps the baseline was missing. Carbohydrates were the single biggest missing piece, and the interaction term captures the extra caloric density of protein-rich fatty foods that a purely additive model misses.

<iframe src="assets/pred_vs_actual.html" width="800" height="500" frameborder="0"></iframe>


## Fairness Analysis

**Groups:**
- Group X (High-protein recipes): Top quartile of protein PDV (>= 75th percentile), the "diet-friendly" recipes a weight-loss app user cares most about
- Group Y (Low-protein recipes): Bottom quartile of protein PDV (<= 25th percentile), usually desserts, baked goods, and carb-heavy dishes

Evaluation metric: RMSE (same as overall model evaluation)

**Hypotheses:**
- Null: The model is fair. Its RMSE for high-protein and low-protein recipes is roughly the same; any difference is due to random chance.
- Alternative: The model is unfair. Its RMSE differs meaningfully between the two groups.
- Test statistic: |RMSE(high-protein) - RMSE(low-protein)|
- Significance level: alpha = 0.05

**Results:**

| Group | RMSE |
|---|---|
| High-protein recipes | 28.65 cal |
| Low-protein recipes | 44.35 cal |
| Observed difference | 15.70 cal |
| P-value | 0.0008 |

We reject the null hypothesis. The model predicts calories significantly less accurately for low-protein recipes. This likely reflects that low-protein recipes like desserts, baked goods, carb-heavy dishes, have more variable caloric density that the macro features do not fully capture. This is a meaningful limitation to disclose if this model is used for diet planning, where users could be tracking both high and low protein meals.

<iframe src="assets/fairness_test.html" width="800" height="500" frameborder="0"></iframe>
