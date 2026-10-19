# Can Kickstarter Success Be Predicted at Launch?

## Logistic regression in Alteryx using only information available before a campaign begins

**Project question:** *To what extent can information available when a Kickstarter campaign launches predict whether it will achieve its funding goal?*

This project builds an end-to-end **Alteryx** workflow that takes raw Kickstarter project files through data preparation, exploratory analysis, feature engineering, logistic-regression modelling, cross-validation and final evaluation on later campaigns.

The aim was not to build the most complex possible model. The priority was to build a model that was **interpretable, reproducible and free from obvious outcome leakage**. Only information that could reasonably be known when a campaign launches was allowed into the predictor set.

---Detailed GitHub 

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

