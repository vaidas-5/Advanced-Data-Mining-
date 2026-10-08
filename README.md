# ECG Data Preparation Pipeline

This project prepares ECG heartbeat data for arrhythmia classification using the **MIT-BIH Arrhythmia Database**. The pipeline performs data download, ECG lead selection, annotation mapping, preprocessing, train/validation/test splitting, class-weight calculation, data-quality checks, and reproducibility checks.

The full implementation is provided in:

```text
Data_Processing_2nd_edition.ipynb
```

The notebook should be run from top to bottom.

---

## 1. Dataset

This project uses two PhysioNet datasets:

### MIT-BIH Arrhythmia Database

Main ECG dataset:

https://physionet.org/content/mitdb/

The notebook downloads the required records automatically using WFDB, so the database does not need to be downloaded manually.

### MIT-BIH Noise Stress Test Database

Used only to demonstrate real ECG noise:

https://physionet.org/content/nstdb/

The pipeline downloads:

```text
bw  - baseline wander
ma  - muscle artifact
em  - electrode motion
```

These noise records are used for exploratory examples and are **not used as test data or as model robustness results**.

---

## 2. Requirements

Use Python 3 and install the required packages:

```bash
pip install wfdb numpy pandas scipy matplotlib
```

Jupyter is also required to run the notebook:

```bash
pip install jupyter
```

---

## 3. Running the project

The project is submitted as a ZIP file containing the notebook, code files, and README. After extracting the ZIP, open a terminal in the project directory.

### 1. Install dependencies

```bash
pip install wfdb numpy pandas scipy matplotlib jupyter
```





### 2. Open the notebook

```bash
jupyter notebook
```

Open:

```text
Data_Processing.ipynb
```

### 3. Run the notebook

Run all cells in order from top to bottom.

The notebook automatically downloads the required MIT-BIH and NSTDB records using WFDB and creates the processed datasets and analysis files.


# 4. Data split

The project uses a record-level split rather than randomly splitting individual heartbeats.

This is important because beats from the same ECG recording are highly correlated. Splitting individual beats randomly could allow very similar beats from the same recording to appear in both training and evaluation data.

### Development records (DS1)

The assignment provides the following records for development:
```text
101 106 108 109 112 114 115 116 118 119
122 124 201 203 205 207 208 209 215 220
223 230
```

### Test records (DS2)

```text
100 103 105 111 113 117 121 123 200 202
210 212 213 214 219 221 222 228 231 232
233 234
```

### Validation records

Four development records are reserved for validation:

```text
223 205 124 109
```

All other eligible DS1 records are used for training.

---

## 5. Strict patient separation

Records `201` and `202` belong to the same subject.

Because `202` is in the test set, the default pipeline removes `201` from the development set:

```python
STRICT_PATIENT_SPLIT = True
```

This prevents the same patient from appearing in both development and test data.

For the strict protocol:

```text
201 → excluded from development
202 → test
```

The notebook verifies that the final training, validation, and test sets do not share patients.



---

# 6. ECG preprocessing

The preprocessing pipeline is:

```text
MIT-BIH ECG
     ↓
Select MLII lead
     ↓
Read reference annotations
     ↓
Band-pass filter (0.5–40 Hz)
     ↓
Extract 150-sample beat window
     ↓
Reject invalid/boundary beats
     ↓
Per-beat z-score normalization
     ↓
Patient-independent Train / Validation / Test split
     ↓
Save prepared datasets
```

### Sampling frequency

MIT-BIH records are expected to have:

```text
360 Hz
```

The notebook checks this for every record.

### Lead selection

The pipeline uses the **MLII** ECG lead.

The MLII lead is selected by its name from each record's header rather than assuming that it is always channel 0. This is important because the channel ordering can differ between records. For example, in record 114, the channels are [V5, MLII], so MLII is located at channel index 1 rather than 0.

The selected lead information is saved to:

```text
results/lead_selection.csv
```

---

# 7. Annotation classes

MIT-BIH annotation symbols are mapped into the four required AAMI classes used in this project.

