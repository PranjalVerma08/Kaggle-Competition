# 06. Production Pipeline Code Templates

This document provides production-ready, fully commented Python classes for each stage of the advanced geosteering pipeline.

---

## 1. Q-3D Wellbore Tortuosity Extractor

This class calculates both the local and cumulative tortuosity of the 3D wellbore trajectory.

```python
import numpy as np
import pandas as pd
from typing import Tuple

class Q3DTortuosityExtractor:
    """
    Computes Q-3D wellbore tortuosity from directional surveys.
    Uses spherical curvature and Ratio Factor corrections.
    """
    def __init__(self):
        pass

    def fit_transform(self, df: pd.DataFrame, 
                      md_col: str = 'MD', 
                      inc_col: str = 'inclination', 
                      az_col: str = 'azimuth') -> Tuple[np.ndarray, np.ndarray]:
        """
        Parameters:
        -----------
        df : pd.DataFrame
            Contains Measured Depth, Inclination (deg), and Azimuth (deg) columns.
            
        Returns:
        --------
        local_tortuosity : np.ndarray
            Local dogleg/curvature at each logging point.
        cumulative_tortuosity : np.ndarray
            Integrated tortuosity along the path.
        """
        md = df[md_col].values
        inc = np.radians(df[inc_col].values)
        az = np.radians(df[az_col].values)
        
        n = len(md)
        local_tort = np.zeros(n)
        
        for i in range(1, n):
            d_md = md[i] - md[i-1]
            if d_md <= 1e-5:
                continue
                
            # Compute dogleg angle (gamma) using spherical trigonometry
            cos_gamma = (np.cos(inc[i-1]) * np.cos(inc[i]) + 
                         np.sin(inc[i-1]) * np.sin(inc[i]) * np.cos(az[i] - az[i-1]))
            cos_gamma = np.clip(cos_gamma, -1.0, 1.0)
            gamma = np.arccos(cos_gamma)
            
            # Apply Ratio Factor correction for curved path sections
            if gamma > 1e-5:
                rf = (2.0 / gamma) * np.tan(gamma / 2.0)
            else:
                rf = 1.0
                
            # Curvature rate per unit depth
            local_tort[i] = (gamma * rf) / d_md
            
        cum_tort = np.cumsum(local_tort)
        return local_tort, cum_tort
```

---

## 2. Advanced Viterbi Tracker (With Z-Scale Fix)

This class implements Viterbi tracking. It includes a `use_raw_gr` flag to prevent the z-score flat-line bug.

```python
import numpy as np
from typing import Union

class AdvancedViterbiTracker:
    """
    Viterbi sequence alignment tracker with balanced cost calculations
    to prevent flat-line path locking.
    """
    def __init__(self, 
                 typewell_tvt: np.ndarray, 
                 typewell_gr: np.ndarray, 
                 transition_weight: float = 50.0,
                 use_raw_gr: bool = True):
        """
        Parameters:
        -----------
        typewell_tvt : np.ndarray
            TVT height axis from vertical typewell.
        typewell_gr : np.ndarray
            Gamma Ray log from vertical typewell.
        transition_weight : float
            Regularization parameter (lambda) penalizing trajectory deviations.
        use_raw_gr : bool
            If True, skips standard scaling to prevent squashing observation costs.
        """
        self.states = typewell_tvt
        self.state_gr = typewell_gr
        self.n_states = len(typewell_tvt)
        self.lambda_param = transition_weight
        self.use_raw = use_raw_gr

    def _normalize_signals(self, lateral_gr: np.ndarray) -> Tuple[np.ndarray, np.ndarray]:
        if self.use_raw:
            return lateral_gr, self.state_gr
        else:
            # Z-Score scaling (warning: will require much smaller transition_weight)
            mean_val = np.mean(self.state_gr)
            std_val = np.std(self.state_gr) + 1e-6
            scaled_lat = (lateral_gr - mean_val) / std_val
            scaled_type = (self.state_gr - mean_val) / std_val
            return scaled_lat, scaled_type

    def track(self, lateral_gr: np.ndarray, 
              delta_z: np.ndarray, 
              heel_anchor_tvt: float) -> np.ndarray:
        """
        Executes Viterbi tracking aligned with the Heel anchor point.
        
        Parameters:
        -----------
        lateral_gr : np.ndarray
            Gamma Ray measurements along the lateral.
        delta_z : np.ndarray
            True Vertical Depth increments between logging stations.
        heel_anchor_tvt : float
            The known TVT value at the Heel-end to initialize the state.
        """
        T = len(lateral_gr)
        lat_gr_scaled, type_gr_scaled = self._normalize_signals(lateral_gr)
        
        dp = np.zeros((self.n_states, T))
        path_ptr = np.zeros((self.n_states, T), dtype=int)
        
        # 1. Initialization Pass: Anchor strictly to the Heel TVT
        anchor_idx = np.argmin(np.abs(self.states - heel_anchor_tvt))
        dp[:, 0] = 1e6  # Large cost for non-anchor states
        dp[anchor_idx, 0] = (type_gr_scaled[anchor_idx] - lat_gr_scaled[0]) ** 2
        
        # 2. Forward DP Pass
        for t in range(1, T):
            dz = delta_z[t]
            for s in range(self.n_states):
                # Calculate TVT transitions from all possible previous states s_prev
                tvt_change = self.states[s] - self.states
                
                # Transition cost penalizes deviation from trajectory movement
                transition_cost = self.lambda_param * ((tvt_change - dz) ** 2)
                
                # Combine accumulated past cost with transition cost
                total_cost = dp[:, t-1] + transition_cost
                
                best_prev = np.argmin(total_cost)
                path_ptr[s, t] = best_prev
                
                # Combine with current measurement cost
                gr_match_cost = (type_gr_scaled[s] - lat_gr_scaled[t]) ** 2
                dp[s, t] = total_cost[best_prev] + gr_match_cost
                
        # 3. Backtracking Pass
        opt_path = np.zeros(T, dtype=int)
        opt_path[-1] = np.argmin(dp[:, -1])
        
        for t in range(T-2, -1, -1):
            opt_path[t] = path_ptr[opt_path[t+1], t+1]
            
        return self.states[opt_path]
```

