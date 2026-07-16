# Network Intrusion Detection using Machine Learning (NSL-KDD)

This project implements a network intrusion detection system using the NSL-KDD dataset. It was developed as a reproduction and evaluation of the methodology described in the paper "A Subset Feature Elimination Mechanism for Intrusion Detection System" (Nkiama, Said & Saidu, 2016).

## Project Overview

Network intrusion detection involves classifying network traffic as either normal or an attack. This project trains and evaluates three machine learning models — Decision Tree, Random Forest, and Logistic Regression — on the NSL-KDD dataset, following the feature selection approach proposed in the reference paper, and critically evaluates how well the paper's reported results could be reproduced.

Each model is implemented inside a scikit-learn `Pipeline`, so that all preprocessing (one-hot encoding of nominal features and standard scaling of numeric features) is fitted independently within every cross-validation fold, preventing data leakage. Models are assessed with stratified 5-fold cross-validation, which preserves the normal/attack ratio in each fold — important given how imbalanced the dataset is.

Beyond the standard train-and-evaluate loop, the project adds two further analyses:

- **Feature comparison** — the full feature set is cross-validated against a reduced feature set (with highly correlated columns removed), so the feature elimination decision is backed by measured performance rather than the correlation heatmap alone. Training time is compared alongside accuracy.
- **Threshold analysis** — the Random Forest's decision threshold is swept across 19 values (0.05 to 0.95), measuring precision, recall, F1-score, false positives, and false negatives at each, to expose the trade-off between missed attacks and false alarms.

## Reference Paper

Nkiama, H., Said, S. M., & Saidu, M. (2016). *A Subset Feature Elimination Mechanism for Intrusion Detection System*. International Journal of Advanced Computer Science and Applications (IJACSA).
Paper link: https://thesai.org/Publications/ViewPaper?Volume=7&Issue=4&Code=ijacsa&SerialNo=45

## Dataset

**NSL-KDD Dataset** — an improved version of the KDD Cup 1999 dataset that removes duplicate records and provides a more balanced set of attack categories (DoS, Probe, R2L, U2R).

Dataset link: https://www.unb.ca/cic/datasets/nsl.html

## Reference GitHub Repository

This project also references the implementation approach used in:
Cynthia Koopman's Network-Intrusion-Detection repository: https://github.com/CynthiaKoopman/Network-Intrusion-Detection

## Repository Structure

```
Network-Intrusion-Detection-ML/
│
├── data/
│                                    # NSL-KDD dataset files (KDDTrain.csv, KDDTest.csv)
│
├── notebooks/
│   └── NSL_KDD_Intrusion_Detection.ipynb     # Main analysis and modeling notebook
│
├── project_outputs/  # Created at runtime by the notebook — copies of every exported file below
│
├── requirements.txt
└── README.md
```

The notebook writes its exported files to its working directory and then copies them into `project_outputs/`, so these paths are relative to wherever the notebook is executed from.

## How to Run

**1. Clone the repository**
 
```bash
git clone https://github.com/nadineshehabuni/Network-Intrusion-Detection-NSL-KDD.git
cd Network-Intrusion-Detection-NSL-KDD
```

**2. Install dependencies**

```bash
pip install -r requirements.txt
```

**3. Add the dataset**

Place `KDDTrain.csv` and `KDDTest.csv` inside the `data/` folder.

**4. Launch Jupyter Notebook**

```bash
jupyter notebook
```

Open `notebooks/NSL_KDD_Intrusion_Detection.ipynb` and run all cells in order. The notebook loads the dataset from `data/KDDTrain.csv` and `data/KDDTest.csv` relative to its working directory, so start Jupyter from the repository root.

## Required Libraries

| Library | Purpose |
|---|---|
| pandas | Data manipulation and analysis |
| numpy | Numerical computations |
| matplotlib | Data visualization |
| seaborn | Statistical data visualization |
| scikit-learn | Machine learning models and evaluation |
| joblib | Saving the trained pipelines |
| jupyter | Notebook environment |

```bash
pip install pandas numpy matplotlib seaborn scikit-learn joblib jupyter
```

## Methodology

**1. Data Loading** — the NSL-KDD training and test files are loaded from `data/`, with the 42 column names assigned from the dataset documentation.

**2. Data Preprocessing** — features are checked for data types, missing values, constant columns, and duplicate rows. The constant column `num_outbound_cmds` is dropped from both the training and test sets.

**3. Exploratory Data Analysis** — the class balance (normal vs attack) and attack category distribution (DoS, Probe, R2L, U2R) are visualised, numeric features are summarised, outliers are checked with log-scale boxplots, and feature relationships are examined with a Pearson correlation heatmap and categorical crosstabs.

