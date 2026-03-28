# Dynamic Plan Recommendation Engine

An interactive, browser-based tool that recommends the most suitable SaaS plans for customers using a weighted scoring algorithm — built from a product specification I designed as a PM.

**Live demo:** [your-username.github.io/dynamic-plan-recommender](https://anubhutipandey1.github.io/dynamic-plan-recommender)

---

## The Problem

Most SaaS businesses struggle to match the right plan to the right customer especially at scale. Customers are left to navigate feature tables on their own. This leads to under-selling, over-selling, and high churn when customers land on the wrong plan.

This project explores a data-driven solution: a recommendation engine that scores and ranks plans for each customer based on their profile and usage patterns.

---

## How It Works

The engine follows a 3-step pipeline defined in the product spec:

### 1. Input
Two data sources are analysed:

**Customer data**
| Field | Description |
|---|---|
| Customer ID | Unique identifier |
| Lifetime Revenue | Total revenue generated from the customer |
| Renewal Count | How many times the customer has renewed |
| Current Plan ID | The plan the customer is currently subscribed to (optional — enables Category Match scoring) |
| Average Revenue | Computed as Lifetime Revenue ÷ Renewal Count |

**Plan data**
| Field | Description |
|---|---|
| Plan ID | Unique identifier |
| Category | Budget / Standard / Premium |
| Plan Price | Monthly price of the plan |
| Billing Frequency | Monthly / Quarterly / Annual — used to weight the renewal rate score |
| Active Subscriptions | Number of users currently on this plan (popularity proxy) |
| Total Subscriptions | Total users ever on this plan |
| Total Revenue | Cumulative revenue from this plan |

Current Plan ID lives directly in the customer record — no separate mapping table needed. The engine looks up the plan's category from this field to compute Category Match scoring as defined in the spec.

---

### 2. Recommendation Logic

For each customer–plan pair, the engine computes a score using four weighted parameters:

```
Score = W1 × Popularity + W2 × Renewal Rate + W3 × Affordability + W4 × Category Match
```

| Parameter | Definition | Weight |
|---|---|---|
| Popularity | Normalised active subscriptions of the plan | W1 |
| Renewal Rate | (Active Subscriptions ÷ Total Subscriptions) × Frequency Multiplier (Monthly=1, Quarterly=2, Annual=4) | W2 |
| Affordability | Inverse of (Plan Price ÷ Customer Avg Revenue) | W3 |
| Category Match | 1 = same category as current plan, 0.5 = adjacent category, 0 = no match | W4 |

All parameters are normalised to a 0–1 scale before scoring to ensure fair comparison across different units. Weights are configurable via sliders (0–10), so the logic can be tuned for different business contexts.

---

### 3. Output

Plans are ranked in descending order of score. The user selects Top 2, Top 3, or Top 5. Each result shows:
- Overall score with a visual bar relative to the top-ranked plan
- A plain-English explanation of *why* this plan was recommended — written in terms a sales rep can read and use, not just raw numbers
- Per-parameter score breakdown (Popularity, Renewal, Affordability, Category Match)
- Plan metadata (price, renewal rate, affordability ratio)

---

## Key Product Decisions

| Decision | Reasoning |
|---|---|
| Editable input tables | Makes the tool testable with real or hypothetical data without any backend |
| Current Plan ID in customer record | Rather than a separate mapping table, Current Plan ID is a column in the customer table itself — simpler to fill in, easier to read, and keeps all customer context in one place |
| Plain-English "why" explanation per result | Numbers alone don't help a sales rep. The why-text translates the score into a sentence a human can act on — e.g. "This plan has a strong renewal rate and sits in the same category as the customer's current plan" |
| Realistic seed data | Pre-loaded with B2B SaaS-representative values: renewal rates of 75–90%, plan prices calibrated to customer average revenues, and customer–plan mappings that make business sense. This lets anyone validate the output logic immediately |
| CSV upload for customer data | Real-world sales teams maintain customer lists in spreadsheets — uploading a CSV is far faster than entering rows manually. Auto-detects headers, validates each row, and shows specific error messages on bad data |
| Currency selector ($ / ₹) | SaaS businesses operate across markets — a single toggle switches all price displays between USD and INR without re-entering any data |
| Configurable weights via sliders | Different businesses prioritise differently — a budget-focused team would increase W3, an enterprise sales team might increase W4 |
| Normalised scoring | renewal rate of plan with annual billing frequency vs a plan with monthly billing frequebcy can't be put on same scale, hnece the addition of Frequency multiplier |
| Customer ID text input instead of dropdown | A dropdown grows unwieldy as the customer list scales. A text input is faster for users who already know the ID, and shows a specific inline error if the ID isn't found |
| Top N as toggle buttons (Top 2 / Top 3 / Top 5) | A sales rep needs the top 2–3 options to present, not a long ranked table. Fixed buttons are faster to use than a dropdown for a small set of choices |

---

## How to Use

1. Open `dynamic-plan-recommender.html` in any browser — no install or server needed
2. **Load customer data** — either click "↑ Upload CSV" to import a file, or add rows manually
3. For each customer, select their **Current Plan ID** from the dropdown in the customer table — this enables accurate Category Match scoring
4. Edit the **Plan Data** table with your plan details
5. Select your **currency** ($ USD or ₹ INR) from the top-right dropdown — all price displays update instantly
6. Adjust the **W1–W4 weight sliders** to reflect your business priorities
7. **Enter a Customer ID** (e.g. `C001`), choose **Top 2**, **Top 3**, or **Top 5**, and click **Generate recommendations**
8. Read the ranked results — each plan includes a plain-English explanation of why it was recommended

### CSV format for customer upload

```
customer_id, lifetime_revenue, renewal_count, current_plan_id
C001, 5880, 4, P003
C002, 1188, 2, P001
C003, 47880, 10, P005
```

The `current_plan_id` column is optional — rows without it will have Category Match scoring disabled. The header row is also optional; the parser auto-detects it. If any row contains invalid data, a specific error message identifies the exact row and field.

---

## Roadmap: From Rule-Based to ML

This prototype uses a manually weighted scoring model. The weights reflect judgment — not data. Here is the logical progression to a production ML recommendation engine.

### Stage 1 — Better data, same model
*The most important step before any ML.*

The current affordability metric uses historical revenue as a proxy for spending capacity, which is imperfect. Before introducing ML, the inputs need to be improved:
- Add a **budget / willingness-to-pay** field to customer data
- Track **actual conversion outcomes** — did the customer accept the recommendation, upgrade, or churn?
- Add **plan usage intensity** — heavy users are upsell candidates; light users are churn risks

Without labelled outcome data, no ML model can learn anything meaningful.

### Stage 2 — Learn the weights from data
*The smallest meaningful ML upgrade.*

Instead of manually setting W1–W4, a **Logistic Regression** model learns the optimal weights from historical conversion data. For each past recommendation, the model sees: what were the feature values, and did the customer convert (yes/no)? It finds the weight combination that best predicts conversion.

Why logistic regression first: it is explainable (the learned coefficients are directly interpretable), it needs relatively little data (a few hundred labelled examples), and it directly answers the business question — *what combination of factors predicts whether a customer subscribes?*

In a production version of this tool, the W1–W4 sliders would be replaced by model-derived coefficients, retrained periodically as new conversion data comes in.

### Stage 3 — Collaborative filtering
*What Netflix, Spotify, and Amazon use at their core.*

Instead of scoring plans by fixed rules, the model finds customers with similar profiles and recommends what *they* chose:

- **User-based:** "Customers like C001 (mid-revenue, Standard plan, 4 renewals) also upgraded to P004 — recommend P004."
- **Item-based:** "Customers on P003 (Standard, monthly) tend to move to P005 (Premium, annual) — recommend P005 to anyone currently on P003."

This requires a full customer–plan interaction history (not just the current plan), implemented using matrix factorisation. Tools: Python `surprise` or `implicit` libraries.

### Stage 4 — A full ML pipeline
*Only worth building once Stages 2 and 3 are validated.*

- **Feature engineering:** add tenure, usage intensity, support ticket volume, login recency, price-to-market ratio
- **Model:** Gradient Boosted Trees (XGBoost / LightGBM) — handles non-linear relationships, standard for tabular recommendation problems in industry
- **Serving:** a Python API (FastAPI) that replaces the JavaScript scoring logic, called by the front end
- **Feedback loop:** the tool logs which recommendations were accepted; the model retrains on new data automatically

---

## Known Limitations

This is a portfolio prototype, not a production system.

**Affordability uses revenue-from-customer, not customer budget.** Average revenue is computed from what the customer has historically paid — not their actual spending capacity. A long-tenure customer who spent $48,000 over 10 renewals looks "high-value", but that doesn't mean they can afford a more expensive plan. In a real system, budget data or explicit willingness-to-pay signals would be needed.

**No data persistence.** All data resets on page refresh. A production version would need a backend, database, or at minimum localStorage to retain sessions between uses.

**Weight total is capped at 10.** The four sliders must sum to exactly 10 or less — the UI enforces this by clamping a slider if you try to exceed it. This keeps scores on a consistent, comparable scale across different configurations. Weights summing to less than 10 are valid but mean some scoring "budget" is unused.

**Renewal rate is now frequency-adjusted.** The formula applies a multiplier based on billing cycle — Monthly × 1, Quarterly × 2, Annual × 4. This means an annual plan renewing at 80% scores higher than a monthly plan at the same raw rate, correctly reflecting the stronger customer commitment a longer billing cycle represents. The multiplier values are opinionated (doubling per tier) and could be calibrated further with real retention data.

**Single active plan per customer assumed.** The mapping table supports one plan per customer. Real customers may have multiple concurrent subscriptions across product lines, which would require a more complex matching model.

**Not tested against live conversion data.** The output has been validated for logical consistency using the seed data, but has not been backtested against real upsell/renewal outcomes. A production engine would need A/B testing or historical validation to confirm the scoring model drives better decisions than the status quo.

---

## Tech

Pure HTML, CSS, and vanilla JavaScript. Single self-contained file. No frameworks, no libraries, no build step, no backend.

---

## What This Project Demonstrates

As a PM, I wrote the product specification for this engine — defining the data inputs, scoring formula, and output format. I then translated that spec directly into a working prototype, iterated on it based on critical self-review, and documented both the decisions and the limitations honestly.

This project reflects how I think about product work:
- Starting with a clearly defined problem and user need
- Designing a solution with transparent, explainable logic
- Iterating based on honest assessment rather than just shipping the first version
- Documenting limitations and a concrete improvement path so stakeholders can make informed decisions about what to build next

The current prototype is intentionally rule-based. The manual weights are a design choice, not a technical constraint — I understand where logistic regression replaces them, where collaborative filtering adds a new capability layer, and what a production ML pipeline would require in terms of data, infrastructure, and evaluation. That roadmap is documented above.

---

## About Me

I'm a Product Manager interested in data-driven product thinking, growth, and building tools that bridge the gap between product strategy and engineering execution.

[LinkedIn](https://linkedin.com/in/anubhutipandey12)