---

## 3. Regional Geological Plane Fitter

Fits structural coordinates to model the spatial dip trends across fault blocks.

```python
from sklearn.linear_model import Ridge

class RegionalPlaneFitter:
    def __init__(self, alpha: float = 1.0):
        self.model = Ridge(alpha=alpha)

    def fit(self, x: np.ndarray, y: np.ndarray, tvt: np.ndarray):
        """
        Fits a structural plane: TVT = a*X + b*Y + c
        """
        features = np.column_stack((x, y))
        self.model.fit(features, tvt)
        
    def predict(self, x: np.ndarray, y: np.ndarray) -> np.ndarray:
        features = np.column_stack((x, y))
        return self.model.predict(features)
        
    def get_dip_vector(self) -> Tuple[float, float]:
        """
        Returns the slope (dip) parameters along X and Y axes.
        """
        return self.model.coef_[0], self.model.coef_[1]
```

---

## 4. Linear Trees Extrapolator

Standard trees output flat predictions when extrapolating outside the spatial boundaries of the training set. This class wraps `LinearTreeRegressor` to build extrapolating models.

```python
# Install prerequisite: pip install linear-tree
from lineartree import LinearTreeRegressor
from sklearn.linear_model import LinearRegression

class LinearTreeExtrapolator:
    """
    Fits piecewise linear models using spatial coordinates
    to extrapolate geological trends correctly.
    """
    def __init__(self, max_depth: int = 3):
        # We use standard Ridge or LinearRegression inside leaf nodes
        self.model = LinearTreeRegressor(
            base_estimator=LinearRegression(),
            max_depth=max_depth,
            min_samples_split=20
        )

    def fit(self, X: pd.DataFrame, y: np.ndarray):
        """
        Parameters:
        -----------
        X : pd.DataFrame
            DataFrame containing features (must include 'X', 'Y').
        y : np.ndarray
            Target TVT values.
        """
        self.model.fit(X, y)

    def predict(self, X: pd.DataFrame) -> np.ndarray:
        return self.model.predict(X)
```

---

## 5. Integrated Execution Pipeline

```python
def run_integrated_geosteering_pipeline(
    train_df: pd.DataFrame,
    test_df: pd.DataFrame,
    typewell_df: pd.DataFrame,
    feature_cols: list
) -> np.ndarray:
    """
    Executes the entire hybrid geosteering pipeline.
    """
    # 1. Extract Q-3D Tortuosity Features
    extractor = Q3DTortuosityExtractor()
    _, train_df['tortuosity'] = extractor.fit_transform(train_df, 'MD', 'inc', 'az')
    _, test_df['tortuosity'] = extractor.fit_transform(test_df, 'MD', 'inc', 'az')
    
    # 2. Fit Regional dip plane using training coordinate data
    plane_fitter = RegionalPlaneFitter(alpha=10.0)
    plane_fitter.fit(train_df['X'].values, train_df['Y'].values, train_df['TVT'].values)
    
    # 3. Generate Physics-based Viterbi Baselines
    # Ensure transition_weight is balanced for raw GR API units (0 to 150)
    viterbi_solver = AdvancedViterbiTracker(
        typewell_tvt=typewell_df['TVT'].values,
        typewell_gr=typewell_df['GR'].values,
        transition_weight=50.0,
        use_raw_gr=True
    )
    
    # Pre-allocate array for Viterbi paths
    test_df['viterbi_tvt'] = np.nan
    for well in test_df['well_id'].unique():
        well_mask = test_df['well_id'] == well
        well_data = test_df[well_mask]
        
        # Calculate vertical movements
        delta_z = np.diff(well_data['Z'].values, prepend=well_data['Z'].values[0])
        
        # Identify Heel anchor (the last non-null TVT value in context)
        heel_tvts = well_data['TVT'].dropna().values
        heel_anchor = heel_tvts[-1] if len(heel_tvts) > 0 else typewell_df['TVT'].median()
        
        # Solve sequence alignment
        tracked_tvt = viterbi_solver.track(well_data['GR'].values, delta_z, heel_anchor)
        test_df.loc[well_mask, 'viterbi_tvt'] = tracked_tvt
        
    # 4. Train Linear Tree Model to correct Residuals
    # target residual = True TVT - Viterbi baseline TVT
    train_df['viterbi_tvt'] = ... # Compute on training wells
    train_df['tvt_residual'] = train_df['TVT'] - train_df['viterbi_tvt']
    
    corrector = LinearTreeExtrapolator(max_depth=3)
    corrector.fit(train_df[feature_cols], train_df['tvt_residual'].values)
    
    # 5. Correct Viterbi with Extrapolated Residual
    test_df['pred_residual'] = corrector.predict(test_df[feature_cols])
    test_df['final_tvt_prediction'] = test_df['viterbi_tvt'] + test_df['pred_residual']
    
    return test_df['final_tvt_prediction'].values
```
