# Can Kickstarter Success Be Predicted at Launch?

## Logistic regression in Alteryx using only information available before a campaign begins

**Project question:** *To what extent can information available when a Kickstarter campaign launches predict whether it will achieve its funding goal?*

This project builds an end-to-end **Alteryx** workflow that takes raw Kickstarter project files through data preparation, exploratory analysis, feature engineering, logistic-regression modelling, cross-validation and final evaluation on later campaigns.

The aim was not to build the most complex possible model. The priority was to build a model that was **interpretable, reproducible and free from obvious outcome leakage**. Only information that could reasonably be known when a campaign launches was allowed into the predictor set.

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

The Alteryx workflow uses a **Directory** tool and **Dynamic Input** to read all CSV components from the selected snapshot folder and append them into one stream. This makes the ingestion repeatable if the snapshot contains multiple component files.

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

Duplicate project IDs were investigated before a unique-project stream was created. The final modelling table contains **138,189 unique campaign IDs**.

### 6. Parse nested fields

The workflow investigates nested JSON-style fields for category, creator, location and video information. Category information was retained because it produced stable, interpretable predictors. Creator, location-detail and video fields were investigated but were not added to the final model because they were either too granular, not clearly useful, or not sufficiently defensible as stable launch-time predictors for this project.

### 7. Parent category clean-up

Some records had no parent category because the recorded category itself represented a top-level Kickstarter category. These were investigated and the parent-category field was completed for modelling.

### 8. Final QA gate

Before modelling, records were checked for:

- positive USD funding goal
- positive campaign duration
- non-negative preparation duration
- non-null target
- non-null category and country

The purpose of these checks is reproducibility: even if the current data pass a rule, the workflow will also protect a future refresh from silently passing invalid rows downstream.

---

## Feature engineering

The workflow creates a small set of features that can be understood by a non-technical reader and are available at launch.
