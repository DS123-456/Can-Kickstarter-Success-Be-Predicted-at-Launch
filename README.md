# Can Kickstarter Success Be Predicted at Launch?

## Logistic regression in Alteryx using only information available before a campaign begins

**Project question:** *Can we predict whether a Kickstarter campaign will achieve its funding goal using only information available when it launches?*

This project explores that question using 138,189 completed Kickstarter campaigns and logistic regression in Alteryx.

I wanted to understand which campaign characteristics are associated with success and whether these could be used to predict the outcome of future projects.

The analysis focuses on information available at launch, including the funding goal, campaign duration, project category, preparation time and the length of the project title and description.

Two models were developed and compared before the final model was tested on campaigns launched from 2025 onwards.

**Key results:**

-The final model achieved **79.6% accuracy**, compared with 73.5% from simply predicting that every campaign would succeed.

-It correctly identified **93.3% of successful campaigns**, but only **41.6% of failed campaigns**.

-Funding goals, preparation time, campaign duration and title length were among the characteristics associated with success.

The results suggest that launch-time information can provide useful predictions, although the model is considerably better at recognising successful campaigns than identifying failures.

The aim was not to create the most complex model, but to develop an interpretable approach that could explain its predictions and be evaluated on later campaigns.

This project builds an end-to-end **Alteryx** workflow that takes raw Kickstarter project files through data preparation, exploratory analysis, feature engineering, logistic-regression modelling, cross-validation and final evaluation on later campaigns.

---

## Contents

