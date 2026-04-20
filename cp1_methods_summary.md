# Detailed Summary of Statistical Tests, Analysis Tools, and Visualisation Methods
## Hindi Mental Lexicon Project — Behavioural Research Methods and Statistical Models

---

## Overview

This document provides a comprehensive explanation of every statistical test, analysis tool, and visualisation method used in this project, along with the reasoning behind each choice. It is organised by analysis stage rather than by hypothesis, so you can trace the full logical flow of the project.

---

## Part 1 — Data Cleaning and Preprocessing

### 1.1 Manual Normalisation Dictionary

**What it is:**
A hand-crafted Python dictionary mapping 337 Roman script entries (transliterations, English equivalents, and spelling variants) to their canonical Devanagari forms. Nine noise words were mapped to `None` and removed.

**Why it was used:**
62.9% of all word responses were typed in Roman script rather than Devanagari. Without normalisation, `kutta`, `dog`, and कुत्ता were treated as three completely different words by every subsequent analysis. This fragmented semantic clusters, inflated vocabulary size artificially, and suppressed statistical results. A dictionary-based approach was chosen over automatic transliteration tools because the dataset was small enough (657 unique Roman tokens) to handle manually with high accuracy — automatic tools frequently make errors on informal Hindi transliteration where spelling is highly variable.

**Alternatives considered:**
- Fuzzy string matching (rapidfuzz) — useful for catching near-duplicate spellings, but cannot resolve meaning (e.g., it cannot know that `sher` = शेर without being told)
- Automatic transliteration APIs — exist but perform poorly on informal mixed-script text
- Manual + fuzzy hybrid — the approach actually taken: manual for meaning mappings, pattern-matching for typo variants

---

### 1.2 IQR-Based Outlier Detection

**What it is:**
The Interquartile Range (IQR) method flags values as outliers if they fall below Q1 − 1.5×IQR or above Q3 + 1.5×IQR, where Q1 and Q3 are the 25th and 75th percentiles respectively.

**Why it was used:**
IRT data is reaction time data, which is known to be susceptible to distraction events — a participant might pause for 42 seconds between two words simply because they were interrupted. These extreme values inflate means and can distort regression coefficients. The IQR method was chosen over Z-score based outlier detection because IRT data is right-skewed (skewness = 2.698), and Z-scores assume a normal distribution. The IQR method is non-parametric and therefore more appropriate for skewed data.

**Results:**
- Animals: 15 outliers (4.1%), upper threshold 13.80s
- Foods: 21 outliers (6.6%), upper threshold 15.00s
- Body-Parts: 11 outliers (5.3%), upper threshold 14.91s
- Colours: 6 outliers (4.1%), upper threshold 9.83s

The ANOVA was rerun after outlier removal (F=3.012, p=0.034) and remained significant, confirming the domain effect was not driven by extreme values.

---

## Part 2 — Clustering Method

### 2.1 K-Means Clustering (k=4)

**What it is:**
K-means is an unsupervised machine learning algorithm that partitions data points into k groups by minimising the within-cluster sum of squared distances to the cluster centroid. Applied here to each participant's SpAM (x, y) coordinates to generate semantic cluster labels.

**Why it was used:**
Each participant placed their words on a 2D canvas according to semantic similarity. K-means on these coordinates provides a principled, data-driven way to identify which words the participant grouped together, without requiring the researcher to manually inspect each canvas. K-means is appropriate here because:
- The input is 2D Euclidean space (SpAM coordinates), which K-means handles naturally
- We want a fixed number of non-overlapping clusters per participant
- The algorithm is deterministic given a random seed, making results reproducible

**Why k=4:**
The choice of k=4 was theoretically motivated — each domain has approximately four broad semantic subgroupings (e.g., Animals: domestic, wild, birds, aquatic). This is an acknowledged limitation: the optimal k was not empirically validated using the elbow method or silhouette scoring, which is recommended for future work.