**4. Feature Engineering** — the target is encoded as a binary label (0 = normal, 1 = attack). The nominal columns `protocol_type`, `service`, and `flag` are deliberately left as raw strings at this stage; they are encoded inside the pipeline instead (see step 7).

**5. Feature Selection** — the redundancy check identifies pairs of numeric features correlated above 0.9. Six of them (`srv_serror_rate`, `dst_host_srv_serror_rate`, `dst_host_serror_rate`, `srv_rerror_rate`, `dst_host_srv_rerror_rate`, `num_root`) are dropped to form a **reduced feature set**, while a **full feature set** is retained for comparison.

**6. Model Training** — three classifiers are trained: Decision Tree, Random Forest, and Logistic Regression.

**7. scikit-learn Pipeline** — every model is wrapped in a `Pipeline` whose first step is a `ColumnTransformer` applying `OneHotEncoder(handle_unknown="ignore")` to the nominal columns and `StandardScaler` to the numeric columns. One-hot encoding is used rather than label encoding because these variables are nominal — integer codes would impose an ordering and spacing the data does not support. `handle_unknown="ignore"` safely handles categories that appear in the test set but never in training.

**8. Stratified Cross Validation** — all models are evaluated with `StratifiedKFold` (5 folds, shuffled, `random_state=42`), so each fold preserves the dataset's normal/attack ratio.

**9. Full vs Reduced Feature Comparison** — both feature sets are cross-validated with the same three pipelines and the same fold split, comparing mean CV accuracy, standard deviation, and mean per-fold training time.

**10. Threshold Analysis** — the Random Forest's decision threshold is swept across `np.arange(0.05, 1.00, 0.05)`, recording precision, recall, F1-score, false positives, and false negatives at each value.

**11. Final Evaluation** — models are evaluated on the held-out test set using accuracy, precision, recall, F1-score, MCC, and ROC-AUC, with confusion matrices, classification reports, and an error analysis that breaks misclassifications down by attack category and checks for attack types absent from the training set.

## Models Used

**Decision Tree** — a single tree-based classifier that splits data based on feature values. It is fast and interpretable, but its predicted probabilities on this dataset are almost entirely saturated at 0.0 and 1.0, since its leaves are nearly pure.

**Random Forest** — an ensemble of decision trees whose predictions are combined by majority voting. Averaging across 100 trees produces graded probability scores, which is why it is the model used for the threshold analysis.

**Logistic Regression** — a linear baseline that models the log-odds of a connection being an attack. It is included for comparison rather than as the expected top performer, since intrusion patterns in this dataset are not expected to be linearly separable.

## Cross Validation

All preprocessing is performed **inside** the scikit-learn `Pipeline` rather than applied to the dataset beforehand. During stratified 5-fold cross-validation, the `ColumnTransformer` is therefore re-fitted on each training fold only: the one-hot vocabulary and the scaling means and standard deviations never see the held-out fold. This prevents data leakage, which would otherwise inflate the cross-validation scores and make them an unreliable estimate of real generalisation. The same applies to the final models, whose preprocessing is fitted on the training set alone.

## Results

**Test set performance (default 0.50 threshold, reduced feature set)**

| Model | Accuracy | Precision | Recall | F1 Score | MCC | ROC-AUC |
|---|---|---|---|---|---|---|
| Decision Tree | 82.51% | 96.86% | 71.59% | 82.33% | 0.6873 | 0.8427 |
| Random Forest | 77.75% | 96.84% | 62.96% | 76.31% | 0.6178 | 0.9598 |
| Logistic Regression | 75.54% | 91.73% | 62.68% | 74.47% | 0.5608 | 0.8081 |

**Full vs reduced feature set (mean 5-fold CV accuracy on the training data)**

| Model | Feature Set | CV Accuracy | Std Dev |
|---|---|---|---|
| Decision Tree | Full (all features) | 99.850% | 0.022% |
| Decision Tree | Reduced (redundant dropped) | 99.850% | 0.017% |
| Random Forest | Full (all features) | 99.889% | 0.020% |
| Random Forest | Reduced (redundant dropped) | 99.882% | 0.016% |
| Logistic Regression | Full (all features) | 97.289% | 0.158% |
| Logistic Regression | Reduced (redundant dropped) | 97.126% | 0.141% |

Mean per-fold training time is also recorded in `feature_set_comparison.csv`. The reduced feature set trains consistently faster for all three models, but the saving is a fraction of a second per fold and therefore negligible at this dataset size. Absolute timings depend on the machine running the notebook.

**Threshold analysis (Random Forest, selected thresholds from the 19 swept)**

