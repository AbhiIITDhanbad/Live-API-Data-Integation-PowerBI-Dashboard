# 🌍 Live API Data Integration: Real-Time Dashboards with Python & Power BI

An interactive Power BI dashboard that explores global economic, health, labour, technology and environmental indicators. Data is pulled live from the **World Bank public API** using Python, processed with pandas, and connected directly to Power BI so the dashboard can refresh as new data becomes available, with no manual downloads.

## 📖 Project Overview

This project builds a live, interactive dashboard showing important global statistics such as health, education, employment and other key indicators. A Python script fetches the data from the API and prepares it for analysis. The script is then connected to Power BI, so the dashboard refreshes whenever new data is available and users do not need to update anything by hand.

The dashboard answers **9 analytical questions** covering the economic and social landscape, regional health spending, long-term trends, the effect of internet access on immunization and unemployment, poverty reduction performance, and the relationships between health indicators.

---

## 🛠 Tech Stack

| Tool | Purpose |
|------|---------|
| **Python** (`requests`, `pandas`, `numpy`) | Fetching and cleaning data from the World Bank API |
| **Python** (`seaborn`, `matplotlib`) | Heatmap and regression scatter plot |
| **Power BI** | Data modelling, DAX measures and dashboard visuals |
| **World Bank API** | Live data source (`api.worldbank.org`) |

## 🔄 Data Pipeline

All data collection is done in **`API_Data.ipynb`**:

1. **Countries metadata**: fetched from the World Bank countries endpoint. The nested `region`, `incomeLevel` and `lendingType` fields are flattened, unused columns (`adminregion`, `capitalCity`) are dropped, and `iso2Code` is renamed to `country_id`.
2. **Indicator catalogue**: all available indicators are paged through and saved to `final_df.csv`.
3. **Indicator data by domain**: for each domain, the selected indicators are fetched for all countries with pagination (with `time.sleep` to be gentle on the API). Only records **after 2015** are kept.
4. **Merge with country metadata**: each domain table is joined to the countries table on `country_id`, then redundant columns (`indicator_id`, `name`, `id`) are dropped.

### Domain tables collected

| Table | Indicators |
|-------|-----------|
| `economic` | GDP growth (annual %), GDP per capita (current US$) |
| `labour_market` | Unemployment (total), youth unemployment (15–24), total labour force |
| `trade` | Exports and imports of goods & services (current US$) |
| `poverty` | Poverty headcount ratio at national poverty lines, Gini index |
| `environment` | Renewable energy consumption (% of total), forest area (% of land area) |
| `health` | Life expectancy, infant mortality, basic water access, health expenditure (% of GDP), DPT & measles immunization, maternal mortality / risk, communicable disease deaths, TB incidence, skilled birth attendance, population 65+, HIV incidence |
| `technology` | Internet users (% of population), mobile cellular subscriptions (per 100 people) |

---

## 🧩 Data Model

All of the tables collected in the notebook are loaded into Power BI, and the relationships are built as follows:

- **`countryIDs` table**: a separate dimension table holding all the **unique `country_id`s**. It is used to establish **many-to-one relationships** between every fact table and the country dimension, so one slicer or filter works across all tables.
- **Appended table for Problem 3**: `environment` + `technology` + `economy` + `labour_market`, so that all the critical indicators in Problem 3 live in one table.
- **`ForScatterPlot` (appended table for Problem 4)**: `technology` + `health` + `labour_market`, so that all the critical indicators in Problem 4 live in one table.
- **Measure tables** (`Immunization`, `Internet`, `Unemployment`, `Poverty Reduction %`) are used for Problems 4–7.

---

## ✅ Problem Statements & Solutions

| # | Problem Statement | Solution in Power BI |
|---|-------------------|----------------------|
| **1** | Understand the overall economic and social landscape: average GDP per capita, trade value, health spending (% of GDP), GDP growth rate and forest area proportion | **Card visuals** for each average metric, plus a **region slicer** to filter by region |
| **2** | Compare average health spending (% of GDP) across world regions | **Bar chart** of average health expenditure by region |
| **3** | Analyze trends in key socio-economic indicators over time (forest area, mobile and internet subscriptions, GDP, renewable energy, unemployment) | A new appended table (environment + technology + economy + labour_market), shown as a **line chart** with the indicators as the legend/labels |
| **4** | Evaluate the impact of internet access on immunization awareness | Appended table `ForScatterPlot` (technology + health + labour_market), shown as a **year-wise play-axis scatter plot** of average Internet vs. Immunization values. Uses the `Immunization` and `Internet` measures |
| **5** | Assess the role of internet access in reducing unemployment | Uses the `Internet` and `Unemployment` measures to compare how the average values move together over time |
| **6** | Identify the bottom 10 countries with the least progress in poverty reduction | `Poverty Reduction %` measure with a **Bottom 10** filter |
| **7** | Identify the top 10 countries with the most progress in poverty reduction | `Poverty Reduction %` measure with a **Top 10** filter |
| **8** | Examine interrelationships among health indicators | **Correlation heatmap** (Python visual) |
| **9** | Investigate the relationship between government health spending and life expectancy | **Scatter plot with trend line** (Python visual) |

