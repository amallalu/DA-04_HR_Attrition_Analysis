# DA-04 · HR Attrition Analysis

**Category:** Data Analytics · **Tools:** Power BI Desktop, Power Query, DAX · **Author:** Amal Lalu K K

![Dashboard preview](preview/dashboard_preview.png)

## 1. Business problem

A 1,400-person services company is losing staff faster than it can hire. HR needs to see where attrition is concentrated (department, role, age, tenure) and which factors, such as overtime and low satisfaction, are linked with people leaving.

**Questions this dashboard answers**

- What is our attrition rate and how many people left?
- Which departments and roles lose the most people?
- Are overtime, low job satisfaction or short tenure linked to leaving?
- Which groups should HR focus on first?

> **About the data:** Synthetic HR records generated for this project, structured like the widely used IBM HR Attrition sample. No real employees.

## 2. What's in this folder

| Path | What it is |
|---|---|
| `data/employees.csv` | One row per employee (current and those who left in the last 12 months) (1,484 rows) |
| `powerquery/*.m` | Power Query code for each table, ready to paste into the Advanced Editor |
| `dax/measures.dax` | Every DAX measure used in the report |
| `dax/calculated_columns_and_tables.dax` | Calculated columns and tables |
| `theme/AmalAnalytics_theme.json` | Report colour theme (colourblind-safe palette) |
| `preview/dashboard_preview.png` | What the finished dashboard looks like |
| `STEPS.html` | This guide, formatted to open in any browser |

## 3. Data dictionary

### Employees — `data/employees.csv`

One row per employee (current and those who left in the last 12 months)

| Column | Type | Description |
|---|---|---|
| `EmployeeID` | Text |  |
| `Department` | Text |  |
| `JobRole` | Text |  |
| `Gender` | Text |  |
| `Age` | Whole number |  |
| `MonthlyIncome` | Decimal number | Gross monthly salary in ₹ |
| `YearsAtCompany` | Whole number |  |
| `OverTime` | Text | Regularly works overtime (Yes/No) |
| `JobSatisfaction` | Whole number | Survey score 1 = Low … 4 = Very High |
| `WorkLifeBalance` | Whole number | Survey score 1 = Bad … 4 = Best |
| `DistanceFromHomeKm` | Whole number |  |
| `PromotedLast2Years` | Text |  |
| `Attrition` | Text | Yes = left the company in the last 12 months |

*Raw file contains deliberate data-quality issues to clean in Power Query: 1,484 raw rows → 1,459 clean rows.*

## 4. Build it step by step

### Step 1 · Set up a new report

1. Copy this whole folder somewhere permanent, e.g. `C:\PowerBI\DA-04_HR_Attrition_Analysis\`.
2. Open **Power BI Desktop** → **Blank report**.
3. **File → Options and settings → Options → Current file → Data Load** → untick **Auto date/time** (we use our own Date table). Also untick **Autodetect new relationships after data is loaded** under *Relationships*; you will create them yourself in a later step.
4. **View → Themes → Browse for themes** → pick `theme/AmalAnalytics_theme.json`.
5. **File → Save as** → `DA-04_HR_Attrition_Analysis.pbix` in this folder. Save often (Ctrl+S).

### Step 2 · Create the folder path parameter

All queries read files relative to one parameter, so the report works on any PC.

1. **Home → Transform data** to open the Power Query Editor.
2. **Home → Manage Parameters → New Parameter**.
3. Name: `FolderPath` · Type: **Text** · Current Value: the full path of this folder **ending with a backslash**, e.g. `C:\PowerBI\DA-04_HR_Attrition_Analysis\`.
4. Click **OK**.

### Step 3 · Load and clean the data in Power Query

Repeat for each table below: **Home → New Source → Blank Query** → **Home → Advanced Editor** → delete what's there → paste the code → **Done** → in *Query Settings* on the right, rename the query to the name shown.

#### Query `Employees`  (file `powerquery/Employees.m`)

What it does:

- reads `data\employees.csv` and promotes the first row to headers
- sets the data type of every column (using the en-US locale so dates and decimals parse the same on any PC)
- trims stray spaces in `Department`
- fixes inconsistent capitalisation (e.g. `north`, `NORTH` → `North`) in `Department`
- removes incomplete rows where `MonthlyIncome` is blank
- removes exact duplicate rows

```powerquery
let
    Source = Csv.Document(File.Contents(FolderPath & "data\employees.csv"), [Delimiter=",", Encoding=65001, QuoteStyle=QuoteStyle.Csv]),
    #"Promoted Headers" = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    #"Changed Type" = Table.TransformColumnTypes(#"Promoted Headers", {{"EmployeeID", type text}, {"Department", type text}, {"JobRole", type text}, {"Gender", type text}, {"Age", Int64.Type}, {"MonthlyIncome", type number}, {"YearsAtCompany", Int64.Type}, {"OverTime", type text}, {"JobSatisfaction", Int64.Type}, {"WorkLifeBalance", Int64.Type}, {"DistanceFromHomeKm", Int64.Type}, {"PromotedLast2Years", type text}, {"Attrition", type text}}, "en-US"),
    #"Trimmed Text" = Table.TransformColumns(#"Changed Type", {{"Department", Text.Trim, type text}}),
    #"Capitalized Each Word" = Table.TransformColumns(#"Trimmed Text", {{"Department", Text.Proper, type text}}),
    #"Removed Blank MonthlyIncome" = Table.SelectRows(#"Capitalized Each Word", each [MonthlyIncome] <> null),
    #"Removed Duplicates" = Table.Distinct(#"Removed Blank MonthlyIncome")