| Final class | Annotation symbols |
|---|---|
| `N` | `N`, `L`, `R`, `e`, `j` |
| `S` | `A`, `a`, `J`, `S` |
| `V` | `V`, `E` |
| `F` | `F` |

The following symbols are excluded:

```text
/
f
Q
```
These correspond to paced or unclassifiable beats that are outside the four target classes.

Other unsupported annotation symbols are retained in the audit as `OTHER` rather than being silently assigned to one of the four classes.

The complete annotation audit is saved to:

```text
results/annotation_audit.csv
```

---

# 8. Beat extraction and filtering

A fourth-order Butterworth band-pass filter is applied to ECG signal:

```text
Low cutoff:  0.5 Hz
High cutoff: 40 Hz
Order:       4
```

The implementation uses second-order sections and `sosfiltfilt`.

The filter is therefore offline and non-causal. This is suitable for the current data-preparation and experimental pipeline, but a real-time MAX78002 implementation would require a separately validated causal filtering approach.

Each annotated beat is extracted using:

```python
window_before = 60
window_after = 90
```

At 360 Hz this gives:

```text
60 samples before the annotation
1 annotation sample
89 samples after the annotation
--------------------------------
150 samples total
```

At the sampling frequency of 360 Hz, 150 samples correspond to approximately 0.417 seconds.

---

# 9. Beat quality checks

A beat is rejected when:

- the extraction window crosses the record boundary;
- the window contains non-finite values; or
- the window is effectively flat (`std < 1e-8`).

Rejected beats are recorded in:

```text
results/rejections.csv
```

Additional record-level quality information is saved in:

```text
results/quality.csv
```

The pipeline also performs data-quality checks for missing values, extreme amplitudes, and ECG lead configuration.

---

# 10. Normalization

After extracting and checking the heartbeat windows, each heartbeat is normalized separately using z-score normalization:

```text
normalized beat =
    (beat - beat mean) / beat standard deviation
```

For each 150-sample heartbeat:

The mean of that heartbeat is calculated.
The standard deviation of that heartbeat is calculated.
Each sample is then normalized using these values.

This makes the heartbeat have approximately:

mean = 0
standard deviation = 1

The normalization is done independently for each heartbeat, so no statistics from other beats are used.

The final inputs are:

```text
dtype: float32
length: 150 samples
mean: approximately 0
standard deviation: approximately 1
```

---

# 11. Data augmentation

The notebook defines three augmentation methods that can be used with training data.:

### Amplitude scaling

The heartbeat amplitude is randomly multiplied by a value between:

```text
0.9 and 1.1
```
This slightly increases or decreases the signal amplitude.

### Time shifting

The heartbeat is randomly shifted by:

```text
-10 and +10 samples
```
Zero padding is used to keep the heartbeat length at 150 samples.

### Gaussian noise

Small random Gaussian noise can be added to the heartbeat with:

```text
mean = 0
standard deviation = 0.05
```

The notebook currently demonstrates these augmentation methods. If augmentation is used during model training, it will be applied only to the training data.

---

# 12. Class weights

The four classes are imbalanced, so class weights are calculated after the train/validation/test split.

Weights are calculated using training labels only:

```python
weight =
    number_of_training_samples /
    (number_of_classes * samples_in_class)
```

This gives a higher weight to classes with fewer training samples and a lower weight to classes with more samples.

The resulting weights are saved to:

```text
results/class_weights.json
```

Validation and test labels are not used to calculate the weights.

---

# 13. Exploratory data analysis

EDA is performed using development records only.

The notebook produces:

```text
results/raw_ecg.png
results/eda_6.png
results/eda_8.png
results/eda_9.png
results/eda_10.png
```

These include examples such as:

- raw ECG from record 101;
- example beats from the four classes;
- class distribution;
- the number of beats in each record;
- ECG amplitude distribution.

DS2 is excluded from current exploratory analysis and split-selection decisions.

Note: DS2 was inspected by the original notebook before this reviewed pipeline was created. Therefore, DS2 cannot be described as a completely untouched test set historically. In the reviewed pipeline, DS2 is kept separate from the current EDA, class-weight calculation, augmentation, and split-selection decisions.

---

# 14. Noise examples

