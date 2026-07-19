# 02. Data Input Overview & Evaluation Metrics

To construct an effective geosteering pipeline, we must dissect the inputs, targets, and evaluation metrics of the competition.

---

## 1. Input Data Structure

The competition dataset consists of two primary datasets: **Horizontal Well Log Sequences** (where we need to predict TVT) and **Typewell Vertical Reference Logs** (which act as our geological ground truth reference).

### A. Horizontal Well Dataset (The Laterals)
These files represent the paths and sensors of the horizontal wells. They contain sequential logging-while-drilling (LWD) measurements.

| Feature Name | Type | Description |
| :--- | :--- | :--- |
| `well_id` | Categorical / ID | Unique identifier for the horizontal well. |
| `MD` (Measured Depth) | Numeric (Float) | Cumulative length of the wellbore path from the surface (feet/meters). |
| `X` | Numeric (Float) | Easting UTM spatial coordinate. |
| `Y` | Numeric (Float) | Northing UTM spatial coordinate. |
| `Z` (True Vertical Depth) | Numeric (Float) | Vertical elevation below the surface/sea level. Always negative. |
| `GR` (Gamma Ray) | Numeric (Float) | Real-time Clay/Shale index sensor reading (API units). Typically ranges from 0 to 200. |
| `TVT` (True Vertical Thickness) | Numeric (Float) | **Target Variable**. Stratigraphic depth relative to the geological datum. |

> [!NOTE]
> *   **Training Wells:** Have complete `TVT` coverage along the entire lateral length (Heel to Toe).
> *   **Test Wells:** Have `TVT` populated for the **Heel section** (the first 20-30% of the horizontal lateral) to provide context/anchor parameters, but have the `TVT` completely masked (NaN or missing) for the **Toe section** (the remaining 70-80%).

### B. Typewell Reference Dataset (The Verticals)
These vertical wells are drilled nearby to map the region's geological layers before horizontal drilling begins. They serve as the baseline template.

| Feature Name | Type | Description |
| :--- | :--- | :--- |
| `typewell_id` | Categorical / ID | Unique identifier for the vertical reference well. |
| `TVT` | Numeric (Float) | Stratigraphic height column representing the vertical layers. |
| `GR` (Gamma Ray) | Numeric (Float) | Gamma Ray profile characteristic of each geological layer. |
| `Formation` | Categorical | Names of specific geological formations/strata corresponding to each TVT interval. |

```
Typewell GR Profile               Lateral GR Sequence (Stretched/Squeezed)
      TVT                               MD (Measured Depth)
   0  |-----\                        Heel |-----\
      |      \                            |      \_______
  10  |======/                            |======/       \_____ (Toe)
      |     /                             |     / 
  20  |____/                              |____/
```

---

## 2. Target Variable: TVT
The target is `TVT` (True Vertical Thickness). Unlike `Z` (which represents the physical elevation), `TVT` represents the geological layer height. 

Because geological layers dip (tilt) over space, a drill bit maintaining a constant elevation $Z$ will cross different rock layers, causing the $TVT$ to change.

$$\text{TVT}_t = Z_t - \text{StructureElevation}(X_t, Y_t)$$

Your model must learn to predict how the structural plane tilts across the masked toe-end to accurately estimate `TVT`.

---

## 3. Evaluation Metric: Root Mean Squared Error (RMSE)

The competition is evaluated on the **RMSE** of the predicted `TVT` values in the masked toe-end sections of the test wells.

$$\text{RMSE} = \sqrt{\frac{1}{N} \sum_{i=1}^{N} (\hat{y}_i - y_i)^2}$$

Where:
*   $\hat{y}_i$ is the predicted TVT for point $i$ in the masked toe-end of the test lateral.
*   $y_i$ is the actual true TVT.
*   $N$ is the total number of masked points across all test wells.

### Engineering Implications of the Metric
1.  **Sensitivity to Outliers:** RMSE penalizes large errors heavily due to squaring. A few large misalignments (e.g., getting off by 50 feet due to a missed geological fault) will ruin your leaderboard score.
2.  **Smoothness Penalty:** Because geological layers change smoothly, physically impossible spikes or jitter in predictions will increase the RMSE. Predictions must be constrained to fit smooth physical paths.
3.  **Trend Drift:** If your model fails to predict the structural dip (slope) of the layers, the errors will accumulate along the lateral, leading to a massive drift in predicted TVT at the toe-end. Predicting the low-frequency trend correctly is critical to keeping the RMSE low.