in
    #"Removed Duplicates"
```

When all queries show data, click **Home → Close & Apply**.

### Step 4 · Calculated tables and columns

Select table **Employees** → **Table tools → New column** — `Age Band`: groups ages

```dax
Age Band =
SWITCH ( TRUE (), Employees[Age] < 25, "Under 25", Employees[Age] < 35, "25-34", Employees[Age] < 45, "35-44", Employees[Age] < 55, "45-54", "55+" )
```

Then **Column tools → Sort by column → Age Band Sort**.

Select table **Employees** → **Table tools → New column** — `Age Band Sort`: sort order for Age Band (create this one first)

```dax
Age Band Sort =
SWITCH ( TRUE (), Employees[Age] < 25, 1, Employees[Age] < 35, 2, Employees[Age] < 45, 3, Employees[Age] < 55, 4, 5 )
```

Select table **Employees** → **Table tools → New column** — `Tenure Band`: groups tenure

```dax
Tenure Band =
SWITCH ( TRUE (), Employees[YearsAtCompany] <= 2, "0-2 yrs", Employees[YearsAtCompany] <= 5, "3-5 yrs", Employees[YearsAtCompany] <= 10, "6-10 yrs", "10+ yrs" )
```

Then **Column tools → Sort by column → Tenure Band Sort**.

Select table **Employees** → **Table tools → New column** — `Tenure Band Sort`: sort order for Tenure Band (create this one first)

```dax
Tenure Band Sort =
SWITCH ( TRUE (), Employees[YearsAtCompany] <= 2, 1, Employees[YearsAtCompany] <= 5, 2, Employees[YearsAtCompany] <= 10, 3, 4 )
```

### Step 5 · Create the measures

1. **Home → Enter data** → name the table `_Measures` → **Load**. (A dedicated table keeps all measures in one place.)
2. Select `_Measures` in the Data pane → **Home → New measure**, paste one measure, press Enter. Repeat for each.
3. After each one, set the format in **Measure tools → Format** as listed.
4. Once you have measures, delete the empty `Column1` from `_Measures`; the table moves to the top of the Data pane.

| Measure | Format | What it means |
|---|---|---|
| `Headcount` | Whole number, thousands separator | Employees in scope |
| `Leavers` | Whole number, thousands separator | Employees who left |
| `Attrition Rate` | Percentage, 1 decimal | Leavers ÷ headcount |
| `Active Employees` | Whole number, thousands separator | Headcount minus leavers |
| `Avg Monthly Income` | Currency, 0 decimals | Average salary |
| `Avg Years at Company` | Decimal number, 1 decimal | Average tenure |
| `Overtime Attrition Rate` | Percentage, 1 decimal | Attrition among people who work overtime |
| `Avg Job Satisfaction` | Decimal number, 2 decimals | Average survey score (1–4) |

```dax
Headcount =
    COUNTROWS ( Employees )

Leavers =
    CALCULATE ( [Headcount], Employees[Attrition] = "Yes" )

