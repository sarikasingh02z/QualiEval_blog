# QualiEval: What Really Happens When Your Training Data Is Broken?

*A 94.86% F1 score, a 97.2/100 robustness rating — and a framework for understanding why data quality kills ML models before deployment ever does.*
---

If you've ever trained a model that performed beautifully in the notebook but behaved strangely in production, you've probably blamed the model.

You probably shouldn't have.

Most ML failures aren't architecture failures. They're **data failures** — and they're quiet. No error messages. No obvious crash. Just a model that slowly becomes less reliable while you keep wondering why.

This project started with one question I couldn't get a straight answer to:

> *What actually happens to an ML model when your data is bad? Not theoretically — actually.*

So I built QualiEval to find out.

---

## The Core Problem: Textbooks Don't Show You the Numbers

Every ML course covers data quality. Missing values, class imbalance, label noise — the theory is well documented.

What's rarely shown is the **measurable impact**. How much does 15% missing data actually hurt? How does a model degrade under label noise versus outliers? Which algorithms survive bad data and which ones collapse?

These aren't rhetorical questions. In real-world datasets — network traffic, fraud detection, medical records — data quality issues are the norm, not the exception. And the gap between a clean training set and messy production data is where most models silently fail.

---

## The Dataset

I used **UNSW-NB15** — a network intrusion detection dataset.

- **247,872 total samples** across train and test sets
- **49 network traffic features**
- **Binary target:** Normal (0) vs Attack (1)
- **Inherent class imbalance** — making it a realistic, production-like dataset

This wasn't chosen for convenience. Imbalanced, real-world datasets are exactly where data quality problems hit hardest — and where default ML practices fail first.

---

## The Experiment: Two Versions of the Same Dataset

Instead of just training on clean data, I created two parallel versions:

**Version 1 — Clean:** UNSW-NB15 as-is, properly preprocessed.

**Version 2 — Faulty:** Same dataset with deliberately injected faults.

 **Fault**          |  **Amount Injected**    |   **Why It Hurts** 

 Missing Values          15%                       Forces models to drop signal or impute noise 
 Label Noise             12%                       Model trains on incorrect ground truth — undetectable 
 Extreme Outliers         8%                       Values at 20–200x normal range skew decision boundaries 
 Duplicate Rows           5%                       Model overfits to artificial patterns 
 Sensor Drift            20%                       Gradual feature shift across rate, sload, dload columns 

Then I trained the same models on both versions and measured the gap.

---

## The Architecture: Three Models, Three Different Jobs

Early in this project I made a mistake that's worth documenting.

I treated all three models as competitors in the same task — attack classification. Isolation Forest kept underperforming with ~32% accuracy. I assumed it was a weak model.

It wasn't weak. **I was asking it to do the wrong job.**

Isolation Forest is unsupervised — it requires no labels. It identifies statistical outliers: data points that look anomalous compared to the majority. Asking it to classify attacks is like asking a quality control inspector to run the assembly line. Related roles, fundamentally different jobs.

Once I understood this, the architecture became clear:

```
STAGE 1 — Data Quality Audit (Isolation Forest)
    Unsupervised — trained ONLY on normal samples
    "Is this data point statistically suspicious?"
    Clean anomaly rate:  1.2%
    Faulty anomaly rate: 4.7%
    Integrity Index: +3.5% → confirms fault injection worked 

STAGE 2 — Attack Classification (Random Forest + XGBoost)
    Supervised — trained on labeled data
    "Is this network traffic an attack?"
    RF ROC-AUC:  0.9856
    XGB ROC-AUC: 0.9849

STAGE 3 — Robustness Comparison
    Clean vs Faulty performance gap — quantified
    XGBoost Robustness Score:       97.2/100
    Random Forest Robustness Score: 96.8/100
```

This separation is what made the results meaningful. Each model answers a different question. Each model is evaluated on its own terms.

---

## Feature Selection: Cutting 42 Features to 18

Before training, I ran a hybrid feature selection process combining three methods:

- **Manual selection** based on network traffic domain knowledge — weighted 2 votes
- **Mutual Information scoring** — 1 vote
- **Random Forest feature importance** — 1 vote

Features that appeared across multiple methods scored higher votes and made the final cut.

**Result: 42 features → 18 features** with no F1 loss — slight cross-validation improvement.

One critical catch during this stage: I had engineered a feature called `byte_ratio` (sbytes / dbytes) that was highly correlated with the target. This is **data leakage** — the model would learn from it during training but the feature wouldn't exist reliably in production. It was dropped *before* feature selection, not after. The ordering matters.

---

## The Results: Clean vs Faulty

After SMOTE balancing, preprocessing, and threshold tuning on the validation set — here's what the models achieved on **175,341 held-out test samples:**

