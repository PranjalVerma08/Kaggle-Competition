# 07. Competition Dataset Files & Schema Details

To build a robust submission pipeline that passes Kaggle's hidden test evaluation, you must understand the directory structures, schema formats, and constraints of the data files.

---

## 1. Directory Tree & File Inventory

The competition dataset is located at `/kaggle/input/competitions/rogii-wellbore-geology-prediction/`. It is structured as follows:

```
/kaggle/input/competitions/rogii-wellbore-geology-prediction/
├── sample_submission.csv
├── AI_wellbore_geology_prediction_task_en.pptx
├── train/
│   ├── {WELLNAME}__horizontal_well.csv  (773 files)
│   ├── {WELLNAME}__typewell.csv         (773 files)
│   └── {WELLNAME}.png                   (773 files)
└── test/
    ├── {WELLNAME}__horizontal_well.csv  (3 files in public subset)
    └── {WELLNAME}__typewell.csv         (3 files in public subset)
```

### File Counts Summary
*   **Total train wells:** 773
    *   `train/` contains 2,319 files (773 laterals, 773 verticals, and 773 cross-section plots).
*   **Total test wells:** 3 (Public leaderboard subset)
    *   `test/` contains 6 files (3 laterals and 3 verticals).
    *   **Crucial Alert:** The test folder **does not contain any `.png` cross-section plots**.

---

## 2. Detailed File Schemas & Column Formats

Every well is identified by an 8-character unique alphanumeric hash (e.g., `000d7d20`, `00bbac68`).

### A. Horizontal Well File (`{WELLNAME}__horizontal_well.csv`)
This file records measurements along the horizontal wellbore trajectory as it drills through the formation.

| Column Name | Data Type | Units | Description | Present In Train | Present In Test |
| :--- | :--- | :--- | :--- | :---: | :---: |
| `MD` | Float | Feet | Measured Depth along the borehole path from the surface. | Yes | Yes |
| `X` | Float | Feet | Easting coordinate (horizontal position). | Yes | Yes |
| `Y` | Float | Feet | Northing coordinate (horizontal position). | Yes | Yes |
| `Z` | Float | Feet | True Vertical Depth (elevation below surface). Negative values. | Yes | Yes |
| `GR` | Float | API | Gamma Ray sensor readings (clay content indicator). | Yes | Yes |
| `TVT` | Float | Feet | **Target Variable**. Stratigraphic thickness relative to datum. | Yes | **No** |
| `TVT_input` | Float | Feet | TVT values containing known Heel context and `NaN` values. | **No** | Yes |
| `Formation` (e.g., `BUDA`) | Float | Feet | Depths of individual formation surfaces. | Yes | **No** |

### B. Typewell File (`{WELLNAME}__typewell.csv`)
The vertical reference log drilled nearby to serve as a geological blueprint.

| Column Name | Data Type | Units | Description | Present In Train | Present In Test |
| :--- | :--- | :--- | :--- | :---: | :---: |
| `TVT` | Float | Feet | Stratigraphic thickness coordinate index. | Yes | Yes |
| `GR` | Float | API | Gamma Ray profile characteristic of the layers. | Yes | Yes |
| `Geology` | String | Categorical | Names of specific geological intervals. | Yes | Yes |

### C. Sample Submission File (`sample_submission.csv`)
Template defining the output structure for the prediction file.

| Column Name | Data Type | Description | Example Format |
| :--- | :--- | :--- | :--- |
| `id` | String | Well identifier and row index. | `000d7d20_142` |
| `tvt` | Float | Predicted True Vertical Thickness value. | `15.428` |

---

## 3. Crucial Modeling Considerations & Rules

To avoid submission errors, keep these factors in mind when designing your training and inference loops:

### A. Dynamic File Scanning for Code Evaluation
Because this is a Kaggle code competition:
*   The 3 test wells in the `test/` directory are just placeholders for your public notebook runs.
*   When you submit your code, **Kaggle replaces the `test/` directory with a hidden test set containing hundreds of unseen wells**.
*   **The Fix:** You must scan the directory dynamically using glob:
    
    ```python
    import glob
    test_horizontal_paths = sorted(glob.glob('/kaggle/input/competitions/rogii-wellbore-geology-prediction/test/*__horizontal_well.csv'))
    ```

### B. The Heel Context & Target Bins
In test wells, the `TVT_input` column has:
*   **Known values** (the first 20–30% of rows), which represent the Heel context.
*   **NaN values** (the remaining 70–80% of rows), which represent the evaluation zone.
Your model must use the known Heel rows as the starting condition (anchor) and predict TVT values for the NaN rows.

### C. The PNG Exclusion Constraint
The training set includes `{WELLNAME}.png` files showing cross-section figures. However, these are omitted from the test directory. 
*   **Constraint:** You **cannot** use image models or computer vision networks that process the PNG plots as part of your final submission pipeline, since those files do not exist at inference time.

### D. Reconstructing IDs for Submission
The submission file expects a row-wise mapping. The `id` column must be constructed by pairing the 8-character well hash with the row index of the lateral wellbore:

```python
# Constructing prediction IDs
test_well_id = "000d7d20"
df_lateral = pd.read_csv(f"/kaggle/input/competitions/rogii-wellbore-geology-prediction/test/{test_well_id}__horizontal_well.csv")

# Create IDs for rows where TVT_input is NaN (the prediction zone)
prediction_indices = df_lateral[df_lateral['TVT_input'].isna()].index
submission_ids = [f"{test_well_id}_{idx}" for idx in prediction_indices]
```
