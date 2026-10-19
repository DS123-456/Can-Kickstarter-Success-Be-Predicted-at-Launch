# Can Kickstarter Success Be Predicted at Launch?

## Logistic regression in Alteryx using only information available before a campaign begins

**Project question:** *To what extent can information available when a Kickstarter campaign launches predict whether it will achieve its funding goal?*

This project builds an end-to-end **Alteryx** workflow that takes raw Kickstarter project files through data preparation, exploratory analysis, feature engineering, logistic-regression modelling, cross-validation and final evaluation on later campaigns.

The aim was not to build the most complex possible model. The priority was to build a model that was **interpretable, reproducible and free from obvious outcome leakage**. Only information that could reasonably be known when a campaign launches was allowed into the predictor set.

### Headline results

| Measure | Result |
|---|---:|
| Completed campaigns used | **138,189** |
| Training campaigns, launched through 2024 | **108,393** |
| Test campaigns, launched from 2025 onwards | **29,796** |
| Training success rate | **64.9%** |
| Test success rate | **73.5%** |
| Selected model | **Expanded logistic regression** |
| Cross-validated AUC | **0.770** |
| Final test accuracy | **79.6%** |
| Final test AUC | **0.7966** |
| Final test F1 | **0.8705** |