---

## 🧮 DAX Measures

### Immunization (average of DPT and measles immunization)

```dax
Immunization =
CALCULATE(
    AVERAGE('ForScatterPlot'[value]),
    FILTER(
        'ForScatterPlot',
        'ForScatterPlot'[indicator_name] = "Immunization, DPT % of children ages 12-23 months"
            || 'ForScatterPlot'[indicator_name] = "Immunization, measles % of children ages 12-23 months"
    )
)
```

### Internet (average internet penetration)

```dax
Internet =
CALCULATE(
    AVERAGE('ForScatterPlot'[value]),
    FILTER(
        'ForScatterPlot',
        'ForScatterPlot'[indicator_name] = "Individuals using the Internet % of population"
    )
)
```

### Unemployment (average unemployment rate)

```dax
Unemployment =
CALCULATE(
    AVERAGE('labour_market'[value]),
    FILTER(
        'labour_market',
        'labour_market'[indicator_name] = "Unemployment, total (% of total labor force) (modeled ILO estimate)"
    )
)
```

### Poverty Reduction % (Problems 6 & 7)

Compares each country's poverty headcount ratio in its **earliest** and **latest** year with data, then calculates the relative change.

```dax
Poverty Reduction % =
VAR MINYEAR =
    CALCULATE(
        MIN('property'[year]),
        FILTER(
            'property',
            'property'[indicator_name] = "Poverty headcount ratio at national poverty lines (% of population)"
                && NOT ISBLANK('property'[value])
        )
    )
VAR MAXYEAR =
    CALCULATE(
        MAX('property'[year]),
        FILTER(
            'property',
            'property'[indicator_name] = "Poverty headcount ratio at national poverty lines (% of population)"
                && NOT ISBLANK('property'[value])
        )
    )
VAR POVMIN =
    CALCULATE(
        MIN('property'[value]),
        FILTER(
            'property',
            'property'[indicator_name] = "Poverty headcount ratio at national poverty lines (% of population)"
                && NOT ISBLANK('property'[value])
                && 'property'[year] = MINYEAR
        )
    )
VAR POVMAX =
    CALCULATE(
        MAX('property'[value]),
        FILTER(
            'property',
            'property'[indicator_name] = "Poverty headcount ratio at national poverty lines (% of population)"
                && NOT ISBLANK('property'[value])
                && 'property'[year] = MAXYEAR
        )
    )
RETURN
    (POVMIN - POVMAX) / POVMAX
```

> `property` is the poverty table as named in the Power BI model.

---

## 🐍 Python Visuals (Problems 8 & 9)

Both visuals are built with Python (see the last cells of `API_Data.ipynb`).

**Problem 8: Correlation heatmap of health indicators**

```python
df_wide = health.pivot_table(index=["country_value", "year"],
                             columns="indicator_name",
                             values="value")
corr = df_wide.corr()

plt.figure(figsize=(16, 16), facecolor="none")
sns.heatmap(corr, annot=True, cmap="coolwarm", cbar_kws={"label": "Correlation"})
plt.title("Relationship between various health indicators")
plt.show()
```

**Problem 9: Health expenditure vs. life expectancy (with trend line)**

```python
df_pivot = health.pivot(index=["country_value", "year"],
                        columns="indicator_name",
                        values="value").reset_index()

sns.regplot(data=df_pivot,
            x="Current health expenditure (% of GDP)",
            y="Life expectancy at birth, total (years)",
            scatter_kws={"alpha": 0.5, "s": 40},
            line_kws={"color": "red", "lw": 2})
plt.title("Relationship between Expenditure and Life Expectancy")
plt.show()
```

Power BI is connected to the Python script, so these visuals and the rest of the dashboard update as the live API data changes.

---
