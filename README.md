# Business Case: Flagging Late Deliveries at Checkout

> **Status:** Portfolio project (v1) using public historical data, Sept 2016 – Aug 2018. Not affiliated with Olist.

---

## 1. Executive summary

About **1 in 12 delivered orders (8.1%)** on the Olist marketplace arrived after the date the customer was promised. Today that is discovered only after the promise is broken, when the remaining options are apologies and refunds.

This project tests whether lateness can be predicted **at the moment of purchase**, using only information available at checkout. A first-version model (logistic regression) does this at roughly **twice the rate of random targeting**: on the most recent three months of orders, flagging about a quarter of orders catches **about half (48%) of all late deliveries**.

That level of accuracy supports **low-cost, high-volume actions**, such as proactive customer messages or adding a buffer to the promised date for flagged orders. It does not yet justify expensive actions, such as paying for expedited shipping, on every flagged order.

**Recommendation:** pilot a low-cost intervention on flagged orders with an A/B test, while improving the model (Section 10) before using it for costly interventions.

---

## 2. Context

Olist connects small Brazilian sellers to large online marketplaces under a single contract. At checkout, each order receives an **estimated delivery date**, which acts as a promise to the customer. Fulfilment is spread across thousands of independent sellers and carriers covering a continent-sized country, so delivery reliability varies widely by route, region and product.

---

## 3. The problem

| Signal | Value |
|---|---|
| Delivered orders analysed | 96,470 |
| Share that arrived after the promised date | **8.1%** |
| Late rate, highest-volume states | Bahia ~14%, Rio de Janeiro ~13%, Espírito Santo ~12% |
| Late rate, São Paulo (largest market) | ~6% |
| Seasonal peaks | Late rate spiked to roughly 14% in Nov 2017 and up to ~20% in Feb–Mar 2018 |

Two patterns stand out:

- **Lateness is concentrated.** It is two to three times more common for customers in some north-eastern and south-eastern states than in São Paulo, and it spikes around peak shopping periods.
- **Lateness is a problem of broken promises, not only slow delivery.** Orders that arrived late had *shorter* promised delivery windows on average. In other words, the promised date was too optimistic for the route.

Why it matters: review scores are public and affect seller rankings, and customers who receive a broken promise are less likely to buy again. *The review impact has not yet been quantified in this analysis; see Section 10.*

---

## 4. Proposed solution

A risk score attached to every order at checkout. The model uses **only information known at the moment of purchase**:

| Group | Features |
|---|---|
| Promise | Promised delivery window (days from purchase to estimated date) |
| Route | Customer state, seller state, same-state flag, buyer–seller distance (km) |
| Product | Main category, average weight and volume |
| Order | Number of items, products and sellers; total price, freight, freight-to-price ratio |
| Payment | Payment type, instalments, number of payment methods, total payment |
| Timing | Purchase day of week, month, weekend flag, holiday-season flag (Nov–Dec) |

**Deliberately excluded** to prevent leakage: approval time, carrier hand-off date, shipping limit date, actual delivery date, and all review data. These are only known after the customer has checked out.

**Who would use it**

| User | What they see | What they do |
|---|---|---|
| Customer experience | Risk flag on the order | Send an early, realistic delivery update |
| Promise / pricing team | Risk by route and category | Adjust promised dates where they are consistently too optimistic |
| Operations | Highest-risk orders | Targeted follow-up with the seller (once precision improves) |

---

## 5. Results

**Validation approach:** time-based split, mirroring real use (train on the past, predict the future).

- Train: 77,176 orders, Sept 2016 – May 2018 (late rate 8.8%)
- Test: 19,294 orders, late May – Aug 2018 (late rate 5.3%)

| Model | PR-AUC | ROC-AUC | F1 (late class) |
|---|---|---|---|
| Baseline (always "on time") | 0.053 | 0.50 | 0.00 |
| Random forest | 0.066 | 0.56 | 0.10 |
| **Logistic regression** | **0.120** | **0.73** | **0.17** |

PR-AUC is the main metric because late orders are rare. The logistic regression's PR-AUC is about **2.3x the baseline**.

**At the default decision threshold (test period):**

| | Value | Plain-English meaning |
|---|---|---|
| Recall (late orders caught) | 48% | Catches about half of all late deliveries |
| Precision (flagged orders that were late) | ~11% | Roughly 1 in 10 flagged orders is actually late, versus about 1 in 19 at random |
| Share of orders flagged | ~24% | Roughly a quarter of orders get flagged |

**What drives risk (model coefficients)**