**Alternative:** Hierarchical clustering or DBSCAN could be used instead. DBSCAN has the advantage of not requiring k to be specified in advance and can handle non-spherical clusters, but requires tuning of density parameters which is harder to justify for small per-participant datasets.

---

## Part 3 — Hypothesis Tests

### 3.1 Independent Samples T-Test (H1 — original)

**What it is:**
Tests whether the means of two independent groups differ significantly. Computes a t-statistic as the ratio of the mean difference to the standard error of the difference.

**Why it was used initially:**
To compare within-cluster IRTs vs between-cluster IRTs — two independent groups of continuous values — per domain. This is the standard first choice for comparing two group means.

**Why it was later replaced:**
Levene's test for homogeneity of variance failed for all four domains (all p < 0.05), indicating that within-cluster and between-cluster IRT groups have unequal variances. The standard t-test assumes equal variances and is therefore not the most appropriate test here.

---

### 3.2 Welch's T-Test (H1 — primary test after assumption check)

**What it is:**
A variant of the independent samples t-test that does not assume equal variances. It adjusts the degrees of freedom downward using the Welch-Satterthwaite equation, making it more conservative when variance is unequal.

**Why it was used:**
Levene's test confirmed unequal variances across all four H1 domains. Welch's t-test is the recommended alternative in this case and is increasingly considered the default t-test in modern statistics (Delacre et al., 2017). The formula in Python is simply `stats.ttest_ind(group1, group2, equal_var=False)`.

**Key result that changed:** Body-Parts moved from borderline (standard t: p=0.0095) to significant (Welch's t: p=0.0041), crossing the Bonferroni threshold. This is a genuine finding that would have been missed without assumption checking.

---

### 3.3 Mann-Whitney U Test (H1 — non-parametric robustness check)

**What it is:**
A non-parametric test that compares the distributions of two independent groups by ranking all observations together and testing whether one group tends to have higher ranks than the other. It makes no assumptions about the shape of the distribution.

**Why it was used:**
Shapiro-Wilk tests confirmed that all H1 groups were non-normally distributed (all p < 0.05). Since both normality and equal-variance assumptions were violated, a non-parametric test was run alongside the Welch's t-test as a robustness check. If both tests agree, the finding is robust regardless of distributional assumptions.

**Reasoning for the three-test approach:**
- Standard t-test: reported for direct comparison with Report 1
- Welch's t-test: preferred when variances are unequal (which Levene's confirmed)
- Mann-Whitney U: preferred when normality is violated (which Shapiro-Wilk confirmed)
- Agreement across all three = fully robust; disagreement = interpret with caution

**Key finding from disagreement:** Colours showed Welch's t borderline (p=0.008) but Mann-Whitney non-significant (p=0.21). This disagreement, combined with N=11, indicated the parametric result was likely driven by a few extreme IRT values rather than a genuine distributional shift.

---

### 3.4 Pearson Correlation (H2 and H3)

**What it is:**
Measures the strength and direction of the linear relationship between two continuous variables. Produces r (ranging −1 to +1) and a p-value testing whether r differs significantly from zero.

**Why it was used for H2:**
Both variables — SpAM neighbourhood distance and mean IRT per word — are continuous, and the hypothesis predicts a specific linear relationship (greater isolation → slower retrieval). Pearson correlation directly quantifies this relationship.

**Why it was used for H3:**
Both proficiency ratings (1–5) and retrieval outcomes (total words, mean IRT) are treated as continuous variables. Pearson correlation is appropriate for testing whether two continuous variables co-vary linearly.

**Limitation acknowledged:** Pearson assumes a linear relationship and is sensitive to outliers. For H3, the range restriction in proficiency scores (most participants rating themselves 4–5/5) limited the power to detect any relationship even if one existed in the population.

---

### 3.5 One-Way ANOVA (Domain comparison)

**What it is:**
Analysis of Variance tests whether the means of three or more independent groups differ significantly. It computes an F-statistic as the ratio of between-group variance to within-group variance. A significant F indicates that at least one group mean differs from the others.

