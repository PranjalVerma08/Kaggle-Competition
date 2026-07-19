# 06. Leaderboard Pipeline Code Templates

This document provides modular Python implementations for the core components of a high-performing geosteering pipeline. It includes Q-3D wellbore tortuosity calculation, Viterbi sequence tracking, regional geological plane fitting, and LightGBM residual alignment.

---

## 1. Q-3D Wellbore Tortuosity Feature Extractor
This module calculates the cumulative changes in inclination and azimuth along the well path, capturing the steering frequency of the well.

```python
import numpy as np
import pandas as pd

def calculate_q3d_tortuosity(md, inc_deg, az_deg):
    """
    Computes Q-3D tortuosity from wellbore survey stations.
    
    Parameters:
    -----------
    md : np.ndarray
        Measured Depth along the wellpath.
    inc_deg : np.ndarray
        Wellbore inclination in degrees.
    az_deg : np.ndarray
        Wellbore azimuth in degrees.
        
    Returns:
    --------
    tortuosity : np.ndarray
        Accumulated tortuosity values of the same length as md.
    """
    # Convert angles to radians
    inc = np.radians(inc_deg)
    az = np.radians(az_deg)
    
    n = len(md)
    dls = np.zeros(n)  # Dogleg Severity / local angle change
    
    for i in range(1, n):
        d_md = md[i] - md[i-1]
        if d_md <= 0:
            continue
            
        # Spherical curvature calculation (cosine law of geodesics)
        cos_alpha = (np.cos(inc[i-1]) * np.cos(inc[i]) + 
                     np.sin(inc[i-1]) * np.sin(inc[i]) * np.cos(az[i] - az[i-1]))
        
        # Clip to prevent floating point issues outside [-1, 1]
        cos_alpha = np.clip(cos_alpha, -1.0, 1.0)
        alpha = np.arccos(cos_alpha)  # Angle change in radians
        
        # Local tortuosity: angle change normalized by depth step
        dls[i] = alpha / d_md
        
    # Compute cumulative tortuosity along the path
    cumulative_tortuosity = np.cumsum(dls)
    return cumulative_tortuosity
```

---

## 2. Viterbi Path Alignment (Sequence Tracking)
This class implements dynamic programming to align the lateral Gamma Ray sequence with the vertical typewell reference.

```python
import numpy as np

class ViterbiGeosteeringTracker:
    def __init__(self, typewell_tvt, typewell_gr, transition_penalty=5.0):
        """
        Parameters:
        -----------
        typewell_tvt : np.ndarray
            True Vertical Thickness column of the vertical typewell.
        typewell_gr : np.ndarray
            Gamma Ray readings corresponding to each TVT state.
        transition_penalty : float
            Cost multiplier for deviating from physical trajectory step.
        """
        self.states = typewell_tvt
        self.state_gr = typewell_gr
        self.penalty = transition_penalty
        self.n_states = len(typewell_tvt)

    def track(self, lateral_gr, delta_z):
        """
        Finds the optimal TVT sequence.
        
        Parameters:
        -----------
        lateral_gr : np.ndarray
            Gamma Ray sequence recorded along the horizontal well.
        delta_z : np.ndarray
            Physical change in well elevation between sequential logging points.
        """
        T = len(lateral_gr)
        dp = np.zeros((self.n_states, T))
        path_ptr = np.zeros((self.n_states, T), dtype=int)
        
        # Initialize first state based on Heel anchoring (assumed known)
        # For simplicity, we initialize with uniform probabilities matching Heel TVT
        dp[:, 0] = (self.state_gr - lateral_gr[0]) ** 2
        
        # Dynamic programming forward pass
        for t in range(1, T):
            dz = delta_z[t]
            for s in range(self.n_states):
                # Calculate costs from all previous states s_prev
                tvt_change = self.states[s] - self.states
                
                # Penalty: deviation of stratigraphic transition from physical delta_z
                transition_costs = self.penalty * (tvt_change - dz) ** 2
                
                # Total cost for transitioning from s_prev to s at time t
                total_costs = dp[:, t-1] + transition_costs
                
                best_prev = np.argmin(total_costs)
                path_ptr[s, t] = best_prev
                
                # Measurement cost (GR match) + transition cost
                gr_match_cost = (self.state_gr[s] - lateral_gr[t]) ** 2
                dp[s, t] = total_costs[best_prev] + gr_match_cost
                
        # Backward pass to reconstruct optimal path
        opt_path = np.zeros(T, dtype=int)
        opt_path[-1] = np.argmin(dp[:, -1])
        
        for t in range(T-2, -1, -1):
            opt_path[t] = path_ptr[opt_path[t+1], t+1]
            
        return self.states[opt_path]
```

