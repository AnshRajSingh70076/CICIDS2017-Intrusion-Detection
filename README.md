# CICIDS2017 Multiclass Intrusion Detection

A machine learning project for multiclass network intrusion detection using the **CICIDS2017** dataset.

The project focuses not only on training classifiers, but also on **data quality, sampling, leakage prevention, class imbalance, label taxonomy, and model evaluation**.

> **Key finding:** The biggest improvement did not come from SMOTE or a more complex model. It came from correcting an inconsistent label definition that caused persistent confusion between `WEBATTACK` and `BRUTEFORCE`.

## Results

After correcting the label taxonomy, the final Random Forest model achieved:

| Metric       |     Result |
| ------------ | ---------: |
| Accuracy     | **99.72%** |
| Macro F1     | **0.9902** |
| WEBATTACK F1 | **0.9883** |
| BOT F1       | **0.9609** |

The final evaluation used 36,126 test samples.

Additional models were evaluated using the same corrected labels and train/test split:

| Model         | Accuracy | Macro F1 |
| ------------- | -------: | -------: |
| Decision Tree |   99.67% |   0.9872 |
| Random Forest |   99.72% |   0.9902 |
| LightGBM      |   99.81% |   0.9921 |
| XGBoost       |   99.83% |   0.9933 |

These results come from a single 80/20 train/test split, so they should not be interpreted as proof that one model is universally superior.

## Problem

The initial Random Forest achieved high overall accuracy, but `WEBATTACK` had a very low F1-score of **0.40**.

Most WEBATTACK errors were being classified as BRUTEFORCE.

Several approaches were tested:

* Class weighting
* SMOTE
* SMOTETomek

These methods improved recall in some cases but did not solve the underlying confusion.

## The Actual Root Cause

The investigation showed that the problem was related to the way the raw CICIDS2017 labels were grouped.

The raw dataset contains:

* `Web Attack - Brute Force`
* `FTP-Patator`
* `SSH-Patator`

The original grouping placed all three under `BRUTEFORCE`.

However, at the network-flow level, `Web Attack - Brute Force` behaved more like other HTTP-based attacks such as XSS and SQL Injection than FTP/SSH brute-force traffic.

### Corrected taxonomy

```text
BRUTEFORCE
├── FTP-Patator
└── SSH-Patator

WEBATTACK
├── Web Attack - Brute Force
├── Web Attack - XSS
└── Web Attack - Sql Injection
```

After this correction, the WEBATTACK/BRUTEFORCE confusion almost disappeared without SMOTE or SMOTETomek.

## Data Processing

### 1. Multi-file sampling

The CICIDS2017 data was distributed across multiple CSV files.

The initial sampling approach could cause individual classes to be dominated by particular files.

A two-pass sampling strategy was introduced:

1. Scan all files and count raw labels.
2. Allocate class quotas across available files.
3. Redistribute shortages when a file does not contain enough samples.

This provided more balanced representation across the dataset files.

### 2. Duplicate removal

The project identified **17,354 exact duplicate rows**, approximately 9% of the sampled data.

These duplicates were removed before the train/test split to reduce the possibility of identical records appearing in both sets.

### 3. Leakage prevention

Missing-value imputation was performed only after splitting the data.

The median was calculated from the training set and then applied to both training and test data.

The same principle was followed for resampling:

```text
Train/Test Split
      ↓
Training Data
      ↓
Imputation / SMOTE
      ↓
Model Training
```

The test set remains untouched during preprocessing.

## Models

The project evaluates several tree-based classifiers:

* Decision Tree
* Random Forest
* LightGBM
* XGBoost

Random Forest was configured with:

```python
RandomForestClassifier(
    n_estimators=200,
    class_weight="balanced"
)
```

No SMOTE or SMOTETomek was used for the final Random Forest results.

## Final Classes

The final model evaluates:

```text
BENIGN
BOT
BRUTEFORCE
DOS
PORTSCAN
WEBATTACK
```

The `INFILTRATION` class was removed because the complete dataset contained only 36 samples, making reliable evaluation difficult.

## Important Findings

### 1. More sophisticated resampling was not the solution

SMOTE and SMOTETomek changed the precision/recall trade-off but did not remove the underlying confusion.

### 2. Data definitions matter

A model can appear to have a difficult classification problem when the real issue is inconsistent class construction.

### 3. Model complexity was not the main factor

After fixing the labels, even a Decision Tree achieved a macro-F1 of 0.9872.

The boosted models provided only relatively small improvements over Random Forest on the same split.

## Limitations

This project has several important limitations:

* Evaluation uses a single 80/20 train/test split.
* No k-fold cross-validation was performed.
* No systematic hyperparameter search was performed.
* BOT and WEBATTACK remain smaller classes.
* Destination Port remains an important feature and may capture dataset-specific network characteristics.
* CICIDS2017 represents a specific lab environment and one week's traffic.
* Performance on live or differently configured networks is not guaranteed.

## Future Work

Potential improvements include:

1. Perform k-fold cross-validation.
2. Run systematic hyperparameter tuning.
3. Evaluate using a completely separate day's traffic as a held-out test set.
4. Re-evaluate SMOTE and SMOTETomek after the corrected taxonomy.
5. Investigate feature importance and potential dependence on network-specific ports.
6. Evaluate the model on a different intrusion-detection dataset.

## Tech Stack

* Python
* Pandas
* NumPy
* Scikit-learn
* Random Forest
* Decision Tree
* LightGBM
* XGBoost
* SMOTE
* SMOTETomek
* Matplotlib
* Seaborn

## Dataset

This project uses the **CICIDS2017** network intrusion detection dataset.

The dataset contains benign traffic and multiple attack categories collected in a controlled network environment.

## Project Structure

```text
CICIDS2017-Intrusion-Detection/
│
├── CICIDS2017_multiclass_all_days.ipynb
├── README.md
├── requirements.txt
└── results/
    ├── confusion_matrix.png
    └── model_comparison.png
```

## Key Takeaway

The most important lesson from this project was simple:

> **When a model repeatedly confuses two classes, don't immediately assume that you need a more powerful model or more synthetic data. First verify that the class definitions themselves make sense.**

In this project, correcting the label taxonomy had a much larger impact than the resampling techniques that were initially investigated.