**Why it was used instead of multiple t-tests:**
Comparing four domains pairwise would require six t-tests. Running six tests at α=0.05 gives approximately a 26% chance of at least one false positive (family-wise error). ANOVA controls this by testing all groups simultaneously with a single F-test, keeping the overall error rate at 5%.

**Why Levene's test was important here:**
ANOVA assumes equal variances across groups. Levene's test passed for the ANOVA groups (F=1.327, p=0.270), confirming this assumption was satisfied — unlike the H1 t-test groups.

**Effect size reported:** η² (eta-squared) = 0.093, a medium effect. η² represents the proportion of total IRT variance explained by domain membership, making it more interpretable than the F-statistic alone.

---

### 3.6 Bonferroni Post-Hoc Pairwise T-Tests

**What it is:**
After a significant ANOVA, pairwise t-tests between all domain pairs identify which specific groups differ. Bonferroni correction divides the significance threshold by the number of comparisons (0.05/6 = 0.0083) to control the family-wise error rate.

**Why Bonferroni specifically:**
Bonferroni is the most conservative and simplest correction method — multiply each p-value by the number of comparisons (or equivalently, divide α). It was chosen because the number of comparisons (6) is small and the method is transparent and widely understood. More powerful alternatives like Tukey's HSD or Holm's method could be used but were not necessary given the small number of comparisons.

**Limitation:** Bonferroni becomes overly conservative as the number of comparisons grows. For 12 tests (3 hypotheses × 4 domains), α = 0.0042 is a strict threshold, which is why some genuine effects are classified as "borderline."

---

### 3.7 Multiple Linear Regression

**What it is:**
Models a continuous outcome (IRT) as a linear combination of multiple predictors simultaneously. Each predictor gets a coefficient representing its unique contribution after controlling for all other predictors. R² measures the proportion of outcome variance explained by the model.

**Why it was used:**
H3's simple correlations could only test one predictor at a time. Regression allows simultaneous modelling of word position, domain, and proficiency — controlling for confounds. For example, the proficiency effect (β=+0.490) only becomes visible when word position (which is strongly negatively correlated with IRT) is held constant. This is a key advantage over simple correlation.

**Dummy coding for domain:**
Domain is a categorical variable with four levels. Regression requires numeric inputs. Dummy coding creates three binary variables (is_foods, is_body_parts, is_colours) with Animals as the reference category. Each coefficient then represents the IRT difference of that domain compared to Animals, after controlling for other predictors.

**R² = 0.099:** Explaining 9.9% of IRT variance with three predictors is reasonable for behavioural timing data, which has high inherent noise from individual differences, attention fluctuations, and typing speed variation.

---

### 3.8 Gamma Generalised Linear Model (GLM) with Log Link

**What it is:**
A GLM is a flexible extension of linear regression that allows the outcome variable to follow distributions other than the normal. The Gamma family models positive, right-skewed continuous data. The log link means the model predicts log(IRT) as a linear combination of predictors, which is equivalent to modelling IRT as a multiplicative function of predictors after exponentiation.

**Why it was used:**
Linear regression makes three assumptions that are violated by IRT data:
1. **Normality of residuals** — IRT is right-skewed (skewness = 2.698); residuals are not normal
2. **Homoscedasticity** — IRT variance tends to scale with the mean (larger IRTs have more variance)
3. **Unbounded outcome** — linear regression can predict negative IRT values, which is impossible

The Gamma distribution with log link naturally handles all three violations: it produces positive-only predictions, accommodates right skew, and allows variance to scale with the mean. It is the standard recommendation for reaction time modelling in psycholinguistics (Lo & Andrews, 2015).

**Exponentiated coefficients as multipliers:**
Because of the log link, coefficients are on the log scale. Exponentiating them converts to IRT multipliers — e.g., exp(-0.049) = 0.952 means each additional word position multiplies IRT by 0.952 (4.8% faster). This is more interpretable for right-skewed data than additive seconds.

**Validation purpose:** The GLM was run primarily to verify that the linear regression findings were not artefacts of distributional misspecification. Full agreement across both models (same three significant predictors, same two non-significant predictors, same directions) confirmed robustness.

