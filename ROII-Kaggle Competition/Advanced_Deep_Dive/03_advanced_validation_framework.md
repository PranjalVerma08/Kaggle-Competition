# 03. Advanced Validation Framework & Leakage Diagnostics

To build a winning solution, you must trust your local validation. Chasing leaderboard feedback leads to overfitting and leaderboard "shakeouts." This document details the physical mechanisms of spatial leakage and provides step-by-step diagnostic tests to build a bulletproof validation system.

---

## 1. The Physics of Spatial Leakage

Why does spatial interpolation fail in geosteering? The answer lies in **fault blocks** and **geological unconformities**.

```
    Fault Block A                       Fault Block B
    (Training Wells)                    (Test Well)
        Well 1       Well 2                 Well 3
   _____\_____________\______________________\____________________
         \             \                      \
  ========\=============\=========             \================== (Stratum boundary)
                                  \            \
                                   \ Fault      \
                                    \ Line       \
```

If we train our model using standard random splits, the validation set will contain points from the same geological fault block as the training set. The model learns to interpolate the vertical plane coordinates:

$$\text{TVT} = a X + b Y + c$$

Because Block A is structurally continuous, the plane fit is highly accurate, resulting in a low local CV. However, the test well (Well 3) is located in Block B, across a fault line. The geological layers have slipped vertically by 40 feet (a fault throw). The regional plane formula from Block A is completely wrong for Block B, leading to a massive leaderboard error.

---

## 2. Step-by-Step Validation Controls

To ensure your model is learning sequence-matching mechanics rather than memorizing spatial coordinates, you must pass three validation gates:

### Gate 1: The Sequence Shuffle Test
*   **Purpose:** Quantify the model's reliance on coordinate leakage vs. sequence alignment.
*   **Step-by-Step Procedure:**
    1.  Train your model on the training folds.
    2.  Create a copy of your validation fold. For each well, randomly shuffle the rows, breaking the Measured Depth (`MD`) sequence order. This preserves the individual $(X, Y, Z, \text{GR})$ values but destroys the continuous path context.
    3.  Generate predictions on the shuffled validation fold and calculate the RMSE: $\text{RMSE}_{\text{shuffled}}$.
    4.  Compare this to the standard sequential validation RMSE: $\text{RMSE}_{\text{seq}}$.
*   **Decision Rule:** 
    
    $$\text{Leakage Index} = \frac{\text{RMSE}_{\text{shuffled}}}{\text{RMSE}_{\text{seq}}}$$

    *   **If $\text{Leakage Index} \approx 1.0$:** The model performs equally well on shuffled data. This is a failure. The model is ignoring sequence structure and memorizing coordinates. Discard the model.
    *   **If $\text{Leakage Index} \gg 2.5$:** The model's performance degrades when sequence continuity is broken, proving it relies on sequential geosteering logic. The model is approved.

### Gate 2: The No-op Flat Line Benchmark
*   **Purpose:** Establish a minimum baseline that any model update must outperform.
*   **Step-by-Step Procedure:**
    1.  For each validation well, calculate the flat-line prediction. Take the last known TVT value from the Heel section ($\text{TVT}_{\text{heel\_end}}$) and hold it constant across the entire masked Toe:
        
        $$\hat{y}_{t}^{\text{no-op}} = \text{TVT}_{\text{heel\_end}}, \quad \forall t \in \text{Toe}$$
        
    2.  Calculate the out-of-fold RMSE of this flat-line prediction: $\text{RMSE}_{\text{no-op}}$.
*   **Decision Rule:** Any proposed model update must satisfy:
    
    $$\text{RMSE}_{\text{model}} < \text{RMSE}_{\text{no-op}} - \delta$$
    
    Where $\delta \ge 0.5$ feet. If a model fails to beat this flat baseline, it is overfitting to noise.

### Gate 3: Spatial Block Cross-Validation (Leave-Spatial-Out)
*   **Purpose:** Simulate extrapolation to unseen fault blocks.
*   **Step-by-Step Procedure:**
    1.  Apply K-Means clustering to the average $X$ and $Y$ coordinates of each well to segment the dataset into $K$ spatial clusters (representing geological blocks).
    2.  Perform cross-validation where each fold corresponds to holding out a complete spatial cluster.
    3.  Verify that your model generalizes well to these remote clusters.

---

## 3. StratifiedGroupKFold Implementation

Below is the Python implementation to split your data by well while stratifying by both drilling azimuth (angle of dip crossing) and median stratigraphic depth.

```python
import numpy as np
import pandas as pd
from sklearn.model_selection import StratifiedGroupKFold

def prepare_stratified_spatial_folds(df, n_splits=5):
    """
    Prepares validation folds that prevent intra-well leakage
    and balance geological properties across splits.
    """
    # 1. Compute drilling azimuth for each well
    # Azimuth = arctan2(dY, dX)
    well_coords = df.groupby('well_id').agg(
        x_start=('X', 'first'), x_end=('X', 'last'),
        y_start=('Y', 'first'), y_end=('Y', 'last'),
        heel_tvt=('TVT', lambda x: x.dropna().median() if len(x.dropna()) > 0 else np.nan)
    ).reset_index()
    
    dx = well_coords['x_end'] - well_coords['x_start']
    dy = well_coords['y_end'] - well_coords['y_start']
    well_coords['azimuth'] = np.degrees(np.arctan2(dx, dy)) % 360
    
    # Fill missing TVT values with overall median for stratification purposes
    overall_median = well_coords['heel_tvt'].median()
    well_coords['heel_tvt'] = well_coords['heel_tvt'].fillna(overall_median)
    
    # 2. Bin properties for stratification
    well_coords['azimuth_bin'] = pd.qcut(well_coords['azimuth'], q=4, labels=False, duplicates='drop')
    well_coords['tvt_bin'] = pd.qcut(well_coords['heel_tvt'], q=4, labels=False, duplicates='drop')
    
    # Combine bins into a single stratification key
    well_coords['stratify_key'] = (well_coords['azimuth_bin'].astype(str) + "_" + 
                                   well_coords['tvt_bin'].astype(str))
    
    # Map stratification keys back to the main dataframe
    df = df.merge(well_coords[['well_id', 'stratify_key']], on='well_id', how='left')
    
    # 3. Create Folds
    sgkf = StratifiedGroupKFold(n_splits=n_splits)
    df['fold'] = -1
    
    for fold, (train_idx, val_idx) in enumerate(sgkf.split(df, df['stratify_key'], groups=df['well_id'])):
        df.loc[val_idx, 'fold'] = fold
        
    return df
```