**Clean Data Performance:**

  Model         | Scenario     | Threshold      | Precision     | Recall     | F1 |

  XGBoost        Security          0.10            96.76%        91.57%       94.09% 
  XGBoost        Balanced          0.524           98.91%        85.04%       91.45% 
  XGBoost        Operations        0.90            99.60%        77.86%       87.40% 
  Random Forest  Security          0.10            95.50%        94.23%       94.86% 
  Random Forest  Balanced          0.50            98.96%        85.14%       91.54% 
  Random Forest  Operations        0.90            99.87%        75.12%       85.75% 

**Clean vs Faulty Degradation:**

| Model | Clean F1 | Faulty F1 | Drop |
|---|---|---|---|
| Random Forest | 94.86% | ~91.8% | ~3% |
| XGBoost | 94.09% | ~92.5% | ~3% |

A 3% drop sounds small. In a system processing millions of network packets, it translates to thousands of missed detections — with no warning that performance has degraded.

**That's the core finding: bad data doesn't cause loud failures. It causes quiet ones.**

---

## Threshold Tuning: The Most Underrated Lever

Default threshold of 0.5 is almost never optimal on imbalanced data.

I tuned the decision threshold on the validation set across 50 values and found three distinct operating points depending on the deployment context:

  Scenario       | Threshold     | Trade-off 

  Security          0.10           Maximum recall — catch everything, tolerate false alarms 
  Balanced          0.52           Optimal F1 — balanced precision and recall 
  Operations        0.90           Maximum precision — fewer false positives, accept missed detections 

Same model. Same weights. Three completely different operational profiles.

Choosing a threshold isn't a technical decision — **it's a business decision.** It depends entirely on whether a missed attack or a false alarm is more costly in your specific context.

---

## The Framework: QualiEval

After validating these findings experimentally, I built QualiEval — a 6-tab Gradio app that applies this pipeline to any uploaded dataset.

**Tab 1 — Upload & Inspect**
Detects missing values, class imbalance, data leakage, and outliers the moment a CSV is uploaded. Severity-rated results, not just counts.

**Tab 2 — Model Risk Assessment**
Scores Logistic Regression, Random Forest, and XGBoost (0–100) based on the specific issues found in your data — not generic recommendations.

**Tab 3 — Model Explanation**
Explains why each model succeeds or fails given your dataset's actual issues. Tailored to what the inspection found.

**Tab 4 — Stress Tests**
Estimates performance drops across 4 fault scenarios — calculated dynamically from your dataset profile, not hardcoded.

**Tab 5 — Actionable Fixes**
Prioritized fix plan (CRITICAL → HIGH → MEDIUM → STANDARD) with code snippets for each issue.

**Tab 6 — Final Recommendation**
Robustness scores and threshold sensitivity analysis across all three deployment scenarios.

---

## What This Means Beyond the Numbers

Three principles that came out of this project that I think apply broadly:

**1. Data quality is a measurable variable, not a vague concept.**
The 3% degradation under fault injection isn't an estimate — it's a measured result from controlled experiments on the same models, same test set, same evaluation criteria. Data quality can be quantified.

**2. Model choice matters less than data preparation.**
XGBoost and Random Forest both degraded by approximately the same amount under faulty data. The architecture wasn't the differentiator — the data pipeline was.

**3. Every tool has a specific job.**
Isolation Forest as a classifier: 32% accuracy. Isolation Forest as a data quality auditor: meaningful anomaly rate signals that validated the entire experiment. Right tool, right role.

---

**Requirements:**

gradio · pandas · numpy · plotly · scikit-learn · xgboost · joblib

 **Live Demo:** [huggingface.co/spaces/sarikasingh02z/QualiEval](https://huggingface.co/spaces/sarikasingh02z/QualiEval)
 *Free tier — if sleeping, click Restart and wait 2–3 minutes*

 **GitHub:** [https://github.com/sarikasingh02z/QualiEval](github.com/sarikasingh02z/QualiEval)

---

## What's Next

QualiEval currently works on static uploaded datasets. The natural next steps would be:

- Real-time data stream monitoring
- Automated threshold recommendation based on deployment context
- Integration with preprocessing pipelines before model training
- Expanded fault injection scenarios for regression and NLP datasets

These are open problems worth exploring.

---

## Final Thought

We spend a lot of time optimizing models. Loss functions, architectures, hyperparameters — the model gets most of the attention.

But in real deployments, the data pipeline is where things go wrong. Not dramatically. Quietly.

QualiEval is an attempt to make that quiet degradation visible — before it becomes a production problem.

---

*This is a student project. The findings are from controlled experiments on UNSW-NB15 — results will vary on different datasets and deployment contexts. Feedback, critique, and collaboration are welcome.*

*If you're working on ML reliability or data quality, I'd love to connect.*

**Tags:** #MachineLearning #DataScience #DataQuality #Python #OpenSource #StudentProject #NetworkSecurity #XGBoost #Gradio