Attrition Rate =
    DIVIDE ( [Leavers], [Headcount] )

Active Employees =
    [Headcount] - [Leavers]

Avg Monthly Income =
    AVERAGE ( Employees[MonthlyIncome] )

Avg Years at Company =
    AVERAGE ( Employees[YearsAtCompany] )

Overtime Attrition Rate =
    CALCULATE ( [Attrition Rate], Employees[OverTime] = "Yes" )

Avg Job Satisfaction =
    AVERAGE ( Employees[JobSatisfaction] )
```

### Step 6 · Build the report pages

Set the page size first: **View → Page view → Fit to page**, and in **Format page → Canvas settings** use 16:9. Add a text box at the top of each page with the page title.

#### Page: Attrition Overview

| # | Visual | Fields to drag in | Settings |
|---|---|---|---|
| 1 | Card (x5) | Headcount · Leavers · Attrition Rate · Avg Monthly Income · Avg Years at Company |  |
| 2 | Clustered bar chart | Y-axis: Employees[Department] · X-axis: Attrition Rate · Tooltips: Leavers | Add an average line (Analytics pane) to compare against company rate |
| 3 | Clustered bar chart | Y-axis: Employees[JobRole] · X-axis: Attrition Rate | Sort descending |
| 4 | Donut chart | Legend: Employees[Attrition] · Values: Headcount |  |
| 5 | Slicers | Employees[Department] · Employees[Gender] |  |

#### Page: Attrition Drivers

| # | Visual | Fields to drag in | Settings |
|---|---|---|---|
| 1 | Clustered column chart | X-axis: Employees[OverTime] · Y-axis: Attrition Rate |  |
| 2 | Clustered column chart | X-axis: Employees[Age Band] · Y-axis: Attrition Rate |  |
| 3 | Clustered column chart | X-axis: Employees[Tenure Band] · Y-axis: Attrition Rate |  |
| 4 | Clustered column chart | X-axis: Employees[JobSatisfaction] · Y-axis: Attrition Rate |  |
| 5 | Key influencers | Analyze: Employees[Attrition] · Explain by: OverTime, JobSatisfaction, Tenure Band, Age Band, WorkLifeBalance, PromotedLast2Years, DistanceFromHomeKm | Set 'What influences Attrition to be' = Yes. Power BI's built-in AI visual ranks the drivers |

#### Page: Employee List

| # | Visual | Fields to drag in | Settings |
|---|---|---|---|
| 1 | Table | EmployeeID, Department, JobRole, Age, YearsAtCompany, OverTime, JobSatisfaction, Attrition | Filter Attrition = No and OverTime = Yes and JobSatisfaction ≤ 2 to list people at risk |

### Step 7 · Check your numbers

With **no slicers or filters applied**, your cards should show these values. If one is different, re-check that measure and the cleaning steps.

| Measure | Expected value |
|---|---|
| `Headcount` | 1,459 |
| `Leavers` | 325 |
| `Attrition Rate` | 22.3% |
| `Active Employees` | 1,134 |
| `Avg Monthly Income` | ₹88,324 |
| `Avg Years at Company` | 4.3 |
| `Overtime Attrition Rate` | 40.3% |
| `Avg Job Satisfaction` | 2.71 |

### Step 8 · Save and share

1. **File → Save** — your finished `DA-04_HR_Attrition_Analysis.pbix`.
2. **File → Export → Export to PDF** for a static copy, or take a screenshot of each page for your portfolio and GitHub.
3. Optional: **Home → Publish** to Power BI Service (needs a work or school account) to get a shareable link.

## 5. Key findings from this data

- **Attrition rate by department:** highest is Sales (29.1%), lowest is Technology (16.1%).
- **Attrition rate by age band:** highest is Under 25 (39.6%), lowest is 55+ (9.1%).
- **Attrition rate by tenure:** highest is 0-2 yrs (33.7%), lowest is 10+ yrs (11.9%).
- **Attrition rate by job satisfaction:** highest is 1 Low (33.3%), lowest is 4 Very High (17.9%).
- Overtime workers leave at 40.3% vs 14.9% for those who don't.

## 6. Skills shown

- DAX calculated columns with SWITCH(TRUE()) banding and sort columns
- Key influencers AI visual
- HR analytics: attrition rate, drivers, at-risk lists
- Power Query cleaning
