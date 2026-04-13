# 📦 Smart Resource Allocation Tool

> AI-powered logistics and distribution intelligence for Indian retail.  
> Runs on **Google Colab** or your **local machine** — no server required.

---

## Table of Contents

1. [What This Tool Does](#1-what-this-tool-does)
2. [How It Works — Module by Module](#2-how-it-works--module-by-module)
3. [Alert System](#3-alert-system)
4. [Dashboard & Outputs](#4-dashboard--outputs)
5. [CSV File Format](#5-csv-file-format)
6. [Running on Google Colab](#6-running-on-google-colab)
7. [Running on Your Local Machine](#7-running-on-your-local-machine)
8. [Customisation Reference](#8-customisation-reference)
9. [Troubleshooting](#9-troubleshooting)

---

## 1. What This Tool Does

The Smart Resource Allocation Tool reads your sales data (as a CSV), applies machine learning to understand demand patterns, and tells you exactly how much stock each state needs for each product — and which areas are healthy, under-supplied, or over-stocked.

**Core capabilities:**

- **AI demand scoring** — automatically works out which states have high or low demand for each product (e.g. Veshti is high in Tamil Nadu, low in Punjab)
- **Smart stock allocation** — distributes your total available supply proportionally across states based on their demand, not equally
- **Four-level alert system** — 🔴 Critical, 🟡 Warning, 🔵 Info (excess), ✅ Safe (healthy)
- **Interactive dashboard** — zoomable treemap, bar charts, and a colour-coded live alert board
- **Improvement suggestions** — automatically identifies where stock should be redirected
- **Export** — saves everything to Excel and CSV for further use

---

## 2. How It Works — Module by Module

The script runs nine steps in sequence every time you execute it.

---

### Module 1 — Package Installer

```python
for pkg in ["pandas", "numpy", "scikit-learn", "matplotlib",
            "seaborn", "plotly", "ipywidgets", "openpyxl"]:
    install(pkg)
```

Automatically installs all required Python libraries if they are not already present. On Colab this runs silently. On a local machine, you install these once manually (see [Section 7](#7-running-on-your-local-machine)).

---

### Module 2 — Sample Data Generator (`generate_sample_data`)

Builds a realistic 900-row dataset of Indian textile sales across 15 states, 5 products, and 12 months — so you can test the tool immediately without a CSV.

- **Products:** Veshti, Saree, Dhoti, Salwar Kameez, Kurta
- **States:** Tamil Nadu, Kerala, Karnataka, Andhra Pradesh, Telangana, Uttar Pradesh, Rajasthan, Punjab, Haryana, Delhi, Maharashtra, Gujarat, West Bengal, Odisha, Assam
- **Regional demand logic:** Veshti demand is 3.5× higher in Tamil Nadu than in Punjab (0.2×). Salwar Kameez is 3.5× in Punjab, 0.5× in Kerala.
- **Seasonal bumps:** October and November get a 1.5× festive multiplier. January and April get 1.2×.

Saves the data as `sample_sales_data.csv` in the working directory.

**To use your own data instead:** set `USE_SAMPLE = False` at the bottom of the script.

---

### Module 3 — Data Loader (`load_data`)

```python
df_raw = load_data(use_sample=USE_SAMPLE)
```

Two modes:

| `USE_SAMPLE` value | Behaviour |
|---|---|
| `True` | Generates and loads the built-in demo dataset |
| `False` | On Colab: shows a file-picker button. On local: reads `your_data.csv` from disk (see local setup) |

---

### Module 4 — Preprocessor (`preprocess`)

Cleans and prepares the raw data:

- Strips spaces from column names and normalises them (e.g. `"units sold"` → `"Units_Sold"`)
- Validates that all required columns are present — raises a clear error if anything is missing
- Converts numeric columns, filling blanks with 0
- Calculates three new columns for every row:
  - `Demand_Gap` = Units Sold − Supply Available (positive = shortage)
  - `Gap_Pct` = (Demand Gap ÷ Units Sold) × 100
  - `Shortage_Flag` = True if demand exceeds supply

---

### Module 5 — AI Demand Analysis (`run_ai_analysis`)

This is the core intelligence of the tool. It runs in four sub-steps:

**5a. Aggregation**

Groups the row-level data by State + Product, computing:
- `Total_Sold` — total units sold across all months
- `Total_Supply` — total stock available
- `Total_Revenue` — total revenue
- `Avg_Gap_Pct` — average supply gap percentage
- `Demand_Index` — a 0–100 score showing how high demand for this product is in this state compared to all other states (100 = the highest-demand state for that product)
- `Supply_Ratio` — supply ÷ demand (values below 1.0 mean shortage)

**5b. K-Means Clustering (scikit-learn)**

Every state-product pair is plotted in a 3D feature space using Demand Index, Supply Ratio, and Gap%. K-Means groups them into 4 clusters which are then labelled:

| Cluster | Label | Meaning |
|---|---|---|
| 🔴 | High Demand / Under-supplied | Urgent restock needed |
| 🟢 | High Demand / Well-supplied | Maintain current levels |
| 🟡 | Low Demand / Over-supplied | Redirect excess stock |
| 🔵 | Moderate Demand / Balanced | Monitor periodically |

**5c. Demand Score**

If your CSV includes a `Demand_Score` column, those values are used directly. If not, the tool uses the normalised Demand Index as a proxy.

**5d. Smart Stock Allocation**

For each product, the total available supply is divided among states using this formula:

```
Recommended Stock (State) =
    (Demand Index of State ÷ Total Demand Index of All States)
    × Total Available Supply for that Product
```

A state with Demand Index 70 gets 7× more stock than a state with Demand Index 10. This is proportional, fair, and demand-driven — not a flat equal split.

Also calculates `Stock_vs_Current` = how much more (positive) or less (negative) each state should hold compared to current supply.

---

### Module 6 — Alert Engine (`generate_alerts`)

Scans every state-product pair and assigns one of four alert levels:

| Level | Trigger condition | Row colour |
|---|---|---|
| 🔴 **CRITICAL** | Gap% > 30% | Red |
| 🟡 **WARNING** | Gap% between 10–30% | Yellow |
| 🔵 **INFO** | Supply Ratio > 1.5× demand (over-stocked) | Blue |
| ✅ **SAFE** | Gap% < 10% and supply is healthy | Green |

Every pair gets a message explaining the exact situation and what to do, e.g.:

> *"Tamil Nadu needs Veshti urgently — 34.2% demand unmet. Recommend restocking by 1,204 units."*

> *"Punjab: Veshti is well-supplied. Gap is only 3.1% — no action needed."*

**To change the thresholds**, find this section in the script and edit the numbers:

```python
ALERT_THRESHOLDS = {
    "CRITICAL": 30,   # ← change to 20 for earlier critical warnings
    "WARNING":  10,   # ← change to 5 for more warnings
}
```

---

### Module 7 — Improvement Engine (`generate_improvements`)

Analyses alerts to generate ranked, actionable improvement suggestions:

- **🔴 High priority** — critical shortage states that need immediate stock increases, ranked by revenue impact
- **🟡 Medium priority** — over-stocked states where excess inventory should be redirected
- **🟠 Medium-high priority** — entire regions with systematic supply gaps (e.g. the entire South having a Veshti shortfall)

---

### Module 8 — Static Dashboard (`plot_dashboard`)

Renders and saves six matplotlib/seaborn charts as PNG files:

| Chart | Filename | What it shows |
|---|---|---|
| Demand heatmap | `demand_heatmap.png` | State × Product demand index. Dark red = highest demand |
| Allocation bars | `allocation_<Product>.png` | Recommended stock per state for each product. Red bars = critical shortages |
| Revenue by state | `revenue_by_state.png` | Top 10 states by total revenue |
| Supply vs Demand scatter | `supply_demand_scatter.png` | K-Means cluster visualisation |
| Alert severity pie | `alert_distribution.png` | Proportion of 🔴 Critical / 🟡 Warning / ✅ Safe / 🔵 Info |
| Region stats | `region_stats.png` | Revenue and gap% broken down by North / South / East / West |

---

### Module 9 — Interactive Dashboard (`plot_interactive`)

Renders three interactive Plotly charts inside Colab (or in your browser when running locally):

- **Revenue & Gap Treemap** — zoomable, click Region → State → Product. Red tint = supply gap, green = healthy
- **Recommended Stock Bar Chart** — grouped by product, hover for exact numbers
- **Live Alert Board** — colour-coded table with Level, State, Product, Gap%, Revenue, and the specific suggestion for each row

---

### Module 10 — Console Summary (`print_summary`)

Prints a formatted executive summary to the terminal:

```
=================================================================
  📋  SMART LOGISTICS SYSTEM — EXECUTIVE SUMMARY
=================================================================
  Total Revenue       :  ₹1,234,567,890.00
  Total Units Sold    :       2,345,678
  Total Supply        :       1,987,654
  Overall Demand Gap  :          15.3 %
  States Covered      :              15
  Products Tracked    :               5
=================================================================

  🔴  CRITICAL ALERTS (Immediate Action Required)
  ...

  🟡  WARNINGS
  ...

  ✅  SAFE (Well-supplied, No Action Needed)
  ...

  🏆  TOP 5 STATES BY REVENUE
  ...

  🔧  AREAS OF IMPROVEMENT
  ...
```

---

### Module 11 — Export (`export_results`)

Saves all results to files in the working directory:

| File | Contents |
|---|---|
| `logistics_report.xlsx` | Three sheets: Allocation, Alerts, Improvements |
| `allocation_report.csv` | Full state × product table with all AI scores and recommendations |
| `alerts.csv` | Every alert with level, gap%, revenue, and suggestion |
| `improvements.csv` | Ranked improvement opportunities |

---

## 3. Alert System

Every state-product combination receives exactly one status every time you run the tool.

```
Gap% > 30%              →  🔴 CRITICAL   (red row)
Gap% between 10–30%     →  🟡 WARNING    (yellow row)
Supply Ratio > 1.5×     →  🔵 INFO       (blue row — excess stock)
Everything else         →  ✅ SAFE        (green row — healthy)
```

**Reading the alert pie chart:**

| What you see | What it means | What to do |
|---|---|---|
| All green (✅ SAFE) | Supply meets demand across the board | Nothing urgent — monitor monthly |
| Mix of green and yellow | Most states are fine; some need attention | Check `alerts.csv` for warning details |
| Any red slice | At least one state has a serious shortage | Act immediately — open `alerts.csv`, sort by Revenue, expedite highest-revenue critical shipments first |
| Large blue slice | Excess stock is being held in low-demand areas | Redirect that inventory to states showing warnings or critical alerts |

---

## 4. Dashboard & Outputs

All charts are saved as PNG files regardless of whether you run on Colab or locally. The Plotly interactive charts open in your browser automatically when running locally.

After running, your working directory will contain:

```
smart_logistics_tool.py        ← the script
sample_sales_data.csv          ← auto-generated demo data (if USE_SAMPLE = True)
logistics_report.xlsx          ← main Excel report
allocation_report.csv
alerts.csv
improvements.csv
demand_heatmap.png
allocation_Veshti.png
allocation_Saree.png
allocation_Dhoti.png
allocation_Salwar_Kameez.png
allocation_Kurta.png
revenue_by_state.png
supply_demand_scatter.png
alert_distribution.png
region_stats.png
```

---

## 5. CSV File Format

Your CSV file must have these columns. Names are not case-sensitive and underscores vs spaces are handled automatically.

| Column | Required | Description | Example |
|---|---|---|---|
| `State` | ✅ Yes | State name | `Tamil Nadu` |
| `Product` | ✅ Yes | Product name | `Veshti` |
| `Units_Sold` | ✅ Yes | Units sold in that row | `3500` |
| `Supply_Available` | ✅ Yes | Units currently in stock | `2800` |
| `Revenue` | ✅ Yes | Revenue for that row (₹) | `1575000` |
| `Region` | Optional | Geographic grouping | `South` |
| `Month` | Optional | Month name for trend analysis | `October` |
| `Demand_Score` | Optional | Manual demand weight (see below) | `3.5` |

### Setting Demand Score

The `Demand_Score` column lets you manually override the AI's automatic demand weighting. Add it as a plain number column.

| Score | Meaning | Example |
|---|---|---|
| 3.0 – 4.0 | Very high demand — core market | Veshti in Tamil Nadu → 3.5 |
| 1.5 – 3.0 | Moderate demand | Kurta in Maharashtra → 1.8 |
| 0.5 – 1.5 | Low demand | Salwar Kameez in Kerala → 0.6 |
| < 0.5 | Negligible — minimal stock needed | Veshti in Punjab → 0.2 |

If you leave out the `Demand_Score` column entirely, the tool calculates it automatically from sales volume.

### Minimal valid CSV example

```csv
State,Product,Units_Sold,Supply_Available,Revenue
Tamil Nadu,Veshti,3500,2800,1575000
Punjab,Veshti,200,400,90000
Tamil Nadu,Salwar Kameez,600,700,390000
Punjab,Salwar Kameez,3200,2900,2080000
```

---

## 6. Running on Google Colab

1. Go to [colab.research.google.com](https://colab.research.google.com) and create a new notebook
2. Paste the entire `smart_logistics_tool.py` file into a single cell
3. To use the demo data: leave `USE_SAMPLE = True` and press **Shift + Enter**
4. To use your own CSV: change `USE_SAMPLE = False`, then run — a file-picker button will appear in the cell output. Click it and select your file
5. All output files appear in the **📁 Files panel** on the left sidebar. Right-click any file to download it

> **Note:** Colab sessions are temporary. Download your output files before closing the tab — they will be deleted when the session ends.

---

## 7. Running on Your Local Machine

### Step 1 — Check Python is installed

Open a terminal (Command Prompt on Windows, Terminal on Mac/Linux) and type:

```bash
python --version
```

You need Python 3.8 or newer. If you don't have it, download it from [python.org](https://python.org).

---

### Step 2 — Install the required libraries

Run this once in your terminal:

```bash
pip install pandas numpy scikit-learn matplotlib seaborn plotly ipywidgets openpyxl
```

Or if you use `pip3`:

```bash
pip3 install pandas numpy scikit-learn matplotlib seaborn plotly ipywidgets openpyxl
```

---

### Step 3 — Modify the script for local use

The script uses `from google.colab import files` which only works on Colab. For local use, **replace these two lines** near the top of the file:

**Find and remove:**
```python
from google.colab import files
```

**Also find the `load_data` function and replace it with this local version:**

```python
def load_data(use_sample=True):
    if use_sample:
        print("🔄 Using built-in sample data…")
        return generate_sample_data()
    else:
        # ── LOCAL VERSION: read from a file path ──
        csv_path = "your_data.csv"   # ← change this to your actual filename
        df = pd.read_csv(csv_path)
        print(f"✅ Loaded: {csv_path}  |  Shape: {df.shape}")
        print("   Columns:", list(df.columns))
        return df
```

Replace `"your_data.csv"` with the actual path to your file, e.g.:
- Windows: `"C:\\Users\\YourName\\Desktop\\sales_data.csv"`
- Mac/Linux: `"/Users/yourname/Downloads/sales_data.csv"`
- Same folder as the script: just `"sales_data.csv"`

---

### Step 4 — Place your files in a folder

Create a folder anywhere on your computer, for example `smart_logistics/`. Put these two files in it:

```
smart_logistics/
├── smart_logistics_tool.py    ← the modified script
└── sales_data.csv             ← your CSV (or leave this out if using sample data)
```

---

### Step 5 — Run the script

Open a terminal, navigate to your folder, and run:

```bash
cd smart_logistics
python smart_logistics_tool.py
```

The script will:
1. Print progress updates to the terminal
2. Display static charts in pop-up windows (close each one to continue)
3. Open Plotly interactive charts in your default web browser automatically
4. Print the executive summary to the terminal
5. Save all output files into the same `smart_logistics/` folder

---

### Step 6 — Find your output files

After the script finishes, your folder will contain all the PNG charts, the Excel report, and the CSV exports:

```
smart_logistics/
├── smart_logistics_tool.py
├── sales_data.csv
├── logistics_report.xlsx       ← open in Excel or Google Sheets
├── allocation_report.csv
├── alerts.csv
├── improvements.csv
├── demand_heatmap.png
├── allocation_Veshti.png
├── revenue_by_state.png
├── supply_demand_scatter.png
├── alert_distribution.png
└── region_stats.png
```

---

### Optional: Run in a virtual environment (recommended)

A virtual environment keeps this project's packages separate from the rest of your computer.

```bash
# Create the environment (do this once)
python -m venv venv

# Activate it
# On Windows:
venv\Scripts\activate
# On Mac/Linux:
source venv/bin/activate

# Install packages inside the environment
pip install pandas numpy scikit-learn matplotlib seaborn plotly ipywidgets openpyxl

# Run the script
python smart_logistics_tool.py

# When you're done, deactivate
deactivate
```

---

### Optional: Run as a Jupyter Notebook locally

If you prefer a notebook environment on your own machine:

```bash
pip install jupyter
jupyter notebook
```

Then create a new notebook, paste the script into a cell, and run it — identical to Colab but fully offline.

---

## 8. Customisation Reference

| What to change | Where in the script | How |
|---|---|---|
| Alert thresholds | `ALERT_THRESHOLDS` dict | Change `30` (critical) and `10` (warning) to your preferred gap% values |
| Number of ML clusters | `KMeans(n_clusters=4)` | Increase to 5–6 for more granular segmentation on large datasets |
| Top revenue chart count | `nlargest(10)` in `plot_dashboard` | Change `10` to any number |
| Use sample vs real data | `USE_SAMPLE = True/False` at the bottom | Set to `False` to load your own CSV |
| Add products | Your CSV | Add rows with new product names — no code changes needed |
| Add states | Your CSV | Add rows with new state names — no code changes needed |
| Improvement suggestion count | `head(15)` in `generate_improvements` | Change `15` to show more or fewer suggestions |

---

## 9. Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| `ModuleNotFoundError: No module named 'google.colab'` | Running locally with the Colab import still in the script | Replace the `load_data` function as shown in Step 3 of local setup |
| `Missing required columns: [...]` | Your CSV column names don't match | Rename your columns to exactly: `State`, `Product`, `Units_Sold`, `Supply_Available`, `Revenue` |
| All alerts are WARNING (100% yellow) | No state-product pairs have a gap above 30% or below 10% | Lower `CRITICAL` to 20% or `WARNING` to 5% in `ALERT_THRESHOLDS` |
| No alerts at all | All gaps are below the warning threshold | Lower `WARNING` threshold, or verify your supply data isn't artificially inflated |
| Charts not showing | On local: chart window may be behind other windows | Look in your taskbar — matplotlib opens separate windows |
| Plotly chart not showing | On Colab: renderer issue | Add `fig.show(renderer='colab')` to any `fig.show()` call |
| Excel file not saved | openpyxl not installed | Run `pip install openpyxl` then re-run the script |
| `KeyError` on a column | Special characters or extra spaces in CSV header | Open your CSV in a text editor and check the first line for hidden characters |
| Script runs but all gaps are 0% | Supply_Available column equals Units_Sold in all rows | Verify your CSV is recording actual stock figures, not copying from units sold |

---

## Licence & Data Privacy

This tool runs entirely on your own machine or within your own Colab session. No data is sent to any external server. All machine learning computation is performed locally using scikit-learn.

If running on Google Colab, your data is processed within Google's infrastructure under your Google account. Download all output files before closing the session — Colab does not retain files between sessions.
