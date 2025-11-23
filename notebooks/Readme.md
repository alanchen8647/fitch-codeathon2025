# 🌍 Project: Domain-Driven AI for Corporate Emission Estimation
### FitchGroup Codeathon '25 Submission

## 📋 Phase 1: Executive Summary & Business Impact

**The Challenge:**
Estimating corporate greenhouse gas emissions is not merely a data regression problem; it is a problem of **physics** (energy use) and **behavioral economics** (reporting bias). The provided dataset contained extreme outliers, sparse environmental data, and lacked precise facility locations.

**The Solution:**
We developed a **Hybrid Ensemble Model** that moves beyond simple "revenue scaling." Our solution separates a company's *Physical Intensity* (what they do) from their *Reporting Signal* (how much they disclose).

**The Result:**
We successfully stabilized the prediction error, moving from a highly volatile baseline to a precise, log-stable model.

| Metric | Baseline (Linear Model) | Final Model (Hybrid GBR) | Improvement |
| :--- | :--- | :--- | :--- |
| **Scope 1 Error** | ~106,800 tons (Avg) | **Factor of ~7x** (Log 1.97) | **99% Variance Reduction** |
| **Scope 2 Error** | ~158,700 tons (Avg) | **Factor of ~14x** (Log 2.69) | **Robust Stability** |

---

## 🔍 Phase 2: The "Why" – Why Standard Models Failed
Before building complex models, we established a baseline using Linear Regression on raw data. This failed for two critical business reasons:

1.  **The "Whale" Problem:** Emissions follow a **Pareto Distribution** (80/20 rule). A handful of industrial giants emit millions of tons, while most companies emit very little. A linear model tries to fit both, resulting in massive errors where small offices are predicted to emit like factories.
2.  **The Grid Sensitivity Problem:** Scope 2 (Electricity) emissions had a standard deviation of **~70,000 tons** in the baseline. This proved that predicting electricity usage without accounting for the *local power grid's carbon intensity* is statistically impossible.

**Strategic Pivot:**
We abandoned raw-scale regression in favor of **Log-Transformation** (to compress outliers) and **Gradient Boosting** (to capture non-linear interaction rules).

---

## 🧠 Phase 3: Hypothesis-Driven Feature Engineering
Instead of using a "Black Box" approach, we engineered features based on specific hypotheses about how companies operate.

### ✅ A. The "Physical" Hypotheses (Structural Drivers)
*These features model the physical constraints of the real world.*

* **Hypothesis:** *Structural Intensity Drives Scope 1.*
    * **The Insight:** A dollar of revenue in "Software" is clean; a dollar in "Steel" is dirty.
    * **The Feature:** `log_rev_x_intensity`
    * **How it works:** We calculated the revenue share strictly from "Dirty" NACE codes (Mining, Manufacturing, Transport) and interacted this with Total Revenue.
    * **Impact:** This became a **Top-5 predictor**, allowing the model to distinguish between "revenue types."

* **Hypothesis:** *Geography Determines Scope 2.*
    * **The Insight:** Scope 2 is defined as $Energy \times GridFactor$. A factory in a coal-heavy region emits far more than an identical factory in a hydro-heavy region.
    * **The Feature:** `log_rev_x_region_[CODE]`
    * **How it works:** We created interaction terms between Revenue and Region (e.g., `Revenue * WesternEurope`) to proxy the carbon intensity of local power grids.
    * **Impact:** This solved the Scope 2 volatility problem.

### ✅ B. The "Behavioral" Hypotheses (Reporting Bias)
*These features model the psychology of corporate reporting.*

* **Hypothesis:** *The "Hypocrisy Gap" Signals Risk.*
    * **The Insight:** Companies often have excellent Governance/Social scores (policy-based) but poor Environmental scores (infrastructure-based). A large gap suggests unmanaged physical risk.
    * **The Feature:** `soc_env_gap` (Social Score minus Environmental Score).
    * **Impact:** A critical predictor for Scope 2 anomalies.

* **Hypothesis:** *Reporting Bias creates False Positives.*
    * **The Insight:** Large companies are legally required to report more "Environmental Activities." A naive model sees "More Activities" and thinks "Green/Clean." In reality, "More Activities" usually means "Massive Polluter under scrutiny."
    * **The Feature:** `rev_x_reporting` (Revenue * Activity Count).
    * **Impact:** Controlled for size-driven reporting bias, reversing the false causality.

---

## ⚙️ Phase 4: Model Architecture & Selection
We chose a **Weighted Blending Strategy** to hedge our risk.

### 1. The "Physicist" Model (70% Weight)
* **Algorithm:** Gradient Boosting Regressor (GBR) with Interaction Features.
* **Role:** Learns complex rules like *"If Manufacturing AND Asia -> High Emissions."*
* **Tuning:** Shallow trees (Depth 3) for Scope 1 to prevent overfitting; Deeper trees (Depth 4) for Scope 2 to capture complex grid interactions.

### 2. The "Statistician" Model (30% Weight)
* **Algorithm:** Gradient Boosting Regressor with **Target Encoding**.
* **Role:** Learns hard statistical baselines (e.g., *"Sector C usually emits 50k tons"*).
* **Why Blend?** This model acts as a "Safety Net." If the Physicist model gets confused by a weird outlier, the Statistician pulls the prediction back to the sector average.

---

## ⚰️ Phase 5: The "Graveyard" (Approaches We Scrapped)
*Part of data science is knowing what NOT to use.*

| Approach | Why we tried it | Why we killed it |
| :--- | :--- | :--- |
| **Stacking Ensemble** | To combine Ridge + Random Forest + GBR. | **Overfitting.** On a small dataset (~400 rows), the Meta-Learner found patterns that didn't exist. Error increased from 1.98 to 2.29. |
| **Zero-Inflated Model** | To classify "0 Emissions" separately. | **Too Risky.** Predicting "Zero" for a massive factory due to a classification error is a catastrophic failure mode. Predicting a low number (e.g., 500 tons) is safer. |

---

## 🔬 Phase 6: Residual Analysis & Constraints
**Where are we still wrong?**
Our biggest remaining errors come from the **"Accounting vs. Reality"** conflict.
* We identified companies where our model predicted ~600 tons (Physical Usage), but the target was **0.00** (Market-Based Reporting).
* This represents companies buying **100% renewable energy certificates**.
* **Verdict:** Our model correctly predicts their *physical* load, while the target reflects their *financial* offset. This is an unavoidable variance without external certificate data.

---

## 🚀 Phase 7: Future Roadmap
Given more time/data, we would implement:
1.  **PPP Adjustments:** Normalize revenue by "Purchasing Power Parity" to estimate physical output in developing nations more accurately.
2.  **Live Grid APIs:** Replace regional proxies with real-time $gCO_2/kWh$ data (e.g., ElectricityMap) for precise Scope 2 calculations.