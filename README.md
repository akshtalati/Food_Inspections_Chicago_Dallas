# Food Inspections Data Engineering & Business Intelligence
**Chicago & Dallas – Unified Public Health Analytics System**

## 📌 Project Overview
This project implements a full end-to-end data engineering and business intelligence (BI) system using public food inspection datasets from the cities of Chicago and Dallas.

The core challenge addressed is **semantic incompatibility**: although both cities publish food inspection data, they use fundamentally different schemas, risk definitions, scoring mechanisms, and violation representations. Direct comparison without proper modeling leads to incorrect metrics, inflated counts, and misleading insights.

This project solves that problem by designing a unified dimensional model, supported by city-aware ETL pipelines, grain-safe fact modeling, and business-requirement-driven SQL analytics, culminating in decision-ready Power BI dashboards.

---

## 🎯 Business Objectives
- Enable cross-city comparison of food inspection outcomes
- Normalize risk, inspection results, and violations across incompatible datasets
- Prevent KPI distortion caused by row explosion and schema mismatch
- Support public health and regulatory insights
- Demonstrate industry-grade data engineering + BI practices

---

## 🧩 Business Requirements (What the System Must Answer)
The system is explicitly designed to answer the following:
- How do inspection results vary by inspection type?
- What are the monthly, weekly, and weekday trends in inspections?
- How do risk levels and inspection scores compare?
- What are the most common violations?
- Which businesses are repeat offenders?
- How do facility types perform in terms of failures?
- What geographic patterns exist in inspection activity?

These requirements are formally implemented in SQL and dashboards (see Appendix).

---

## 📂 Data Sources

### 📍 Chicago Food Inspections
- Narrow, reporting-oriented dataset (~17 columns)
- Risk expressed as categorical labels
- Violations often stored as free-text
- Includes business identity, inspection results, location, and violation descriptions

### 📍 Dallas Food Inspections
- Wide, operational dataset (~114 columns)
- Risk inferred from numeric inspection scores
- More structured violation descriptions
- Includes detailed facility and operational metadata

### 📊 Data Scale & Characteristics
- ~200,000+ inspection records per city
- Multiple source tables per city
- Mixed structured and semi-structured fields
- One-to-many and many-to-many relationships (inspections ↔ violations)
- Strong semantic mismatch between sources

---

## 🏗️ Architecture Overview
```
Raw City Data (Chicago / Dallas)
        ↓
Data Profiling (ydata-profiling)
        ↓
City-Specific Cleaning & Standardization (Alteryx)
        ↓
Snowflake Staging Tables
        ↓
Dimensional Model (Star Schema)
        ↓
Business-Requirement-Driven SQL
        ↓
Power BI Dashboards
```

---

## 🛠️ Tools & Technologies

| Layer | Tools |
|-------|-------|
| Profiling | ydata-profiling |
| ETL | Alteryx |
| Data Warehouse | Snowflake |
| Modeling | Dimensional (Star Schema) |
| Analytics | SQL |
| Visualization | Power BI |

---

## 🧠 Data Modeling Approach

### 📐 Fact Table
**FCT_Inspection**
- **Grain:** One row per inspection event
- **Why:** Prevents duplication when inspections have multiple violations

**Measures & Attributes:**
- Inspection results
- Inspection type
- Inspection score / normalized risk
- Source city
- Audit timestamp

This grain decision is the single most important correctness safeguard in the project.

### 📏 Dimension Tables

| Dimension | Purpose |
|-----------|---------|
| DIM_Business | Identify repeat offenders |
| DIM_Facility | Compare performance by facility type |
| DIM_Location | Geographic and city-level analysis |
| DIM_Date | Time-based trends |
| DIM_Risk | Normalize categorical risk and numeric scores |
| DIM_Violation | Structured violation analysis |

### 🔗 Bridge Table
**Inspection ↔ Violation**
- Resolves many-to-many relationship
- Prevents row explosion
- Enables correct violation analytics without inflating inspection counts

---

## 🧠 Design Decisions & Trade-offs (With Evidence)

### 1️⃣ Star Schema vs Fully Normalized Model
**Decision:** Use dimensional modeling  
**Why:** BI workloads prioritize query clarity, performance, and correctness  
**Trade-off:** Some redundancy accepted in exchange for stable analytics  
**Evidence:** Clean mapping of BR queries directly to fact/dim tables