---

## 3. Regional Structural Plane-Fitting (Low-Frequency Trend)
Fits a 3D regional dip plane using geographical coordinates ($X, Y$) and known vertical elevations from reference typewells to predict the base datum plane.

```python
from sklearn.linear_model import Ridge

class RegionalStructuralModel:
    def __init__(self, alpha=1.0):
        self.plane_model = Ridge(alpha=alpha)
        
    def fit(self, x_coords, y_coords, known_tvt):
        """
        Fits a structural plane: TVT = a * X + b * Y + c
        """
        features = np.column_stack((x_coords, y_coords))
        self.plane_model.fit(features, known_tvt)
        
    def predict_trend(self, x_coords, y_coords):
        features = np.column_stack((x_coords, y_coords))
        return self.plane_model.predict(features)
```

---

## 4. LightGBM Residual Corrector (The Machine Learning Stack)
Once the Viterbi path and Regional Dip trend are computed, we use a LightGBM regressor to predict the residual error.

```python
import lightgbm as lgb

def train_lgb_residual_model(train_df, val_df, features, target_col='tvt_residual'):
    """
    Trains a LightGBM model to predict TVT residual corrections.
    """
    params = {
        'objective': 'regression',
        'metric': 'rmse',
        'learning_rate': 0.05,
        'num_leaves': 31,
        'max_depth': 6,
        'feature_fraction': 0.8,
        'bagging_fraction': 0.9,
        'bagging_freq': 1,
        'verbose': -1
    }
    
    train_data = lgb.Dataset(train_df[features], label=train_df[target_col])
    val_data = lgb.Dataset(val_df[features], label=val_df[target_col], reference=train_data)
    
    model = lgb.train(
        params,
        train_data,
        num_boost_round=1000,
        valid_sets=[train_data, val_data],
        callbacks=[lgb.early_stopping(50), lgb.log_evaluation(100)]
    )
    
    return model
```

---

## 5. Integrating the Pipeline (High-Level Execution)

```python
# Pseudo-pipeline matching the top-tier solutions:
# 1. Load data
# train_wells, test_wells, typewells = load_data()

# 2. Extract Q-3D tortuosity
# test_wells['tortuosity'] = calculate_q3d_tortuosity(test_wells['MD'], test_wells['inc'], test_wells['az'])

# 3. Predict the base plane-fit trend using nearest typewells
# structural_model = RegionalStructuralModel()
# structural_model.fit(typewells['X'], typewells['Y'], typewells['TVT'])
# test_wells['trend_tvt'] = structural_model.predict_trend(test_wells['X'], test_wells['Y'])

# 4. Generate physics-based alignment for test well
# tracker = ViterbiGeosteeringTracker(typewells['TVT'], typewells['GR'])
# test_wells['viterbi_tvt'] = tracker.track(test_wells['GR'], test_wells['delta_Z'])

# 5. Correct Viterbi tracking with the LightGBM residual model
# test_wells['pred_residual'] = lgb_model.predict(test_wells[feature_cols])
# test_wells['final_pred_tvt'] = test_wells['viterbi_tvt'] + test_wells['pred_residual']
```
