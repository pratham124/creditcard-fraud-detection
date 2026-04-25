# Analytical Reflection: Credit Card Fraud Anomaly Detection
ECE 447 - Winter 2026 \
Matthew Naruzny, Pratham Sitoula

---

## Trade-offs in Threshold Selection

For a fraud detection system, threshold selection is one of the most consequential design decisions. It directly controls the precision-recall trade-off, and the two error types carry very different costs.

A missed fraud event (false negative) may expose the bank to the full transaction loss. On this dataset, the mean fraudulent amount is $122.21. 
A false alarm (false positive) triggers a manual review, a blocked card, and potential customer friction, but the bank does not lose the transaction value.
All three methods were evaluated at the F1-maximizing threshold.
Full sensitivity curves are in `notebooks/comparison.ipynb`.

For the Mahalanobis Distance detector (`notebooks/mahalanobis_distance.ipynb`), the F1-maximizing threshold was 24.01, yielding precision 0.718, recall 0.770, and F1 0.743. At this operating point the model flagged 149 legitimate transactions while catching 379 of 492 fraud cases. Sensitivity analysis showed that lowering the threshold increases recall sharply but degrades precision quickly: a threshold near 20 pushes recall above 0.85 at the cost of roughly doubling false positives.

For the kNN detector (`notebooks/knn.ipynb`), the best configuration used k=2 nearest neighbors, selected by sweeping k over {2, 5, 10, 20, 50} and choosing the value with the highest PR-AUC. The F1-maximizing threshold was 6.03, yielding precision 0.655, recall 0.652, and F1 0.654. A PR-AUC of 0.616 well below the ROC-AUC of 0.964 indicates data imbalance; the ROC is inflated by the enormous number of true negatives, while PR-AUC correctly reflects performance on the minority class. The confusion matrix shows lower recall than Mahalanobis at the F1-optimal point, suggesting the kNN score distribution overlaps more between classes.

For the Isolation Forest detector (`notebooks/isolation_forest.ipynb`), n_estimators was swept over {50, 100, 200, 300} and n=100 yielded the best PR-AUC. At the F1-maximizing threshold of 0.085 the model achieved precision 0.372, recall 0.533, and F1 0.438. The low PR-AUC of 0.368 and high false positive rate of 0.78% shows that Isolation Forest struggles with this dataset. The PCA-projected feature space does not give random partitioning trees enough leverage to separate fraud from legitimate transactions, and the method flags roughly three times as many false alarms as Mahalanobis.

In production, threshold selection must be driven by a business cost function rather than F1 alone. The optimal operating point shifts toward lower thresholds and higher recall even when precision suffers. The full three-way cost comparison is in `notebooks/comparison.ipynb`.

---

## Impact of Class Imbalance
The dataset used contains 284,807 transactions. Of those transactions, only 492 (0.17%) are fraudulent. This extreme data imbalance affected model design decisions, training, and evaluation.

In training, the models were trained exclusively on legitimate transactions (semi-supervised). The fraudulent transactions were filtered from the datasets. This is done to avoid issues of overfitting or underfitting the fraudulent transactions. 

During evaluation, PR curves and PR-AUC provide the best insight. Standard accuracy is meaningless in these models, as labeling all transactions as legitimate would achieve a very high accuracy. The test dataset contains 492 fraudulent transactions, out of 57,355 total (0.86%). The fraud rate in the overall dataset is 0.17%. For kNN, the gap between the ROC-AUC (0.964) and PR-AUC (0.616) illustrates how ROC-AUC is misleading on this dataset.
(visible in the side-by-side curve plots in `notebooks/comparison.ipynb`)

Two further implications of the imbalance affect how these results should be interpreted. With only 492 fraud samples in the test split, small changes in TP or FP counts produce large swings in precision and recall. Cross-validation over multiple stratified splits would be needed to reliably estimate threshold stability.

The score histograms in `notebooks/comparison.ipynb` show a consistent pattern across all three methods: fraud cases produce a long right tail but overlap substantially with legitimate transactions in the mid-range. Isolation Forest shows the most overlap, explaining its low PR-AUC. The Mahalanobis histogram shows the cleanest separation, with fraud scores concentrated well above the threshold. This overlap is not noise: it reflects genuine fraud that structurally mimics legitimate behavior, a problem no single-threshold detector can fully resolve.

---

## Failure Scenarios in Production

Fraud patterns evolve continuously as attackers adapt to detection. A model trained on 2013 transaction patterns (this dataset's vintage) may fail silently on novel attack vectors. Neither Mahalanobis Distance nor kNN would automatically signal when their score distributions diverge from the baseline; they would simply flag fewer anomalies or produce more false alarms.

Sophisticated fraud rings can probe a deployed system by staging low-value transactions and adjusting behavior to stay below the detection threshold. Distance-based methods like kNN are particularly vulnerable: once an attacker knows the approximate decision boundary, they can craft transactions that sit just inside it. The PCA visualization in `notebooks/knn.ipynb` shows that fraud concentrates at the periphery of the legitimate cluster; an attacker who learns this structure can target the boundary region directly. Isolation Forest is similarly exposed because its decision surface is a product of random axis-aligned cuts that can be mapped with enough probing.

Seasonal spending patterns, new merchant categories, and customer lifecycle changes (e.g., a customer starting a business) can cause legitimate transactions to appear anomalous. This increases false positives and erodes customer trust without any change in actual fraud rates.

A threshold tuned on historical data may become miscalibrated as the transaction volume grows or the fraud rate changes. A fixed threshold that was once optimal could allow more fraud through as the legitimate distribution spreads, or generate alert fatigue if the legitimate population contracts around the trained centroid.

---

## Monitoring and Retraining Strategy

Log the anomaly score distribution for every prediction batch. Track the 95th-percentile score for legitimate transactions and alert when it shifts by more than one standard deviation from a rolling baseline; this is an early signal of distribution drift without requiring ground-truth labels.

Chargebacks and confirmed fraud reports arrive days to weeks after the transaction. A pipeline should ingest these labels and evaluate live model performance on a rolling 30-day window. If recall drops below a defined floor, trigger retraining.

Retrain the normal-behavior model monthly on recent legitimate transactions. Because the semi-supervised approach trains only on genuine data, retraining does not require labeled fraud examples; it only requires a reliable stream of confirmed-legitimate transactions.

After each retraining, rerun the PR-curve analysis on a held-out validation set and recompute the F1-optimal or cost-optimal threshold. Automate this step so threshold updates are always tied to model updates, not manual intervention.

Before promoting a retrained model, run it in shadow mode alongside the production model for 48 to 72 hours. Compare anomaly score distributions and flag rates. Only promote if the shadow model's behavior is consistent with expectations and does not produce a significant increase in false positive rate.

For transactions scored near the decision boundary, route to a human review queue rather than making a hard block/allow decision. This captures borderline cases and generates labeled training data for future supervised or semi-supervised refinements.