---

## Part 4 — Assumption Checking Tools

### 4.1 Shapiro-Wilk Test (Normality)

**What it is:**
Tests whether a sample comes from a normally distributed population. Produces a W-statistic (close to 1 = normal) and a p-value. H₀: data are normally distributed. A p < 0.05 rejects normality.

**Why it was used:**
Normality is an assumption of both the t-test and ANOVA. Shapiro-Wilk was chosen over alternatives (Kolmogorov-Smirnov, Anderson-Darling) because it has better statistical power for small to moderate sample sizes, which is relevant for the per-domain groups.

**Result and consequence:**
All H1 groups failed normality (all p < 0.05), which is typical for raw reaction time data. This motivated the use of Mann-Whitney U as a robustness check. For ANOVA groups, Animals and Body-Parts passed; Foods and Colours failed (the latter likely due to small N=11).

---

### 4.2 Levene's Test (Homogeneity of Variance)

**What it is:**
Tests whether two or more groups have equal variances. H₀: all groups have equal variance. A p < 0.05 indicates unequal variances (heteroscedasticity).

**Why it was used:**
Both the standard t-test and ANOVA assume equal variances across groups. Levene's test was chosen over Bartlett's test because Levene's is more robust to non-normality — it uses absolute deviations from the group median rather than squared deviations, making it appropriate even when the normality assumption is violated (which it was for H1 groups).

**Result and consequence:**
- H1 groups: all four domains failed (all p < 0.05) → Welch's t-test used instead of standard t-test
- ANOVA groups: passed (F=1.327, p=0.270) → standard ANOVA appropriate

---

## Part 5 — Effect Size Measures

### 5.1 Cohen's d (T-Tests)

**What it is:**
Measures the standardised difference between two group means, expressed in units of pooled standard deviation. Computed as (mean₁ − mean₂) / pooled SD.

**Benchmarks (Cohen, 1988):**
- |d| < 0.2 → negligible
- 0.2–0.5 → small
- 0.5–0.8 → medium
- > 0.8 → large

**Why it was used:**
P-values tell you whether an effect is statistically distinguishable from zero, but not whether it is practically meaningful. Cohen's d provides the magnitude of the effect independently of sample size. A result can be highly significant (small p) but practically trivial (tiny d) if the sample is large enough.

**Results:**
- Animals: d = -0.418 (small)
- Foods: d = -0.643 (medium)
- Body-Parts: d = -0.426 (small)
- Colours: d = -0.404 (small)

Foods has the largest effect — the clustering effect is not just statistically significant but practically meaningful in this domain.

---

### 5.2 Eta-Squared η² (ANOVA)

**What it is:**
The proportion of total variance in the outcome explained by the group factor. Computed as SS_between / SS_total where SS = sum of squares.

**Benchmarks (Cohen, 1988):**
- η² < 0.01 → negligible
- 0.01–0.06 → small
- 0.06–0.14 → medium
- > 0.14 → large

**Why it was used:**
η² is the standard effect size for ANOVA, directly interpretable as the percentage of variance explained by domain membership. η² = 0.093 means domain accounts for 9.3% of IRT variance — a medium effect that is practically meaningful despite the modest F-statistic.

---

## Part 6 — Confidence Intervals

### 6.1 95% Confidence Intervals for Mean IRT

**What it is:**
A range within which the true population mean IRT is estimated to lie with 95% confidence. Computed as mean ± t*(α/2, df=n-1) × SE, where SE = standard deviation / √n.

**Why it was used:**
Confidence intervals are more informative than p-values alone because they convey both the direction and precision of an estimate. A narrow CI indicates a precise estimate; a wide CI (as seen for Colours, spanning 2.03s) indicates high uncertainty — usually because N is small.

**Why CIs are important here:**
The non-overlapping CIs between Colours [3.20, 5.23] and Foods [5.99, 8.06] / Body-Parts [6.00, 8.42] provide a visual confirmation of the ANOVA post-hoc findings. When two CIs do not overlap, the corresponding groups are almost certainly significantly different — allowing the reader to assess significance visually without needing to read p-values.