- **Region:** sellers in Maranhão and Amazonas, and customers in Alagoas, Roraima and Sergipe, carry the largest risk increases. Sellers in Pernambuco and Mato Grosso, and customers in Rondônia, Acre and Paraná, carry lower risk.
- **Season:** orders placed in November–December are more likely to be late.
- **Promise length:** longer promised windows reduce risk, which confirms the EDA finding that short promises on difficult routes are the problem.
- **Category:** some categories (home comfort, audio, office furniture, DVDs) carry higher risk; others (computers, flowers, agro-industry) carry lower risk.

*Caution:* several of the strongest coefficients belong to low-volume states and categories, so their individual estimates are less reliable. The regional pattern is consistent with the high-volume EDA results.

---

## 6. Decision framework

With precision around 11%, most flagged orders will arrive on time anyway. The right actions are therefore ones that are **cheap and harmless when the flag is wrong**:

| Action | Cost per order | Harm if order was fine | Fit for v1 model |
|---|---|---|---|
| Proactive "your order may take longer" message | Very low | Minimal | ✅ Good fit |
| Adding a small buffer to the promised date for flagged routes | Low (some conversion risk) | Slightly longer promise | ✅ Good fit, monitor conversion |
| Seller follow-up for the highest-risk orders | Medium | Wasted staff time | ⚠️ Only for the top-scoring slice |
| Paid expedited shipping | High | Wasted freight spend | ❌ Not until precision improves |

---

## 7. Value estimate

Per **10,000 orders** in the test period, at the model's default threshold:

- About **530** orders would arrive late without intervention.
- The model flags about **2,400** orders, of which about **250** would actually be late.

**Break-even rule:** an intervention pays off only if

```
value of saving one late order  >  ~9–10 × cost of one intervention
```

because roughly 9 to 10 orders must be flagged to reach one that is truly late (precision ≈ 11%).

**What this means in practice:**

- A proactive message costing a few cents easily clears this bar.
- An expedited-shipping upgrade costing a meaningful share of the freight fee does not, unless it is limited to a much smaller, higher-precision slice of orders.

The cost and benefit figures themselves are left as assumptions for the pilot to measure. The break-even ratio above comes directly from the model's test results.

---

## 8. Alternatives considered

| Option | Upside | Downside |
|---|---|---|
| **Do nothing** | No cost | 8% of customers keep receiving broken promises |
| **Pad every promised date** | Simple; fewer broken promises | Longer promises on *all* orders, including the 92% that would have arrived on time, likely reduce conversion |
| **Simple rules** (e.g. flag certain states and holiday season) | Easy to build and explain | Not yet benchmarked; see next steps |
| **Predictive model** *(proposed)* | Targets effort where risk is concentrated; combines many factors at once | Moderate accuracy in v1; needs monitoring and retraining |

---

## 9. Pilot plan

1. **Score** all new orders for 4–6 weeks.
2. **Randomise** flagged orders: half receive a proactive delivery update (treatment), half do not (control).
3. **Measure:** review scores, support contacts and repeat purchase rate, treatment vs control.
4. **Guardrails:** intervention cost per 1,000 orders; conversion rate if promised dates are adjusted.

A randomised pilot is necessary because the model shows *which* orders are risky. Only an experiment shows whether intervening *changes* the outcome.

---

## 10. Risks, limitations and next steps

**Risks and limitations**

| Issue | Detail | Mitigation |
|---|---|---|
| **Checkout-only information caps accuracy** | The real causes of many delays (carrier issues, warehouse backlog, weather in transit) happen after purchase | Pair with a second "in-transit" model that uses shipping events to catch orders already running late |
| **Shift between training and test periods** | Late rate fell from 8.8% in training to 5.3% in testing, and training included unusual spikes | Monitor monthly; retrain regularly; report results by period |
| **Possible end-of-data bias** | Orders still in transit when the data ends are excluded as "not delivered", which may remove some of the latest late orders from the test set | Re-check the test late rate after excluding the final weeks of data |
| **Regional fairness** | Some regions are flagged far more often | Use the score to improve service in those regions, never to deprioritise customers |
| **Old data** | Dataset ends in 2018 | Treat as a proof of concept; validate on current data before any deployment |

**Next steps to strengthen the case**

- [ ] Fix the sparse-input error so gradient boosting (HistGradientBoosting / XGBoost) can be compared
- [ ] Report **recall at top 5% and 10%** of orders, which shows whether a high-precision slice exists for costlier actions
- [ ] Benchmark a **simple rules baseline** (state + holiday season) against the model
- [ ] Quantify the **business impact**: average review score and 1-star share for late vs on-time orders
- [ ] Calibrate probabilities and choose the threshold by cost rather than the default
- [ ] Add SHAP explanations once a tree-based model is working

---

*Data: [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce), CC BY-NC-SA 4.0.*