1. [Project framing](#project-framing)
2. [Data source and scope](#data-source-and-scope)
3. [Data preparation](#data-preparation-and-leakage-control)
4. [Feature engineering](#feature-engineering)
5. [Train/test design](#traintest-design)
6. [Exploratory data analysis](#exploratory-data-analysis)
7. [Model design](#model-design)
8. [Cross-validation and model selection](#cross-validation-and-model-selection)
9. [Final model evaluation](#final-model-evaluation)
10. [Interpreting the selected model](#interpreting-the-selected-model)
11. [Limitations](#limitations)
12. [Future development](#future-development)
13. [Repository structure and reproduction](#repository-structure-and-reproduction)

---

## Project framing

Kickstarter campaigns are created with a set of decisions already made before launch: the funding goal, campaign duration, project category, preparation period, project title and short description (blurb), country and launch timing. The central question for this project was whether those **launch-time characteristics** contain enough information to estimate the probability that a project will eventually be successfully funded.

The problem is naturally suited to **binary classification**:

- `1` = successful campaign
- `0` = failed campaign

I chose **logistic regression** for three main reasons:

1. The target is binary and the model produces a probability of success between 0 and 1.
2. Logistic regression is relatively transparent: coefficients and odds ratios can be translated into understandable statements about historical associations.
3. The objective was to understand the main predictive relationships, not simply to maximise predictive accuracy with an algorithm which is harder to understand and communicate.

---

## Data source and scope

The data came from the public **Web Robots Kickstarter dataset**:

https://webrobots.io/kickstarter-datasets/

Web Robots publishes monthly Kickstarter scrapes in CSV and JSON formats. I used the **10 September 2026 CSV snapshot** rather than combining multiple monthly snapshots.

Web Robots notes that its collection process can return duplicate projects because projects may appear in multiple sub-categories. Duplicate handling was therefore treated as a deliberate data-quality step rather than assuming every row was unique.

### Primary modelling population

Only campaigns with a known final outcome were used for the main model:

- 'successful' -> `1`
- `failed` -> `0`

Projects that were live, submitted, started, cancelled or lacked the specific successful/failed outcome required by the primary question were excluded from the main modelling population.

After filtering, validation and deduplication, the dataset contained **138,189 completed campaigns**:

- **92,208 successful**
- **45,981 failed**

The unit of analysis is one Kickstarter campaign.

---

## Data preparation and leakage control

A substantial part of the project was designing a modelling dataset that could realistically have existed **at launch**.

### 1. Load and combine the source files

I used Alteryx to combine the CSV files into one dataset. The process uses Directory and Dynamic Input tools, allowing the same workflow to be used if the source files are updated.

### 2. Remove clearly post-launch information

Fields that directly or indirectly reveal how the campaign performed were removed from the predictor stream. Examples include:

- pledged amount
- USD pledged amount
- converted pledged amount
- percentage funded
- number of backers
- spotlight status
- state-change timing

Including these fields would create **target leakage** because they are only known during or after campaign performance unfolds. A model containing them might look extremely accurate while being useless for a genuine launch-time prediction.

### 3. Keep launched, completed campaigns only

The workflow removes projects that were not launched and then retains the two target states used in the primary model: successful and failed.

### 4. Convert Unix timestamps and enforce date logic

The raw timestamp fields were converted into readable dates for:

- project creation
- launch
- funding deadline

A QA rule requires:

`Created date <= Launch date <= Deadline date`

This makes the chronology explicit and prevents invalid sequences entering the feature calculations.

### 5. Investigate and remove duplicates

The source data can contain the same Kickstarter project more than once because projects may appear in multiple categories.

I checked for duplicate project IDs before removing repeated records, ensuring each campaign was represented only once in the modelling dataset.

After filtering and removing duplicates, the final dataset contained **138,189 unique campaigns**.

### 6. Investigate nested fields

Some information in the source data was stored within nested JSON fields rather than individual columns. These included details about the project category, creator, location and video content.

I investigated these fields to identify whether they contained useful information for predicting success.

Category information was retained because it provided a clear and interpretable way of comparing different types of projects.

The additional creator, location and video details were not included in the final model because they were either too detailed, showed limited usefulness, or could not be confidently treated as reliable information available at launch.

### 7. Parent category clean-up

Some campaigns had no recorded parent category because their existing category was already one of Kickstarter's top-level categories.

I investigated these records and completed the parent-category field where appropriate, ensuring campaigns could be grouped consistently for analysis and modelling.

### 8. Final QA gate

Before modelling, I carried out a final set of checks to ensure the data were suitable for analysis.

These included checking that:

- Funding goals were greater than zero.
- Campaign durations were positive.
- Preparation time was not negative.
- Each campaign had a valid success or failure outcome.
- Category and country information were available.

These checks helped prevent invalid or incomplete records from entering the model. I also kept them within the Alteryx workflow so the same rules could be applied if the dataset was updated in future.

---

## Feature engineering

The workflow creates a small set of features that can be understood by a non-technical reader and are available at launch.

### Funding goal

`Goal_USD = goal x static_usd_rate`

The raw goal was converted to USD so that goals from different currencies could be placed on a common scale.

A second field was then created:

`Log10_Goal_USD = log10(Goal_USD)`

Funding goals varied considerably, with most campaigns seeking relatively small amounts and a small number targeting millions of dollars. I used a logarithmic transformation to reduce the influence of these extreme values and make the funding goal more suitable for regression modelling. Using log10 also means that a one-unit increase represents a tenfold increase in the funding goal, which makes the resulting coefficient easier to interpret.


### Campaign timing

- `Campaign_Duration_Days = deadline - launch date`
- `Preparation_Days = launch date - creation date`
- launch month
- launch weekday
- launch year

Launch year was used for the chronological split and analysis of changes over time, not as a predictor in the final model.

### Text length

- title word count
- blurb word count

The objective was to test whether the **amount of text** contained useful signals.

### Country grouping

Country frequency was calculated from the **training data only**. Countries with fewer than **300 training campaigns** were grouped into `Other`.

The resulting training-derived mapping was then applied unchanged to the test period. A country appearing in Test without an established training category was also mapped to `Other` rather than creating a new unseen model level.

### One-hot encoding

Categorical variables and EDA-defined bands were converted into indicator fields in Alteryx before logistic regression. One category from each field was omitted as the reference level.

---

## Train/test design

The project uses a **chronological Train/Test split**, not a random split. Campaigns launched up to the end of 2024 were used to develop and compare the models, while campaigns launched from January 2025 onwards were reserved for final testing.

| Dataset | Period | Campaigns | Split | Success rate |
|---|---|---:|---:|---:|
| Train | Launch through 31 Dec 2024 | 108,393 | 78.4% | 64.9% |
| Test | Launch from 1 Jan 2025 | 29,796 | 21.6% | 73.5% |

I chose this approach because the objective was to predict outcomes for later campaigns, rather than evaluate a model on a random mixture of older and newer projects.

An important finding was that the success rate increased from 64.9% in the training data to 73.5% in the test data. This difference also meant I needed to assess performance using more than accuracy alone.

![Historical success-rate trend](images/Figure1.png)

All EDA, transformations, bands, rare-country decisions and model selection were based on **Train only**. The Test set was reserved for the final evaluation after the model specification had been locked.

---

## Exploratory data analysis

EDA was used to decide **how variables should enter the model**, not simply to produce descriptive charts. The main questions were:

- Is the variable strongly skewed?
- Does the relationship with success appear linear or non-linear?
- Are categories well populated?
- Does the variable provide a plausible launch-time signal?

### Target balance

The training data contain more successful than failed campaigns, so accuracy alone is not a sufficient evaluation metric. A majority-class classifier would already achieve **64.9% accuracy on Train** and **73.5% on Test** simply by predicting Success for every campaign.

This motivated the later use of AUC, F1 and class-level recall alongside accuracy.

### Funding goal: raw vs log scale

The funding-goal distribution is highly skewed:

- median training goal: approximately **$4,000**
- mean: approximately **$29,529**
- 95th percentile: **$50,000**
- 99th percentile: **$200,000**
- maximum: more than **$100 million**

The raw distribution is dominated by a small number of extremely large values. The log10 transformation compresses that tail and gives a much more useful modelling scale.

![Goal & log histograms](images/Figure2.png)

**Decision:** use `Log10_Goal_USD` rather than raw `Goal_USD`.

### Campaign duration

Duration is not distributed smoothly. Around **43,600 training campaigns run for exactly 30 days**, with another strong concentration at 60 days.

![Duration histograms](images/Figure3.png)

Observed success was also non-linear:

| Duration | Training campaigns | Success rate |
|---|---:|---:|
| 1-14 days | 5,383 | 74.6% |
| 15-29 days | 20,915 | 76.6% |
| 30 days | 43,600 | 61.9% |
| 31-44 days | 18,561 | 72.8% |
| 45-59 days | 8,526 | 57.0% |
| 60+ days | 11,408 | 42.8% |

A single numeric duration term would incorrectly assume that every extra day has the same directional effect on log-odds.

**Decision:** I grouped campaign duration into bands rather than treating it as a continuous variable, because the relationship with success was not consistent across durations. I used 30 days as the reference group because it was the most common campaign length.

### Preparation time

Preparation time is heavily right-skewed. In Train:

- median = **13 days**
- 90th percentile = **105 days**
- 99th percentile = about **650 days**
- maximum = more than **3,900 days**

![Preparation histograms](images/Figure4.png)

The relationship with success is clearly non-linear. Success rises through the early preparation bands, peaks in the 31-106 day group, then falls slightly for the longest-preparation campaigns.

| Preparation time | Success rate |
|---|---:|
| 0-1 day | 45.8% |
| 2-5 days | 57.0% |
| 6-13 days | 68.0% |
| 14-30 days | 72.7% |
| 31-106 days | 73.6% |
| 107+ days | 68.2% |

**Decision:** use preparation-time bands, with **31-106 days as the reference group**.

### Title length

Title length showed a comparatively smooth positive relationship with success. For example, the observed success rate rose from approximately **56% for 1-3 word titles** to around **73% for 10-12 words**.

![Title word count success rate](images/Figure5.png)

The log-odds relationship was sufficiently regular to keep this variable continuous.

**Decision:** use `Title_Word_Count` as a numeric predictor.

### Blurb length

Blurb length showed an inverted-U pattern rather than a linear relationship.

![Blurb word count success rate](images/Figure6.png)

| Blurb length | Success rate |
|---|---:|
| 1-5 words | 58.7% |
| 6-10 words | 65.5% |
| 11-15 words | 69.4% |
| 16-20 words | 66.0% |
| 21-25 words | 62.5% |
| 26-29 words | 57.9% |
| 30+ words | 53.5% |

**Decision:** use blurb-length bands, with **16-20 words as the reference group**.

### Parent category

Success rates varied meaningfully across the 15 parent categories in the training data. Film & Video, Dance, Music and Art were among the higher observed rates, while Fashion, Food and Journalism were much lower.

![Success rate by parent category](images/Figure7.png)

The variation is large enough to justify including parent category as a categorical predictor in the multivariable model.

### Country, launch month and weekday

Country and launch timing also showed differences in observed success. Sparse countries were grouped into 'Other' using the Train-only rule described earlier. Month and weekday were retained as categorical variables because they provide low-complexity launch-time context.

![Success rate for country, month and weekday](images/Figure8.png)

### Correlation checks

Numeric candidate fields were checked for obvious pairwise correlation problems. No issue was large enough to require removing a final numeric predictor. The main transformation decisions came from distribution shape and the relationship with success rather than from correlation alone.

---

## Model design

Two logistic-regression iterations were developed using the same training data.

### Core model

The Core model represents information about the campaign design and launch context:

- Log10_Goal_USD
- duration band
- parent category
- country
- launch month
- launch weekday

### Expanded model

The Expanded model contains all Core predictors plus three additional features related to preparation and presentation:

- preparation-time band
- title word count
- blurb-length band

The purpose of the comparison was simple: **do these additional variables add enough predictive value to justify the extra complexity?**

### Stepwise robustness check

Stepwise AIC versions of both models were also created as a robustness check. The Expanded stepwise version removed only a small number of weaker country/month indicators and produced almost identical predictive performance and coefficient estimates.

The full Expanded model was retained because the stepwise version produced almost identical results, so simplifying the model further offered little practical benefit.

---

## Cross-validation and model selection

To compare the models fairly, I used repeated five-fold cross-validation on the training data. This meant that each model was evaluated on data not used to fit it, rather than relying solely on its performance on the full training dataset.

| Metric | Core | Expanded | Improvement |
|---|---:|---:|---:|
| Accuracy | 0.7149 | **0.7425** | +0.0276 |
| AUC | 0.7288 | **0.7697** | +0.0409 |
| F1 | 0.8010 | **0.8167** | +0.0157 |
| Failure recall | 0.4042 | **0.4833** | +0.0791 |
| Success recall | **0.8821** | 0.8820 | effectively unchanged |

![Cross validation](images/Figure9.png)

The Expanded model performed better across the main evaluation measures.

In particular, **failure recall improved from 40.4% to 48.3%**, while the ability to identify successful campaigns remained almost unchanged at approximately 88.2%.

This was an important improvement because identifying potential failures was already more difficult than identifying successes.

I therefore selected the Expanded model for final testing on later campaigns. A separate stepwise model comparison showed that removing a small number of weaker predictors offered little additional benefit, so I retained the full Expanded model.

---

## Final model evaluation

### Final metrics

| Metric | Test result |
|---|---:|
| Accuracy | **79.60%** |
| AUC | **0.7966** |
| F1 | **0.8705** |
| Failure recall | **41.61%** |
| Success recall | **93.32%** |
| Majority-class baseline accuracy | **73.5%** |

### Confusion matrix

![Confusion matrix](images/Figure10.png)

The test model performed well overall, but the class-level results are important. It correctly identified more than 93% of successful campaigns, while failure recall remained much lower at about 42%.

This matters because a high overall accuracy figure can hide weaker performance on the minority or more difficult class.

### Performance

The final model was evaluated on 29,796 campaigns launched from January 2025 onwards. These campaigns had not been used during model development or selection.

The model achieved **79.6% accuracy**, meaning it correctly classified approximately eight out of ten campaigns.

However, accuracy needs to be considered alongside the overall success rate. As 73.5% of campaigns in the test dataset were successful, a model predicting success for every campaign would already achieve 73.5% accuracy.

The logistic regression model therefore improved accuracy by approximately **6.1 percentage points** over this simple baseline.

The results were also quite different for successful and failed campaigns.

The model correctly identified **93.3% of successful campaigns**, but only **41.6% of failed campaigns**. This means it was much better at recognising projects that would succeed than identifying those that would fail.

![Precision and Recall](images/Figure11.png)

The charts compare the model's ability to correctly identify successful and failed campaigns.

Precision shows how often predictions for each outcome were correct, while recall shows the proportion of actual campaigns in each class that the model successfully identified.

The results highlight the model's stronger performance at identifying successful campaigns, with substantially lower recall for failed campaigns. This is an important limitation when considering its potential use for identifying projects at risk of not achieving their funding goals.

The model also achieved an **AUC of 0.797**, indicating a reasonable ability to distinguish between campaigns with higher and lower likelihoods of success.

![ROC curve](images/Figure12.png)

Overall, these results suggest that campaign characteristics available at launch contain useful predictive information. However, the lower failure recall is an important limitation, particularly if the model were used to identify campaigns at risk of missing their funding goal.

---

## Interpreting the selected model

The final Expanded model is useful because the main coefficients can be translated into odds ratios.

The categorical results below are relative to reference groups:

- Parent category: **Film & Video**
- Country: **US**
- Launch month: **October**
- Launch weekday: **Tuesday**
- Duration: **30 days**
- Preparation: **31-106 days**
- Blurb length: **16-20 words**

Selected odds ratios from the Expanded logistic regression model:

| Predictor / comparison | Odds ratio | Interpretation |
|---|---:|---|
| Tenfold increase in funding goal | **0.383** | About 62% lower odds of success, holding other predictors constant |
| One additional title word | **1.100** | About 10% higher odds per additional word within the observed range |
| Duration 15-29 days vs 30 days | **1.786** | About 79% higher odds |
| Duration 60+ days vs 30 days | **0.528** | About 47% lower odds |
| Preparation 0-1 day vs 31-106 days | **0.205** | About 80% lower odds |
| Preparation 14-30 days vs 31-106 days | **0.833** | About 17% lower odds |
| Blurb 11-15 words vs 16-20 | **1.202** | About 20% higher odds |
| Blurb 30+ words vs 16-20 | **0.582** | About 42% lower odds |

Odds ratios were calculated from the logistic regression coefficients using `exp(coefficient)`.

Several characteristics stood out as useful indicators of whether a campaign would achieve its funding goal.:

### Funding goal is a strong predictor

Campaigns with larger funding goals generally had lower odds of success, after accounting for the other variables in the model. A tenfold increase in the funding goal was associated with approximately 62% lower odds of success.

### Preparation time contains useful signal

Campaigns launched within one day of being created had substantially lower odds of success than those prepared for 31–106 days. This suggests that preparation time may contain useful information about a campaign's likelihood of success.

However, it does not prove that waiting longer before launching directly improves the chances of success.

### Very long durations are associated with weaker outcomes

Campaigns lasting 60 days or more had lower odds of success than 30-day campaigns, suggesting that longer funding windows were not necessarily associated with better outcomes.

### Text length has a measurable but non-causal association

Title length showed a positive relationship with success, while blurb length showed a more complex pattern. Moderate-length descriptions generally performed better than very short or very long ones.

These findings show how logistic regression can help explain relationships within the data, rather than simply producing a predicted outcome.

---

## What the model can and cannot tell us

### It can

- rank campaigns from lower to higher estimated probability of success
- identify launch-time characteristics associated with historical success
- provide interpretable probability estimates
- support discussion about campaign design and risk

### It cannot

- prove that changing a feature will cause a different outcome
- capture unobserved creator reputation, audience size or external promotion
- guarantee that a campaign with a high predicted probability will succeed
- assume that Kickstarter behaviour will remain stable indefinitely

---

## Limitations

### Source-data representativeness

The project uses a public Web Robots scrape, not a controlled random sample. Web Robots also notes historic collection limitations and duplicate possibilities arising from category-based scraping. Results should therefore be interpreted within the observed dataset rather than treated as exact population statistics for all Kickstarter activity.

### Changes over time

The success rate increased from 64.9% in Train to 73.5% in Test. Although AUC remained strong, class-level behaviour changed. Any deployed model would need ongoing monitoring of success rates, predictor distributions and class-level performance.

### Failure recall

The final model identifies successful campaigns much more effectively than failed campaigns. Test failure recall was **41.6%**. The model should therefore not be presented as a reliable filter for rejecting campaigns that will fail.

### Deliberately limited feature set

The workflow does not use project-text semantics, images, creator history, social activity or post-launch behaviour. Some of these could improve prediction, but they would also increase complexity and, in the case of post-launch measures, risk leakage.

---

## Future development

The project could be extended in several directions without changing the core workflow:

1. **Validate on a later independent snapshot** to see whether model performance remains stable.
2. **Monitor change over time** in country mix, category mix, funding goals and overall success rates.
3. **Add creator history** if it can be calculated strictly from information available before each campaign launch.
4. **Test text features** from campaign titles/descriptions, while keeping a simpler logistic model as an interpretable benchmark.

---

## Summary

This project shows that launch-time Kickstarter information contains meaningful predictive signal without using post-launch performance data. The Expanded logistic-regression model was selected because its advantage over the Core model persisted under repeated cross-validation. On campaigns launched from 2025 onwards it achieved **79.6% accuracy, 0.7966 AUC and 0.8705 F1**, outperforming the 73.5% majority-class baseline.

The strongest value of the project is not the headline accuracy alone. The workflow demonstrates the full process from raw public data through leakage control, EDA-led feature design, model comparison and final evaluation on later campaigns, while keeping the final model interpretable enough to explain what it has learned and where its limitations remain.
