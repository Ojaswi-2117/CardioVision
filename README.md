# CardioVision AI - AI Coronary Digital Twin
Multimodal AI Hackathon 2026 (IIT Mandi) - Track A: cardiovascular risk visualization and prediction.

Clinical data (demographic, examination, ECG, laboratory, echo) -> calibrated ML models -> overall CAD risk + LAD / LCX / RCA stenosis risk -> SHAP explanation -> interactive 3D coronary tree with live blood flow.
> **For educational and decision-support purposes only. Not a clinical diagnosis and not a substitute for formal diagnostic imaging.** Colour and flow show vessel-level *predicted risk*, never a detected lesion location.

## Run it (one command on Windows)
Open **cmd** and paste (after downloading `cardiovision.zip` to your Downloads folder):
```
cd "%USERPROFILE%\Downloads" && tar -xf cardiovision.zip && cd cardiovision && run.bat
```
`run.bat` finds (or installs, via winget) Python 3.12, creates a private environment, installs the pinned packages, starts the server and opens http://localhost:7860. If winget is unavailable, install Python 3.12 from python.org (tick "Add to PATH") and run `run.bat` again.
* macOS / Linux: `bash run.sh`   |   manual: `pip install -r requirements.txt` then `python -m uvicorn app:app --port 7860`. No GPU needed.

The first start needs ~20 s while the SHAP explainer warms up. Everything (including three.js) is served locally - no internet needed after install.

## Interactive presentation (projector-ready)
Run `present.bat` (Windows) or `bash present.sh`: it starts the app and opens http://localhost:7860/present.html. 17 clean slides with interactive widgets (leakage switch, clickable workflow, coherence and conformal sliders, risk-to-flow demo, results tabs, modality slider) and a **live demo slide** that drives the real app with step buttons or keys 1-8. Keys: arrows = next/previous, F = fullscreen, O = slide overview, 1-8 = live-demo steps.

## Retrain from the dataset
Download *Extension of Z-Alizadeh Sani Dataset* (UCI id 411) and run `python train.py --data "<file>.xlsx"` (or without `--data` to fetch via `ucimlrepo`). This rewrites `artifacts/model.joblib` and `artifacts/metrics.json`; then run `python analyze_extra.py --data "<file>.xlsx"` to add the modality-ablation and decision-curve results. Seeds are fixed.

## What is in the pipeline
* Leakage control: `LAD`, `LCX`, `RCA`, `Cath` are never model inputs (a leakage audit is reported).
* Models per target (CAD, LAD, LCX, RCA): Logistic Regression / Random Forest / Gradient Boosting, chosen by repeated stratified 5x3 CV; sigmoid calibration; 20 % untouched test set with bootstrap 95 % CIs; metrics: accuracy, precision, recall, F1, ROC-AUC, Brier (`artifacts/metrics.json`).
* **Coherent CAD risk:** CAD means "any vessel stenotic", so P(CAD) is constrained between max and sum of the vessel probabilities (the raw model violated this for ~6 % of test patients).
* **Conformal uncertainty:** 90 %-coverage prediction sets; "uncertain" vessels pulse in the 3D view.
* **Evidence a reviewer can check (Trust panel, `analyze_extra.py`):** modality ablation (cross-validated AUC as demographic -> symptoms/exam -> ECG -> lab/echo data are added), decision-curve analysis, calibration plot, leakage audit.
* **Out-of-range warning** when an input lies outside the training data, and a **printable PDF report** (risk, 3D snapshot, main factors, disclaimer).
* **Territories:** the heart shell is tinted by each artery's approximate supply territory (geometric proximity, `tools/make_territory.py`) and by that artery's predicted risk - not a lesion map.
* SHAP (permutation explainer on the raw clinical features) with the patient's value shown beside each contribution, plus group totals (demographic / symptoms / ECG / lab & echo).
* **Test for new patients** form + **Risk simulation** (counterfactual what-if on modifiable measurements - not a treatment recommendation).
* 3D: cinematic anatomy intro, X-ray torso, realistic heart shell, LAD (amber) / LCX (violet) / RCA (cyan) with baked per-vertex flow distance so blood-flow packets follow the real artery geometry; speed, glow and heat rise with predicted risk; rotate / zoom / pan, heart-size slider, XYZ gizmo (click an axis to snap the view), click an artery to focus.

## Architecture
`train.py` -> `artifacts/` -> `app.py` (FastAPI: `/api/meta`, `/api/predict`, `/api/explain`) -> `static/index.html` (three.js). New clinical features or models: retrain, the form/sliders/SHAP are generated from `artifacts` automatically. New anatomy: add a named node to the GLB and extend `tools/make_flow.py`.

## Limitations
303 patients from a single centre, no external validation; held-out test set is only 61 patients so intervals are wide. Vessel-level AUCs are modest. The coronary tree and heart shell come from different models and are fitted approximately; the ribcage is procedural; the vessel grouping into LAD/LCX/RCA follows anatomical part names and has not been reviewed by a clinician. Left main stem and posterior interventricular artery are kept neutral (dominance-dependent).

## Credits and licences
* Dataset: Alizadehsani et al., Z-Alizadeh Sani / Extension of Z-Alizadeh Sani, UCI Machine Learning Repository.
* Coronary arteries: BodyParts3D, (c) Database Center for Life Science, CC BY-SA 2.1 Japan.
* Body base mesh ("Anatomy Study 2.0 basemesh human male body") and heart mesh ("Realistic human heart"): **add author and licence from the source pages here**. Both were decimated / re-oriented for this project.
* three.js (MIT).
