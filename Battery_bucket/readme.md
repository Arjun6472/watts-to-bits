# 🔋 Battery Bucket & Grid Storage Simulator (Practice Repo: v1 → v3)

A progressive, three-stage Python practice series built to understand how grid-scale Battery Energy Storage Systems (BESS) track energy, respond to power-grid stress, and use Machine Learning to protect battery lifespan.

---

## 🗺️ Progression Overview

| Version | Core Concept | What Was Added | Dependencies |
| :--- | :--- | :--- | :--- |
| **`v1`** | Electro-Thermal Physics | Time-step loop tracking State of Charge (`SoC`), C-rate, I²R Joule heating, and Newton cooling. | Standard Python (`math` / built-ins) |
| **`v2`** | Grid Control Logic | Dynamic power dispatch (`get_battery_power`) for **Peak Shaving** (> 80 MW) and **50 Hz Frequency Droop**. | Standard Python |
| **`v3`** | AI Predictive BMS | Supervised `RandomForestRegressor` predicting health loss (`SoH`) + power throttling (`ai_safety_check`). | `numpy`, `scikit-learn` |

---

## 📘 Version 1: Simple Single Battery Bucket (`v1`)

### 🎯 Learning Goal
Understand how a battery's charge and internal temperature evolve over discrete time steps (Δ t = 0.25 hours / 15 minutes) under a constant power load.

### ⚙️ Core Equations Used
1. **Energy Drain:**
   \[\text{Energy Used (MWh)} = \text{Power (MW)} \times \Delta t\text{ (hr)}\]
   \[\text{SoC (\%)} = \left(\frac{\text{Stored MWh}}{\text{Capacity MWh}}\right) \times 100\]
2. **C-Rate (Relative Stress):**
   \[\text{C-Rate} = \frac{\text{Discharge Power (MW)}}{\text{Total Capacity (MWh)}}\]
   *(Example: 20 MW on a 100 MWh battery = 0.2C, meaning it takes 5 hours to empty).*
3. **Thermal Balance (I²R Heating vs. Cooling):**
   \[\text{Heat Added} = 15.0 \times (\text{C-Rate})^2 \times \Delta t\]
   \[\text{Heat Lost} = 0.3 \times (T_{\text{cell}} - 25.0) \times \Delta t\]
   \[T_{\text{new}} = T_{\text{old}} + \text{Heat Added} - \text{Heat Lost}\]

### 🔑 Key Revision Takeaways
* Doubling discharge power quadruples heat generation because heating scales with (C-Rate)².
* Cooling is zero on step 1 if \(T_{\text{cell}} = 25^\circ\text{C}\), and increases as the temperature gap above ambient (25°C) widens.
* `max(0.0, ...)` prevents stored energy from dropping into impossible negative values.

---

## ⚡ Version 2: Grid-Aware Battery Controller (`v2`)

### 🎯 Learning Goal
Replace fixed constant discharge with an automated decision function (`get_battery_power`) that reacts to live electrical grid conditions while respecting hardware limits.

### 🧠 Decision Logic Rules
1. **Peak Shaving (Overload Relief):**
   * **Condition:** `city_load > 80 MW` AND `SoC > 15%`
   * **Action:** Discharge `city_load - 80` to keep grid line load capped at 80 MW.
2. **Primary Frequency Regulation (50 Hz Droop Support):**
   * **Condition:** `grid_freq < 49.98 Hz` (outside ± 0.02 Hz deadband) AND `SoC > 15%`
   * **Action:** Calculate drop = 50.0 - grid\_freq and add drop × 12.5 MW to help stabilize grid frequency.
3. **Inverter Hardware Cap:**
   * `min(25.0, power)` ensures combined power requests never exceed the inverter's 25 MW rating.

### 🔑 Key Revision Takeaways
* **Value Stacking:** Peak shaving and frequency boost add together (`+=`) when both emergencies happen simultaneously.
* **Deadband & SoC Protection:** Ignoring tiny noise above 49.98 Hz and stopping discharge below 15% SoC prevents unnecessary battery cycling and deep-discharge damage.

---

## 🤖 Version 3: AI-Enhanced Predictive BMS (`v3`)

### 🎯 Learning Goal
Add a predictive Machine Learning layer (`RandomForestRegressor`) that forecasts State of Health (`SoH`) degradation *before* executing a grid power request, throttling output if wear or temperature becomes unsafe.

### 🔄 Two-Stage Execution Flow
```text
[Grid Inputs] ──▶ v2: get_battery_power() ──▶ requested_mw
                                                   │
                                                   ▼
[Temp & SoC]  ──▶ v3: ai_safety_check()   ──▶ actual_mw (100% or -30% throttled)
                           │
                           ▼
                  Update Energy, SoH & Temp
```

### 🧪 How the ML Model Works
* **Training Features (`X_train`):** 1,000 synthetic historical records of `[c_rate, temp_c, soc_percent]`.
* **Target Label (`y_train`):** Physics-informed wear formula combining C-rate stress and Arrhenius exponential heat aging:
  $$y_{\text{train}} = (\text{C-Rate})^{1.5} \times e^{0.05 \times (T_{\text{cell}} - 25.0)} \times 0.01$$
* **Safety Intervention (`ai_safety_check`):**
  * Predicts wear for the proposed C-rate using `ml_wear_model.predict(...).item()`.
  * If `predicted_wear > 0.0025` **OR** `temp_c > 35.0°C`, cuts `requested_mw` by **30%** (`requested_mw * 0.70`) and logs `"AI THROTTLED (-30%)"`.

### 🔑 Key Revision Takeaways
* **`.item()` Extraction:** `scikit-learn`'s `.predict()` returns a 1D NumPy array; `.item()` converts it cleanly into a Python scalar `float` for comparisons and string formatting.
* **Closed-Loop Protection:** Energy drain, `SoH` loss, and $I^2R$ heating are all updated using `actual_mw` (post-throttle) rather than `requested_mw`.

---

## 🛠️ Quick Setup & Run Notes
```bash
# Install dependencies needed for v3
pip install numpy scikit-learn

# Run any version
python3 battery_bucket_v1.py
python3 battery_bucket_v2.py
python3 battery_bucket_v3.py
```

### 💡 Quick Experiments for Future Revision
* Set `temp_c = 36.0` at the start of `v3` to watch `AI THROTTLED (-30%)` trigger immediately on Step 2.
* Lower cooling coefficient (`0.3` $\rightarrow$ `0.05`) in `v2` or `v3` to observe thermal runaway during back-to-back peak events.

