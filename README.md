## How the model works



Features are built from Postgres data (`staging.stg\_bets`, `staging.stg\_customers`), 

using only activity \*\*before a fixed cutoff date\*\*, to avoid leaking future information 

into the model. Customers with no activity before the cutoff are excluded, since churn 

isn't a meaningful concept for someone who wasn't an active customer in the first place.



Churn was originally defined as "no activity in the 60 days after the cutoff", 

but this produced only 0.4% positive cases — far too few to train on. We checked

the actual distribution of "days until next bet" and found that a 14-day window 

gives a much more usable 18% churn rate, with a clear business interpretation: 

a customer becomes at-risk if they go quiet for two weeks.



We also checked whether `segment\_client` was distorting this definition — for example, 

whether "Seasonal" customers looked like churn just because they bet infrequently. 

The data showed the opposite: Seasonal and High Value customers return almost immediately 

(median of 1 day), while Recreational customers take much longer (median of 9 days). 

Since segment is clearly predictive, it was kept as a model feature rather than excluded.



Two models were compared: Logistic Regression and Random Forest, both trained with `class\_weight="balanced"` 

to prevent the model from simply predicting "no churn" for everyone, given the class imbalance. Random Forest 

performed slightly better (ROC-AUC 0.766 vs. 0.754) and was selected as the final model, after dropping three 

highly correlated features (`total\_bets`, `total\_bet\_amount`, `active\_days`, correlation 0.90–0.98) that were 

adding redundancy rather than signal.



The final decision threshold (0.265, instead of the default 0.5) is a separate choice from handling class 

imbalance — it reflects a business tradeoff, not a technical fix. In this context, a false positive 

(contacting a customer who wasn't going to churn) is cheap — a retention email or promotion. A false negative 

(missing a customer who was about to churn) is expensive — losing them entirely. Given that asymmetry, 

the threshold was chosen to prioritize recall: at 0.265, the model catches 95% of actual churn cases, 

at the cost of a lower precision (32%).

A cost-based analysis was explored, estimating false-negative cost from each customer's historical betting value and assuming a conservative false-positive cost ($40 MXN per retention contact). Under this framework, the optimal threshold trends toward very low values, since the estimated cost of losing a customer vastly exceeds the cost of a retention contact. However, this result is highly sensitive to the false-negative cost assumption (90 days of projected future activity), which likely overestimates real recoverable value. The threshold of 0.265 was ultimately chosen as a more conservative, recall-focused compromise, rather than the mathematically "optimal" value from this cost model.