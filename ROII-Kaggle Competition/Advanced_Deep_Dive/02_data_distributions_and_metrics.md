# 02. Data Distributions & Structural RMSE Metrics

This document details the statistical characteristics of the input data and provides a mathematical analysis of the Root Mean Squared Error (RMSE) metric in the context of structural sequence tracking.

---

## 1. Feature Distributions & Stratigraphic Meanings

To perform effective feature engineering, you must understand the ranges and geological significance of the inputs:

### A. Gamma Ray (GR) Logs
*   **Typical Range:** 0 to 200 API units.
*   **Geological Distribution:**
    *   **Low GR (< 40 API):** Sandstones, carbonates, and clean reservoir rocks (high permeability, low clay).
    *   **High GR (> 100 API):** Shales, claystones, and organic-rich source rocks (low permeability, high clay).
*   **Modeling Tip:** The distribution is often bimodal (representing a sand-shale system). Standardizing this variable (z-scoring) across the whole dataset can hide local regional variations. Local min-max scaling per well is often more robust for sequence matching.

### B. Trajectory Coordinates ($X, Y, Z$)
*   **Horizontal Spread ($X, Y$):** The coordinates correspond to UTM meters. Horizontal lateral lengths range from 4,000 to 12,000 feet.
*   **Elevation ($Z$):** TVD is negative (e.g., -8,000 to -11,000 feet subsea). The vertical variation along a lateral is minor (often less than $\pm 100$ feet), but this small change represents the vertical "wiggles" of the well path.
*   **Inclination ($\theta$):** Concentrated in the range $[85^\circ, 95^\circ]$ for the horizontal section.
*   **Azimuth ($\phi$):** Highly clustered. A single well maintains a constant azimuth (e.g., $180^\circ$ for drilling straight South) with minor steering adjustments.

### C. Target Variable (TVT)
*   **Range:** Typically matches the stratigraphy of the reservoir zone (e.g., 0 to 120 feet).
*   **Behavior:** Very smooth. Geological processes deposit layers in continuous sheets; TVT cannot have sudden step-changes unless a fault is crossed.

---

## 2. Mathematical Analysis of the RMSE Metric in Sequence Modeling

The evaluation metric is the Root Mean Squared Error (RMSE) of the predicted TVT over the masked Toe sequence of length $T$:

$$\text{RMSE} = \sqrt{\frac{1}{T} \sum_{t=1}^{T} (\hat{y}_t - y_t)^2}$$

Let us analyze how prediction errors decompose. Suppose our predicted TVT ($\hat{y}_t$) differs from the true TVT ($y_t$) by a vertical datum offset ($\epsilon_D$), a constant geological slope/dip error ($\epsilon_S$), and local shape noise ($e_t$):

$$\hat{y}_t = y_t + \epsilon_D + \epsilon_S \cdot t + e_t$$

Where:
*   $\epsilon_D$ is the constant shift error at the start of the masked Toe.
*   $\epsilon_S$ is the error in the predicted dip rate (slope error per unit step).
*   $e_t$ is the high-frequency shape error at step $t$, assuming $\mathbb{E}[e_t] = 0$ and $\text{Var}(e_t) = \sigma^2$.

Substituting this error model into the Mean Squared Error (MSE) formulation:

$$\text{MSE} = \frac{1}{T} \sum_{t=1}^{T} (\epsilon_D + \epsilon_S \cdot t + e_t)^2$$

Expanding the summation:

$$\text{MSE} = \frac{1}{T} \sum_{t=1}^{T} \left( \epsilon_D^2 + \epsilon_S^2 t^2 + e_t^2 + 2\epsilon_D\epsilon_S t + 2\epsilon_D e_t + 2\epsilon_S t e_t \right)$$

Taking the expectation $\mathbb{E}[\text{MSE}]$ and utilizing $\mathbb{E}[e_t] = 0$:

$$\mathbb{E}[\text{MSE}] = \epsilon_D^2 + 2\epsilon_D\epsilon_S \left( \frac{T+1}{2} \right) + \epsilon_S^2 \left( \frac{(T+1)(2T+1)}{6} \right) + \sigma^2$$

### Insights from the Mathematical Expansion:
1.  **The Step Complexity of Slope Error:** The slope error $\epsilon_S^2$ is multiplied by a term of order $\mathcal{O}(T^2)$. This means that **slope errors dominate the total error** as the sequence length $T$ increases. A minor error in predicting the geological dip results in cubic error growth over long lateral sections.
2.  **The Offset Penalty:** The datum error $\epsilon_D^2$ creates a constant baseline penalty. If the model misidentifies which geological layer the well starts in, the RMSE remains high regardless of how perfectly the shape wiggles match.
3.  **The Residual Role of Shape Noise ($\sigma^2$):** The local shape matching error $\sigma^2$ has no multiplier. It is a constant additive term. 

> [!IMPORTANT]
> Because the error is dominated by the $\mathcal{O}(T^2)$ slope error and the $\mathcal{O}(1)$ datum offset, **optimizing local model architectures (reducing shape noise $\sigma^2$) is secondary**. Your modeling focus must be on minimizing the structural parameters $\epsilon_D$ and $\epsilon_S$.

---

## 3. Path Calibration & Confidence Bounds

Due to the unidentifiability of the datum from GR in homogeneous sequences, predictions must be calibrated. Instead of outputting a single point prediction, high-performing sequence trackers estimate a probability distribution over the stratigraphic states:

$$P(\text{TVT}_t \mid \text{Observation}_{1:t})$$

The variance of this distribution represents the geological uncertainty:
*   **Low Variance:** The wellbore is crossing sharp layer boundaries (interfaces), resulting in distinct GR transitions. The model has high confidence in its exact vertical position.
*   **High Variance:** The wellbore is drilling inside a homogeneous shale or sand layer with flat GR. The model cannot identify its exact TVT, and the confidence bounds expand.

Ensembles should use this variance to dynamically weight models: trust the Viterbi alignment in high-confidence zones, and fallback to the regional plane-fit dip model in low-confidence zones.
