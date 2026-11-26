# Sweet or Savory? Are Desserts Rated Higher Than Main Dishes?

**By Karsten Lowe**

---

## Introduction

Is a brownie easier to love than a pot roast? Desserts are treats, so people might rate them generously. On the other hand, baking is precise and unforgiving: a cake that doesn't rise is a disaster, while a stew that's a little off is still dinner. This project uses recipes and reviews from [food.com](https://www.food.com) to answer one question:

**Are desserts rated higher than main dishes?**

This matters to anyone choosing a recipe from star ratings. If ratings mean different things for different kinds of food, a 4.5-star dessert and a 4.5-star main dish aren't directly comparable.

The data comes in two parts:

- **Recipes:** 83,782 recipes posted since 2008, with cook time, tags, nutrition, steps, and ingredients.
- **Reviews:** 731,927 reviews, each a user's 1–5 star rating and comment on a recipe.

The columns most relevant to this question:

| Column | Description |
|---|---|
| `tags` | food.com tags for the recipe. `desserts` and `main-dish` identify the two groups |
| `rating` | The 1–5 star rating a user gave in a review |
| `avg_rating` | (created) The mean star rating across all of a recipe's reviews |
| `n_ratings` | (created) The number of star ratings a recipe received |
| `nutrition` | Calories (kcal), plus total fat, sugar, sodium, protein, saturated fat, and carbohydrates as % daily value |
| `minutes` | Minutes to prepare the recipe |
| `n_steps`, `n_ingredients` | Number of steps and number of ingredients |

---

## Data Cleaning and Exploratory Data Analysis

### Cleaning

1. **Merged recipes with reviews.** A left merge keeps every recipe, even ones with no reviews.
2. **Treated ratings of 0 as missing.** food.com ratings run from 1 to 5 stars. A 0 is recorded when someone posts a review without choosing a rating, such as a question or a comment. Counting these as 0 stars would unfairly drag averages down, so they were replaced with missing values. This affected 15,035 reviews.
3. **Computed each recipe's average rating** and number of ratings, and added both to the recipes table.
4. **Split `nutrition` into separate columns.** It was stored as text that looks like a list (`"[138.4, 10.0, ...]"`), so it was parsed into seven numeric columns, such as `calories`, `sugar_pdv`, and `protein_pdv`.
5. **Parsed `tags`, `steps`, and `ingredients`** from text into real lists, and converted `submitted` to a date.
6. **Created `course`.** Recipes tagged `desserts` became "dessert" (13,492). Recipes tagged `main-dish` became "main dish" (25,195). The 10 recipes with both tags, and all recipes with neither, became "other". The tags are consistent: every cookie, cake, and pie recipe is also tagged `desserts`. But a recipe whose author added no course tags lands in "other", even if it's clearly a dessert.

A few columns of the cleaned data:

| name | course | minutes | calories | sugar_pdv | protein_pdv | avg_rating | n_ratings |
|:---|:---|---:|---:|---:|---:|---:|---:|
| 1 brownies in the world best ever | dessert | 40 | 138.4 | 50 | 3 | 4 | 1 |
| 1 in canada chocolate chip cookies | other | 45 | 595.1 | 211 | 13 | 5 | 1 |
| 412 broccoli casserole | other | 40 | 194.8 | 6 | 22 | 5 | 4 |
| millionaire pound cake | dessert | 120 | 878.3 | 326 | 20 | 5 | 1 |
| 2000 meatloaf | main dish | 90 | 267.0 | 12 | 29 | 5 | 2 |

### Univariate Analysis

<iframe src="assets/avg-rating-hist.html" width="800" height="450" frameborder="0"></iframe>

Average ratings are heavily skewed toward the top. About 59% of rated recipes have a perfect 5.0, partly because the typical recipe has only one rating. Since almost everything scores high, any difference between desserts and main dishes is likely to be small.

### Bivariate Analysis

<iframe src="assets/stars-by-course-bar.html" width="800" height="450" frameborder="0"></iframe>

Looking at individual reviews, desserts get about **twice as many 1-star reviews** as main dishes (2.6% vs. 1.2%) and slightly more 5-star reviews. Main dishes get more 4-star reviews. Desserts seem to be love-it-or-hate-it.

<iframe src="assets/sugar-by-course-box.html" width="800" height="450" frameborder="0"></iframe>

Nutritionally, the two courses barely overlap. The middle half of desserts has 45–164% of a day's sugar per serving, compared with 8–35% for main dishes.

### Interesting Aggregates

| | dessert | main dish |
|:---|---:|---:|
| Recipes | 13,492 | 25,195 |
| Mean of average rating | 4.582 | 4.590 |
| Median of average rating | 5.000 | 5.000 |
| SD of average rating | 0.751 | 0.636 |
| Share with a perfect 5.0 | 0.585 | 0.531 |
| Share below 4 stars | 0.087 | 0.067 |
| Median calories | 287.2 | 422.1 |
| Median sugar (% DV) | 93 | 18 |
| Median protein (% DV) | 8 | 54 |

The average ratings are nearly identical. The difference is in the **spread**: desserts more often get a perfect score *and* more often fall below 4 stars.

---

## Assessment of Missingness

*Coming soon.*

---

## Hypothesis Testing

- **Null hypothesis:** Desserts and main dishes have the same distribution of average ratings. Any observed difference is due to random chance.
- **Alternative hypothesis:** Desserts have higher average ratings than main dishes.
- **Test statistic:** The mean average rating of desserts minus that of main dishes. The question asks whether desserts are rated *higher*, so the test is one-sided: large positive values support the alternative.
- **Significance level:** 0.05.

A permutation test fits this question: if the null is true, the "dessert" and "main dish" labels are arbitrary, so shuffling them shows how much the difference varies by chance alone. I shuffled the labels 10,000 times across 37,326 rated recipes.

<iframe src="assets/permutation-test.html" width="800" height="450" frameborder="0"></iframe>

The observed difference was **−0.008 stars**, so desserts actually average slightly lower. The p-value was about **0.85**, so I **fail to reject the null hypothesis**. The data doesn't provide evidence that desserts are rated higher than main dishes. This doesn't prove the two are rated the same; a difference this small is simply very consistent with chance.

---

## Framing a Prediction Problem

**Can we tell whether a recipe is a dessert or a main dish from its nutrition and structure alone?**

This is a **binary classification** problem. The response variable is `course` (dessert vs. main dish). A model like this could automatically tag recipes whose authors didn't label their course.

- **Metric: F1-score.** Only about 35% of the recipes are desserts. A model that always guessed "main dish" would score about 65% accuracy while being useless. F1 balances precision and recall and can't be inflated by the majority class.
- **Time of prediction:** The model predicts at the moment a recipe is **submitted**. At that point we know its nutrition, cook time, number of steps, and ingredients. We don't use ratings, because those come later, or tags, because the label is built from them.

---

## Baseline Model

The baseline is a **logistic regression** inside a single `sklearn` pipeline. It uses two **quantitative** features, `sugar_pdv` and `protein_pdv`, both standardized. There are no ordinal or nominal features, so no categorical encoding is needed. I trained it on 80% of the recipes and tested it on the other 20%, keeping the share of desserts the same in both.

| Metric (test set) | Score |
|:---|---:|
| F1 | 0.91 |
| Accuracy | 0.93 |
| Precision | 0.88 |
| Recall | 0.94 |

For two features this is a strong baseline, and the train and test scores are almost identical, so it isn't overfitting. Its weak point is precision. Most of its mistakes are honey- or sugar-glazed main dishes, such as BBQ ribs, honey-roasted chicken, and honey walnut shrimp, which have dessert-level sugar.

---

## Final Model

*Coming soon.*

---

## Fairness Analysis

*Coming soon.*