| Threshold | Precision | Recall | F1 Score | False Positives | False Negatives |
|---|---|---|---|---|---|
| 0.05 | 91.91% | 94.07% | 92.98% | 1063 | 761 |
| 0.15 | 95.33% | 81.34% | 87.79% | 511 | 2394 |
| 0.50 (default) | 96.84% | 63.24% | 76.52% | 265 | 4717 |
| 0.95 | 97.41% | 50.35% | 66.39% | 172 | 6371 |

The complete sweep across all 19 thresholds is exported to `threshold_analysis.csv`.

## Key Findings

- **Pipeline-based preprocessing prevents data leakage.** One-hot encoding and scaling are fitted inside each stratified cross-validation fold rather than across the whole dataset, so the reported CV scores are an honest estimate rather than an inflated one.
- **Precision is high, recall is the weak point.** At the default threshold all three models keep precision above 91% (Decision Tree and Random Forest above 96%), meaning normal traffic is rarely misclassified as an attack. Recall sits between 62% and 72%, so a meaningful share of actual attacks is missed.
- **Logistic Regression is the weakest model,** as expected for a linear baseline on non-linearly-separable intrusion patterns — the lowest F1 (74.47%) and MCC (0.5608) of the three.
- **The Decision Tree scores best at the default threshold** (F1 82.33% vs the Random Forest's 76.31%), despite the Random Forest achieving marginally higher accuracy during cross-validation.
- **But that ranking is an artefact of the threshold.** The Random Forest has a far better ROC-AUC (0.9598 vs 0.8427), which measures ranking quality independently of any cut-off, and at a 0.05 threshold its F1 reaches 92.98% — well above the Decision Tree's best. The Random Forest was ranking connections better all along; 0.50 was simply the wrong place to slice its probabilities.
- **Removing the redundant features costs nothing, but gains nothing either.** Cross-validated accuracy is effectively unchanged for all three models (the Decision Tree is identical, the Random Forest differs by under 0.01 percentage points, and Logistic Regression is 0.16 points *lower* on the reduced set — within its own fold-to-fold variation). The reduced set is therefore justified by parsimony and a simpler model, not by improved performance, and the runtime saving is negligible.
- **The threshold trade-off is strongly asymmetric.** Across the sweep, recall falls from 94.07% to 50.35% while precision moves only about five percentage points. Lowering the threshold from 0.50 to 0.05 adds roughly 800 false alarms but eliminates nearly 4,000 missed attacks. Since a missed intrusion normally costs far more than a false alarm, a threshold in the 0.05–0.15 band is a better operating point than the default. One caveat: F1 peaks at 0.05, the lowest value tested, so the true optimum may lie below the swept range.
- **The Random Forest is the right model for this analysis.** Only 0.02% of the Decision Tree's predicted probabilities fall strictly between 0.01 and 0.99, compared with 33.98% for the Random Forest, so a threshold sweep on a single tree would barely change any decisions.
- **Unseen attack types drive the errors.** Error analysis showed that a large share of misclassifications came from attack types present in the test set but never seen during training. This explains the gap between the near-99% cross-validation accuracy and the 75–83% test accuracy, and is discussed in detail in the project report's Critical Evaluation section.

## Output Files

Running the notebook end to end generates the following files in its working directory, copies them into `project_outputs/`, and archives that folder as `NSL_KDD_Project_Outputs.zip`.

| File | Description |
|---|---|
| `class_distribution.png` | Bar chart of normal vs attack traffic |
| `attack_category_distribution.png` | Bar chart of attack categories (DoS, Probe, R2L, U2R) |
| `outlier_check.png` | Log-scale boxplots of key numeric features |
| `correlation_heatmap.png` | Pearson correlation heatmap of numeric features |
| `confusion_matrices.png` | Confusion matrices for all three models, side by side |
| `threshold_analysis.png` | Precision/Recall/F1 vs threshold, and false positives vs false negatives |
| `model_comparison.csv` | Test-set accuracy, precision, recall, F1, MCC, and ROC-AUC per model |
| `feature_set_comparison.csv` | Full vs reduced feature set: CV accuracy, std dev, and mean fit time |
| `threshold_analysis.csv` | Precision, recall, F1, FP, and FN at all 19 swept thresholds |
| `processed_train_data.csv` | Cleaned training dataset |




## References

1. Nkiama, H., Said, S. M., & Saidu, M. (2016). A Subset Feature Elimination Mechanism for Intrusion Detection System. IJACSA.
2. NSL-KDD Dataset — Canadian Institute for Cybersecurity, University of New Brunswick. https://www.unb.ca/cic/datasets/nsl.html
3. Koopman, C. Network-Intrusion-Detection (reference implementation). https://github.com/CynthiaKoopman/Network-Intrusion-Detection