### 2️⃣ One Row per Inspection (Fact Grain)
**Decision:** Fix fact table grain at inspection level  
**Why:** Violations are many-to-one; joining them directly inflates KPIs  
**Trade-off:** Requires bridge table and slightly more complex SQL  
**Evidence:** Correct computation of % high-risk inspections and repeat offenders

### 3️⃣ City-Aware ETL Instead of Single Pipeline
**Decision:** Separate Alteryx workflows for Chicago and Dallas  
**Why:** Risk and violation semantics differ fundamentally  
**Trade-off:** More ETL logic, but no loss of semantic meaning  
**Evidence:** Successful unification of risk levels and scores

### 4️⃣ Risk Normalization Strategy
**Decision:** Create a unified DIM_Risk  
**Why:** Chicago uses categorical risk; Dallas uses numeric scores  
**Trade-off:** Requires explicit mapping and validation  
**Evidence:** BR3 (Risk Level and Scores) query works across both cities

### 5️⃣ Business-Requirement-Driven SQL
**Decision:** Write SQL directly mapped to BRs  
**Why:** Prevents exploratory, unstructured analytics  
**Trade-off:** Less ad-hoc flexibility  
**Evidence:** Every dashboard visual traces back to a defined BR query

---

## 🧮 SQL Appendix (Mapped to Business Requirements)

### BR1 – Inspection Results by Type
```sql
SELECT
    Inspection_Type,
    Results,
    COUNT(*) AS Total_Inspections
FROM FCT_Inspection
GROUP BY Inspection_Type, Results
ORDER BY Total_Inspections DESC;
```

### BR2 – Monthly / Weekly Inspection Trends
```sql
SELECT
    d.Year,
    d.Month,
    d.Weekday,
    COUNT(*) AS Total_Inspections
FROM FCT_Inspection f
JOIN DIM_Date d ON f.Date_Key = d.Date_Key
GROUP BY d.Year, d.Month, d.Weekday
ORDER BY d.Year, d.Month;
```

### BR3 – Risk Level and Scores
```sql
SELECT
    r.Risk_Level,
    COUNT(*) AS Total_Inspections,
    AVG(r.Inspection_Score) AS Avg_Score
FROM FCT_Inspection f
JOIN DIM_Risk r ON f.Risk_Key = r.Risk_Key
GROUP BY r.Risk_Level
ORDER BY Avg_Score ASC;
```

### BR4 – Top Violations
```sql
SELECT
    v.Violation_Description,
    COUNT(*) AS Violation_Count
FROM FCT_Inspection f
JOIN DIM_Violation v ON f.Violation_Key = v.Violation_Key
GROUP BY v.Violation_Description
ORDER BY Violation_Count DESC
LIMIT 10;
```

### BR5 – Repeat Offender Businesses
```sql
SELECT
    b.DBA_Name,
    b.AKA_Name,
    b.License,
    COUNT(*) AS Failed_Inspections
FROM FCT_Inspection f
JOIN DIM_Business b ON f.Business_Key = b.Business_Key
WHERE f.Results IN ('Fail', 'Out of Business')
GROUP BY b.DBA_Name, b.AKA_Name, b.License
HAVING COUNT(*) > 1
ORDER BY Failed_Inspections DESC;
```

### BR6 – Facility Type Comparison
```sql
SELECT
    fac.Facility_Type,
    COUNT(*) AS Total_Inspections,
    SUM(CASE WHEN f.Results = 'Fail' THEN 1 ELSE 0 END) AS Failures
FROM FCT_Inspection f
JOIN DIM_Facility fac ON f.Facility_Key = fac.Facility_Key
GROUP BY fac.Facility_Type
ORDER BY Failures DESC;
```

### BR7 – Location-Based Insights
```sql
SELECT
    loc.City,
    loc.State,
    loc.Zip_Code,
    COUNT(*) AS Total_Inspections
FROM FCT_Inspection f
JOIN DIM_Location loc ON f.Location_Key = loc.Location_Key
GROUP BY loc.City, loc.State, loc.Zip_Code
ORDER BY Total_Inspections DESC;
```

---

## 📊 Visualization Layer
Power BI dashboards include:
- City-level comparison (Chicago vs Dallas)
- Risk distribution and trends
- Violation frequency analysis
- Facility type performance
- Geographic inspection density

Design emphasizes KPI correctness, clarity, and narrative storytelling.

---

## ✅ Key Outcomes
- Unified two incompatible city datasets into one analytical system
- Eliminated KPI inflation and row duplication
- Delivered trustworthy, comparable public-health insights
- Demonstrated end-to-end data engineering and BI expertise
