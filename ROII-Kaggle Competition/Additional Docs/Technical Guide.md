Technical Specification: Data Architecture for Wellbore Geology Prediction
1. System Overview and Objectives
In the landscape of modern horizontal drilling, predictive geology represents a strategic imperative for maintaining the wellbore within optimal stratigraphic windows. Accurate geological forecasting mitigates operational risk and maximizes hydrocarbon recovery by enabling real-time adjustments to steering parameters. This technical specification serves as the architectural blueprint for automating the estimation of True Vertical Thickness (TVT), providing the engineering framework required to transform Gamma Ray (GR) and spatial coordinate data (XYZ) into high-fidelity geological models.The primary engineering objective is the automated generation of accurate TVT values beyond a designated  Prediction Start (PS)  point. To achieve this, the architecture is founded upon three core pillars:
Data Integration:  The systematic reconciliation of dynamic horizontal drilling data with established vertical stratigraphic benchmarks.
Signature Correlation:  The algorithmic matching of Gamma Ray "fingerprints" to identify formation boundaries and sub-layer transitions.
Spatial Orientation:  The application of geometric constraints, including wellbore azimuth and formation dip, to interpret the intersection of the well path with geological surfaces.This document establishes the data structures and logic gates necessary to support a robust, automated prediction pipeline.
2. Data Schema and Input Specifications
The architecture relies on a bifurcated data model to effectively bridge the gap between real-time drilling telemetry and validated geological references. This dual-stream approach ensures that every lateral measurement is contextualized against a high-resolution vertical baseline.
Horizontal Well Data Schema
Data associated with the lateral section (e.g., Well1XXXX__horizontal_well.csv) provides the primary input for the prediction engine.| Parameter | Technical Description / Unit || ------ | ------ || WELLNAME | Unique identifier for the horizontal wellbore || MD | Measured Depth; total wellbore length (Feet) || X, Y, Z | Spatial coordinates defining the 3D path of the wellbore || GR | Gamma Ray values measured along the lateral (API) || TVT | True Vertical Thickness; ground truth for training/calibration (Feet) || TVT_input | Known geology values provided as the baseline until the PS point (Feet) |
Vertical (Typewell) Data Schema
The vertical baseline (e.g., Well1XXXX__typewell__Typewell2XXXX.csv) establishes the stratigraphic sequence for the target area.| Parameter | Technical Description / Unit || ------ | ------ || TVT | True Vertical Thickness; representing vertical depth in the typewell (Feet) || GR | Gamma Ray values measured in the vertical section (API) || Geology | Categorical name of the specific geological layer or formation |
The system architecture enforces a 1:1 mapping logic: each horizontal well is linked to exactly one reference typewell. By synthesizing these files, the system creates a spatial context that allows the model to project horizontal sensor data into a vertical stratigraphic framework.
3. Spatial and Geological Context Requirements
Spatial awareness is the primary determinant of prediction accuracy. The relationship between the drilling direction (Azimuth) and the orientation of geological layers (Dip) dictates how the Gamma Ray signal is interpreted. Without precise geometric context, a signal change cannot be definitively attributed to either a stratigraphic transition or a change in the wellbore trajectory relative to a dipping formation.The engine must evaluate the following spatial constraints:
Azimuth of Horizontal Drilling:  This parameter dictates the expected geological dip encounter (e.g., flat vs. dipping) and is used to align the interpretation of the lateral signal.
Neighboring/Offset Wells:  The engine is required to utilize geological dips from offset wells as  spatial constraints . This prevents unphysical geological interpretations in areas with sparse data.
Projected Well Path:  To facilitate signature alignment, the system must project the 3D well path onto a  Vertical Plane (Azimuth-aligned) , enabling direct correlation with vertical geological markers.
Geology Behavior: Three-State Logic Gates
The model must execute a three-state state-machine logic to determine "Geology Behavior" based on the slope and correlation of the GR-to-TVT match:
Increasing TVT State:  Occurs when the wellbore traverses into stratigraphically deeper sections.
Decreasing TVT State:  Occurs when the wellbore traverses into stratigraphically shallower sections.
Constant TVT State:  Occurs when the wellbore remains within a stabilized stratigraphic interval.This spatial logic transitions the system from raw geometric data to quantified geological trends.
4. Gamma Ray (GR) Signature Correlation Logic
Gamma Ray (GR) data serves as the critical "fingerprint" for geological identification. In this architecture, it is noted that the horizontal well often provides higher resolution than the typewell due to a higher sampling frequency along the lateral section. This increased resolution makes the lateral GR the preferred baseline for identifying subtle formation variations.
Signature Matching and Calibration
The system must execute precise correlation logic between the horizontal signal and the Typewell GR:
The engine shall utilize GR data and known TVT data from the horizontal well  prior  to the Prediction Start (PS) point.
This pre-PS interval serves as the "fingerprint" baseline to calibrate the correlation for the remainder of the lateral.
PS Boundary and Formation Identification
The  Prediction Start (PS)  point is the critical boundary defined by the Measured Depth (MD) where the dependent variable, TVT_input, ceases and automated prediction begins. To maintain stratigraphic consistency, the system must project the horizontal GR onto the TVT track to identify primary formation tops, specifically:
ANCC
ASTNU
BUDABy quantifying the position of these markers relative to the well path, the system ensures the automated forecast adheres to the known petroleum system architecture.
5. Mathematical Framework for Quality Evaluation
A standardized mathematical framework is required to validate the engineering integrity of the prediction model, ensuring that automated outputs remain within acceptable geological tolerances.
Performance Metrics and Resolution
The system evaluates prediction accuracy using the variable  $dTVT$ , which must be calculated for every point in the predicted lateral at a  required resolution of one-foot steps :  $$dTVT = manualTVT - predictedTVT$$The primary performance metric for total prediction quality is the  Root Mean Square Error (RMSE)  of all  $dTVT$  values across the predicted section. A minimized RMSE indicates successful alignment between the automated model and the actual geological structure.
Calibration and Validation Constraints
Within this architecture, "manualTVT" (ground truth) and specific "Formation Tops" are sequestered within the training dataset and are utilized exclusively for model calibration and performance benchmarking. Final model robustness must be ensured through iterative testing against validation wells to verify that the predictive engine can maintain low RMSE levels in diverse structural settings.