---

## Part 7 — Visualisation Methods

### 7.1 Grouped Bar Chart (H1)

**What it shows:** Mean within-cluster and between-cluster IRTs side by side for each domain, with significance annotations.

**Why this type:**
The t-test compares two group means. A grouped bar chart is the most direct visual representation of this comparison — each domain gets two bars (within and between), and the height difference is the effect size. Significance markers (green ✓ for Bonferroni-significant, orange * for borderline) allow the reader to assess statistical outcomes without reading the table. This is the standard visualisation for paired mean comparisons in psycholinguistics reports.

**File:** `h1_normalised.png`

---

### 7.2 Scatter Plot with Trend Line (H2 and H3)

**What it shows:** One dot per word (H2) or per participant (H3), with one variable on each axis and a linear regression line overlaid.

**Why this type:**
Pearson correlation tests a linear relationship between two continuous variables. The scatter plot is the natural visualisation for this — it shows every data point (not just the summary statistic), allows the reader to assess whether the relationship is genuinely linear, and reveals influential outlier points that might be driving any observed trend. The trend line slope directly corresponds to the correlation direction (positive slope = positive r).

**Files:** `h2_normalised.png`, `h3_normalised.png`

---

### 7.3 Boxplot with Jittered Data Points (ANOVA)

**What it shows:** Distribution of mean IRT per participant per domain, showing median, IQR, whiskers, and individual data points.

**Why this type:**
The ANOVA compares distributions across four groups. A boxplot is more informative than a bar chart of means because it shows the full distribution — median, spread (IQR), range, and outliers. The overlaid jittered individual data points reveal sample size differences visually (Colours visibly has fewer points than Animals/Foods) and allow the reader to see where individual participants fall. The whiskers extend to 1.5×IQR, making outliers visible as individual dots beyond the whiskers.

**Why not a violin plot:** Violin plots would be more informative for large samples but can be misleading for small groups like Colours (N=11). Boxplots are more interpretable at these sample sizes.

**File:** `anova_domains.png`

---

### 7.4 Horizontal Coefficient Plot (Regression and GLM)

**What it shows:** One horizontal bar per predictor, with bar length = coefficient magnitude and bar direction = sign (left = negative/faster, right = positive/slower). Error bars show ±1 SE. Green = significant, coral = not significant.

**Why this type:**
The coefficient plot is the standard visualisation for regression results. It allows the reader to simultaneously assess three properties of each predictor: direction of effect (which side the bar extends to), magnitude (how long the bar is), and uncertainty (width of error bars). Displaying linear regression and GLM side by side on the same layout allows direct comparison — if bars point in the same direction with similar relative sizes, results are consistent across models.

**Why horizontal rather than vertical:** With five predictors and long label names, horizontal bars prevent label overlap and are easier to read.

**File:** `glm_vs_linear_fixed.png`

---

### 7.5 Histogram + Point Plot with CI Bars (IRT Distribution)

**What it shows:**
- Left panel: Frequency histogram of all 1,035 IRT values with mean (red dashed) and median (orange dashed) lines marked
- Right panel: Mean IRT per domain as a point with 95% CI bars

**Why the histogram:**
The histogram directly visualises the distributional shape of IRT data — the right skew (skewness = 2.698), the long tail extending to 42 seconds, and the mean-median gap (5.57s vs 4.43s). This provides the empirical justification for using the Gamma GLM over linear regression, making the modelling choice visually transparent rather than just asserted.

**Why the CI plot rather than a bar chart:**
CI plots (point + error bar) are more statistically honest than bar charts of means. Bar charts implicitly suggest that the value starts at zero, which is misleading for a mean IRT around 6 seconds. CI plots show the estimate and its uncertainty directly. The non-overlapping CIs for Colours vs Foods/Body-Parts provide immediate visual confirmation of the ANOVA post-hoc findings.

**File:** `irt_distribution_ci.png`

---

