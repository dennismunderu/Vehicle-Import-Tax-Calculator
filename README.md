# Kenyan Vehicle Import Tax Calculator (CRSP-Based) 🚗💰

<p align="center">
  <img src="data/project_banner/KRA Import Tax Calculator.png" alt="Import Tax Calculator" width="75%"/>
</p>

[![Python](https://img.shields.io/badge/Python-3.9+-blue?logo=python&logoColor=white)](https://www.python.org/)
[![Pandas](https://img.shields.io/badge/Pandas-2.0+-blue?logo=pandas&logoColor=white)](https://pandas.pydata.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)


**A practical data cleaning and business-rules project that transforms the messy official KRA CRSP dataset into an accurate vehicle import tax calculator for Kenya.**

Importing vehicles into Kenya is a high-risk financial decision for individuals and small businesses due to the country’s complex vehicle taxation framework. Import taxes often range between 80% and 150% of the Cost and Freight (C&F) value, and many importers only discover the true tax burden after the vehicle has already arrived at the port.

This project delivers a **Python-based tax calculator** built on the official Kenya Revenue Authority (KRA) **Current Retail Selling Price (CRSP)** list. Users can estimate total import taxes **before purchase** using just three inputs:
- Vehicle **Make**
- Vehicle **Model**
- **Year of Manufacture**

The core focus addressed in this project is **real-world data cleaning** — turning a highly inconsistent government dataset into a dependable, decision-ready system that reflects actual customs calculations.

---

## Problem Statement

Kenyan vehicle import taxes depend on multiple interdependent factors:
- Government-assigned CRSP value
- Engine capacity thresholds
- Fuel type
- Vehicle age (depreciation rules)

The official CRSP list is publicly available but comes as a poorly structured CSV with significant data quality issues, making manual calculations unreliable.

This project solves that by cleaning the data and implementing **exact KRA tax rules** into a transparent, reusable calculator.

---

## Data Source

- **Source**: Official KRA CRSP List (~5,279 rows)
- **Link**: [KRA Vehicle Import Guidelines & CRSP](https://www.kra.go.ke/component/search/?searchword=CRSP%202025&searchphrase=all&Itemid=0)
- **Key Columns**: Make, Model, Engine Capacity, Fuel, Body Type, Transmission, Drive Configuration, CRSP (KES)

**Major data quality challenges**:
- Inconsistent naming
- Mixed units (cc, Hp, kWh)
- Swapped column values
- Invalid categorical entries
- Non-numeric values in numeric fields

---

## Data Cleaning and Preparation

### 1. Header and Structure Normalization

The raw CSV had column names embedded in the first row and no proper header. I extracted and standardized these to create a valid tabular structure, and also dropped irrelevant columns.

```python
df.columns = df.iloc[0]               # Set first row as columns
df = df[1:]                           # Remove the first row from the data
df = df.reset_index(drop=True)        # Reset index for clean sequencing
```

### 2. Make Column Standardization

Vehicle makes contained trailing spaces, non-breaking characters(\xa0), and inconsistent branding variations (e.g., MERCEDES vs MERCEDES-BENZ). I normalized the column to ensure accurate grouping and lookups.

```python
df.loc[:, 'Make'] = (df['Make']
                    .str.strip()
                    .str.replace('\xa0', '')
                    .str.replace('-BENZ', '')
                    .str.replace('-', ' ')
                    .str.capitalize())

df['Make'].unique()  # Verify cleanup
```

### 3. Transmission and Drive Configuration Corrections

Significant data entry errors caused drivetrain values (e.g., 4WD, AWD) to appear in the transmission column and vice versa. Boolean masking and column-level corrections were applied, followed by standardization of transmission categories.

```python
boolean = df['Transmission'] == '4WD'
df.loc[boolean, ['Transmission', 'Drive_configuration']] = (
    df.loc[boolean, ['Drive_configuration', 'Transmission']].values
)

df[boolean]  # Inspect corrected rows
```

Similar fixes were applied for AWD and 2WD. Drive configurations were then standardized into: 2WD, 4WD, AWD, MULTI.

### 4. Transmission Value Standardization

Transmission entries appeared in dozens of inconsistent formats (MT, 6M, CVT, AMT, etc.). All transmission values standardized into meaningful categories for analysis.

```python
df['Transmission'] = df['Transmission'].str.strip().str.upper()

df.loc[df['Transmission'].str.contains('CVT', regex=False, na=False), 'Transmission'] = 'Continuously variable transmission'
df.loc[df['Transmission'].str.contains('/|-', na=False), 'Transmission'] = 'Multi mode'
df.loc[df['Transmission'].str.contains('AMT', regex=False, na=False), 'Transmission'] = 'Automated manual transmission'
df.loc[df['Transmission'].str.contains('MT|MAN|6M', na=False), 'Transmission'] = 'Manual transmission'
df.loc[df['Transmission'].str.contains('AGS', na=False), 'Transmission'] = 'Auto gear shift'

df.loc[:, 'Transmission'] = df['Transmission'].str.capitalize()

df['Transmission'].unique()  # Final clean categories
```

### 5. Engine Capacity Cleaning

Engine capacity is critical for calculating excise duty. The column contained mixed numeric CC values, HP for trucks, and kWh for electric vehicles. Numeric extraction and validation isolated CC values, while HP and kWh were intentionally set to null, as their excise duty calculation is constant and independent of this column.

```python
df.loc[:, 'Engine_capacity_clean'] = (df['Engine_capacity']
                                     .astype(str)
                                     .str.extract(r'(\d+)')
                                     .astype(float))

df.loc[df['Engine_capacity_clean'] < 400, 'Engine_capacity_clean'] = None
df.loc[df['Engine_capacity'].str.contains('hp', case=False, na=False), 'Engine_capacity_clean'] = None
df.loc[df['Engine_capacity_clean'] < 400, 'Engine_capacity_clean'] = None

df.loc[:, 'Engine_capacity'] = df['Engine_capacity_clean']
df.drop(columns='Engine_capacity_clean', inplace=True)
df['Engine_capacity'] = pd.to_numeric(df['Engine_capacity'], errors='coerce')
```

### 6. Body Type and Fuel Standardization

Vehicle body types and fuel types were highly fragmented. Mapping dictionaries were applied to standardize them into controlled categories:

Body Type: SUV, HATCHBACK, SEDAN, WAGON, PICKUP, VAN, BUS, etc.

Fuel Type: ELECTRIC, PETROL, DIESEL, HYBRID, CNG, LNG

Outcome: Consistent categorical variables for downstream calculations.

### 7. CRSP Validation

The CRSP value is the core input for tax calculations. Records missing CRSP values were removed and resultin Data converted to float to ensure reliable computation. After cleaning, the dataset was fully analysis-ready.

---

## Tax Calculation Logic
The final cleaned dataset powers a robust calculator function that:
- Matches Make + Model to retrieve official CRSP
- Applies age-based depreciation (0–65% over 8 years)
- Computes all levies in correct order:
   - Import Duty (35%)
   - Excise Duty (tiered by engine/fuel; 10% for fully electric)
   - VAT (16%)
   - RDL (2%)
   - IDF (3.5%)

Returns a detailed breakdown
Hybrid vehicles are taxed using petrol engine capacity rules (per KRA policy).

## Example Calculation

A 2020 Peugeot 3008 GT LINE (Petrol, 1600cc):

| Component	| Value (KES) |
|-----------|-------------|
| CRSP	| 3,246,679.16 |
|Depreciation (50%)	| 1,623,339.58 |
|Import Duty (35%)	| 568,168.85 |
| Excise Duty	| 547,877.11 |
| VAT (16%)	| 438,301.69 |
| Railway Development Levy (2%)	| 32,466.79 |
| Import Declaration Fee (3.5%)	| 56,816.89 |
| Total Tax Payable	| 1,643,631.32 |

---

## Results and Insights

- Standardized and validated 5,000+ CRSP records for accurate calculations.
- SUVs and high-engine-capacity vehicles consistently incur the highest taxes, often exceeding 100% of the depreciated CRSP value.
- Electric vehicles attracted lower excise duty (10%), reflecting eco-friendly incentives.
- Calculator eliminates estimation errors and enables pre-purchase risk assessment.

---

## Business Value

This project demonstrates how unstructured regulatory data can be transformed into a practical decision-support tool. The calculator helps importers:

- Estimate true landed costs
- Avoid unexpected tax liabilities
- Make informed sourcing decisions

From an analytics perspective, it showcases data quality handling, validation, and business-rule fidelity.



***The full data cleaning workflow is documented in the Jupyter notebook included in this repository.***

---