The MIT-BIH Noise Stress Test Database is used to demonstrate:

```text
Baseline wander
Muscle artifact
Electrode motion
```

Examples are generated at:

```text
12 dB
6 dB
0 dB
```

and saved as:

```text
results/noise_examples.png
```

These are illustrative EDA examples and should not be interpreted as model robustness measurements.

---

# 15. Output files

After running the notebook, the main outputs are:

```text
results/
├── train.npz
├── validation.npz
├── test.npz
├── class_weights.json
├── manifest.json
├── split_statistics.csv
├── lead_selection.csv
├── annotation_audit.csv
├── quality.csv
├── rejections.csv
└── *.png
```

### Processed datasets

Each `.npz` file contains:

```text
X           ECG input windows
y           class labels
record_ids  source MIT-BIH record
samples     original annotation sample positions
```

For example:

```python
import numpy as np

with np.load("results/train.npz", allow_pickle=False) as data:
    X_train = data["X"]
    y_train = data["y"]
```

The input shape is:

```text
(number_of_beats, 150)
```

### `manifest.json`

This records the main reproducibility information, including:

- random seed;
- Python and package versions;
- sampling frequency;
- heartbeat window size;
- filter settings;
- normalization method;
- strict patient-split setting;
- exact train/validation/test records.

### `split_statistics.csv`

This file contains statistics for the training, validation, and test splits, including the number of beats, classes, records, and subjects.

---

# 16. Reproducibility checks

The final notebook section checks that:

- every split contains 150-sample windows;
- all data are `float32`;
- there are no NaN/Inf values;
- each beat is normalized;
- all four classes are represented;
- every declared record contributes data;
- saved `.npz` files match the arrays in memory;
- records do not overlap between splits;
- patients do not overlap between splits in strict mode;
- class weights are calculated only from training labels;
- exported split statistics match the actual data.

A successful run should finish with:

```text
All final checks passed.
```

---

# 17. Reproducibility notes

For the best reproduction:

1. Use the notebook in this repository as the implementation source.
2. Run all cells in order from a clean environment.
3. Keep the default:
   ```python
   SEED = 42
   ```
4. Keep:
   ```python
   STRICT_PATIENT_SPLIT = True
   ```
5. Do not manually change the record lists.
6. Keep DS2 separate from the current EDA, model development, and parameter selection.
7. Keep `results/manifest.json` with the generated datasets.

The notebook records the actual installed versions of:

```text
Python
wfdb
numpy
pandas
scipy
matplotlib
```

so that the execution environment can be documented.

---

# 18. Important limitations

This pipeline starts from **reference beat annotations** provided by MIT-BIH. It does not perform automatic R-peak detection.

The filtering uses `sosfiltfilt`, which is non-causal. Therefore, this preprocessing should not be considered a complete real-time ECG processing system.

The 150-sample window may not contain the complete P/T-wave morphology for every heartbeat. The suitability of this window should therefore be evaluated during the subsequent modelling stage.

---

# 19. Data citation

Please cite the relevant PhysioNet/MIT-BIH publications when using these datasets.

MIT-BIH Arrhythmia Database:

https://physionet.org/content/mitdb/

MIT-BIH Noise Stress Test Database:

https://physionet.org/content/nstdb/

---

# 20. Project structure

After extracting the submitted ZIP, the project should have a structure similar to:

```text
project/
│
├── Data_Processing_2nd_edition.ipynb
├── README.md
├── requirements.txt       # if provided
├
│
├── mitdb/                 # downloaded/generated when the notebook runs
├── nstdb/                 # downloaded/generated when the notebook runs
│
└── results/               # generated outputs
```

The `mitdb/`, `nstdb/`, and `results/` directories may be created or populated when the notebook is executed. They do not need to be manually downloaded separately unless they are included in the submitted ZIP.

**In short:** Extract the submitted ZIP. Install the required Python packages. Open Data_Processing_Reviewed.ipynb. Run all notebook cells in order.

The notebook downloads the required MIT-BIH data, prepares the ECG heartbeat data, and generates the processed training, validation, and test datasets together with the reproducibility and quality-control files.