## Part 8 — Python Libraries Used

| Library | Purpose | Why chosen |
|---|---|---|
| `numpy` | Array operations, means, SDs, percentiles | Standard scientific computing library; efficient for numerical operations on arrays |
| `scipy.stats` | T-tests, ANOVA, Shapiro-Wilk, Levene's, Pearson correlation, Mann-Whitney U | Comprehensive statistical testing library with well-validated implementations |
| `sklearn.cluster.KMeans` | K-means clustering on SpAM coordinates | Industry-standard implementation; supports random seed for reproducibility |
| `statsmodels` | Gamma GLM with log link | The only major Python library with a full GLM implementation including proper family/link specification and model diagnostics |
| `matplotlib` | All plots and figures | Most flexible Python plotting library; allows fine-grained control over every plot element |
| `pandas` | Data manipulation and groupby operations | Makes it easy to filter, group, and aggregate the dataset by participant and domain |
| `json` | Loading the raw `hindi.json` data file | Standard library; no external dependency needed |
| `collections.Counter` | Word frequency counting | Efficient frequency counting for vocabulary analysis |
| `scipy.spatial.distance.cdist` | Pairwise Euclidean distance matrix for SpAM neighbourhood computation | Vectorised computation of all pairwise distances in one call, much faster than nested loops |
| `matplotlib.font_manager` | Loading and registering the Noto Sans Devanagari font | Required to render Hindi script correctly in matplotlib plots |

---

## Summary Table — All Tests Used

| Test | Applied to | Assumption | Why appropriate | Key result |
|---|---|---|---|---|
| Shapiro-Wilk | H1 IRT groups, ANOVA groups | — | Tests normality before choosing parametric/non-parametric tests | All H1 groups non-normal; ANOVA: 2/4 pass |
| Levene's test | H1 groups, ANOVA groups | — | Tests equal variance before choosing t-test variant | H1: all fail → Welch's used; ANOVA: passes |
| IQR outlier detection | All IRT values per domain | Non-parametric | Right-skewed data; IQR robust to skew unlike Z-score | 4–7% outliers; ANOVA stable after removal |
| K-means clustering | SpAM coordinates | Euclidean space | Principled cluster derivation from participant similarity judgements | k=4 clusters per participant per domain |
| Welch's t-test | Within vs between cluster IRTs | Independent groups, continuous | Appropriate when Levene's fails; preferred modern default | Animals ✓, Foods ✓, Body-Parts ✓, Colours ✗ |
| Mann-Whitney U | Within vs between cluster IRTs | Non-parametric | Appropriate when normality fails; robustness check | Animals ✓, Foods ✓, Body-Parts ~, Colours ✗ |
| Pearson correlation | SpAM distance vs IRT; proficiency vs retrieval | Linear relationship, continuous | Both variables continuous; linear relationship predicted | H2: all non-significant; H3: all non-significant |
| One-way ANOVA | Mean IRT across domains | Normal, equal variance | Comparing >2 groups without inflating Type I error | F=3.466, p=0.019, η²=0.093 |
| Bonferroni post-hoc | ANOVA domain pairs | — | Controls family-wise error for 6 comparisons | Foods vs Colours ✓, Body-Parts vs Colours ✓ |
| Multiple linear regression | IRT ~ position + domain + proficiency | Normal residuals, linearity | Simultaneous control of confounders; identifies joint predictors | R²=0.099; position, proficiency, Colours significant |
| Gamma GLM (log link) | IRT ~ position + domain + proficiency | Positive, right-skewed outcome | Appropriate for reaction time data; distributional robustness check | Full agreement with linear regression |
| Cohen's d | H1 t-test pairs | — | Standardised effect size for two-group comparisons | Foods: medium (d=0.643); others: small |
| Eta-squared η² | ANOVA | — | Proportion of variance explained by group factor | η²=0.093, medium effect |
| 95% Confidence Intervals | Mean IRT per domain | Normal sampling distribution | More informative than p-values; shows precision of estimates | Colours CI does not overlap Foods/Body-Parts |
