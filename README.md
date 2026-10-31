# Amazon Sales Dashboard — Power BI Sales, Profit & Discount Analytics

An end-to-end Business Intelligence project built with Microsoft Power BI on a
US e-commerce transaction dataset of 10,000 orders (Jan–Feb 2026). The project
covers data profiling in Python, star-schema data modeling with three
engineered dimension tables, 20 DAX measures, and an executive dashboard
delivering insights on revenue, estimated profit, discount behaviour, regional
performance, and category mix.

---

## Dataset Overview

| Attribute | Value |
|---|---|
| Transactions | 10,000 |
| Unique customer IDs | 6,016 |
| Period | 1 Jan 2026 – 10 Feb 2026 |
| Gross revenue | $754.21M |
| Net revenue | $641.28M |
| Total discount | $112.94M (15.0% of gross) |
| Total shipment cost | $850.53K |
| Estimated gross profit | $81.09M (12.6% margin) |
| Average order value | $64,213 |
| Total quantity | 30,144 units |
| Product categories | 3 (7 sub-categories) |
| States | 50 (all US states) |
| Payment methods | 4 |

> The dataset is synthetic. Price levels, customer identifiers and date
> sequences do not behave like real transactional data — see
> **Data Quality Findings** below.

## Project Scope

1. **Data profiling (Python / pandas)** — verified the revenue formula
   (`total_sales = quantity × unit_price × (1 − discount) + shipping_cost`),
   tested key uniqueness for `order_id`, `customer_id` and `product_id`, and
   quantified date-sequence anomalies before any modeling decision was made.
2. **Dimension engineering** — the source file lacks profit, region and
   segment columns required by the target dashboard, so three dimension tables
   were built: a state-to-region mapping (50 states, Superstore convention), a
   per-category cost assumption table, and a calendar table.
3. **Data modeling (Power BI)** — star schema with `Sales` as the single fact
   table (one row per order), four dimensions, all relationships one-to-many
   single-direction. Category, sub-category and brand kept as degenerate
   dimensions because `product_id` failed the uniqueness test.
4. **DAX measures** — 20 measures covering core metrics, estimated
   profitability, ratios, contribution and ranking, and short-window time
   intelligence.
5. **Dashboard** — 4 KPI cards, category treemap, geographic profit map,
   category trend, regional comparison, discount analysis by sub-category, and
   a payment-method matrix.

## Key Findings

- **Sales are evenly spread across categories** — Electronics $219.36M,
  Home $214.76M, Fashion $208.01M — a spread of less than 6%, with no category
  dominating the revenue mix.
- **Estimated profit inverts the revenue ranking.** Fashion contributes
  $48.84M of profit against Electronics' $7.01M, despite Electronics selling
  more. This follows directly from the assumed cost ratios (Electronics 0.82,
  Fashion 0.65) and is therefore an output of the assumption, not an
  observation from the data.
- **No dominant payment channel.** COD 25.5%, Card 25.1%, NetBanking 24.7%,
  UPI 24.7% — a near-uniform split consistent with synthetic generation rather
  than real consumer behaviour.
- **Discount is flat at 15.0% of gross revenue**, with negligible variation
  across sub-categories, removing most of the analytical value from the
  discount-by-sub-category visual.
- **Shipment cost is immaterial** at 0.13% of total sales, so the fourth KPI
  card carries little decision-making weight for this dataset.

## Data Quality Findings

These were identified during profiling and constrained the model design:

| Issue | Scale | Consequence |
|---|---|---|
| `ship_date` precedes `order_date` | 49.2% of rows | Fulfilment-lead-time measures excluded from the dashboard |
| `delivery_date` precedes `ship_date` | 48.8% of rows | Delivery-performance analysis not feasible |
| `product_id` maps to multiple categories, names, brands and prices | 99.7% of IDs | No `DimProduct`; product attributes kept in the fact table |
| `customer_id` maps to multiple customer names | 2,780 of 6,016 IDs | `DimCustomer` keyed on `customer_id` only, with a representative name |
| `city` name shared across states | up to 11 states per city name | City never used as a standalone key |
| `order_status` and `country` single-valued | 100% `Delivered`, 100% `USA` | Both excluded as slicers |
| No cost or profit column | — | Profit modeled from category cost assumptions |

## Data Model

Star schema, `Sales` as the single fact table (10,000 rows, one row per order).

| From (1) | To (\*) | Cardinality | Cross filter | Active |
|---|---|---|---|---|
| `DimDate[Date]` | `Sales[order_date]` | One to many | Single | Yes |
| `DimDate[Date]` | `Sales[ship_date]` | One to many | Single | No |
| `DimRegion[state]` | `Sales[state]` | One to many | Single | Yes |
| `DimMargin[category]` | `Sales[category]` | One to many | Single | Yes |
| `DimCustomer[customer_id]` | `Sales[customer_id]` | One to many | Single | Yes |

### Supporting tables

**DimRegion** — 50 US states mapped to West / East / Central / South following
the Superstore convention (Ohio in East, Texas in Central). Includes
`state_code` for map geocoding.

**DimMargin** — cost assumptions used to derive profit:

| category | cost_ratio | implied margin |
|---|---|---|
| Electronics | 0.82 | 18% |
| Fashion | 0.65 | 35% |
| Home | 0.75 | 25% |

**DimDate** — daily calendar covering the transaction window, marked as date
table, with year, month, week-number and weekday attributes.

## DAX Measures (20)

### Core metrics

| Measure | Formula | Purpose |
|---|---|---|
| Total Sales | `SUM(Sales[total_sales])` | Total order value, shipping included |
| Total Shipment Cost | `SUM(Sales[shipping_cost])` | Shipping charged across orders |
| Net Revenue | `[Total Sales] - [Total Shipment Cost]` | Merchandise revenue after discount |
| Gross Sales | `SUMX(Sales, Sales[quantity] * Sales[unit_price])` | Merchandise value before discount |
| Total Discount Amount | `[Gross Sales] - [Net Revenue]` | Discount value given away |
| Avg Discount % | `AVERAGE(Sales[discount])` | Mean discount rate per order line |
| Total Quantity | `SUM(Sales[quantity])` | Units sold |
| Total Orders | `DISTINCTCOUNT(Sales[order_id])` | Unique orders |
| Total Customers | `DISTINCTCOUNT(Sales[customer_id])` | Unique customer IDs |

### Estimated profitability

| Measure | Formula | Purpose |
|---|---|---|
| Total COGS | `SUMX(Sales, Sales[quantity] * Sales[unit_price] * RELATED(DimMargin[cost_ratio]))` | Cost of goods from category assumptions |
| Total Profit | `[Net Revenue] - [Total COGS] - [Total Shipment Cost]` | Gross profit after cost and shipping |
| Profit Margin % | `DIVIDE([Total Profit], [Total Sales])` | Profit as a share of total sales |

### Ratios

| Measure | Formula | Purpose |
|---|---|---|
| AOV | `DIVIDE([Total Sales], [Total Orders])` | Average value per order |
| Avg Order Quantity | `DIVIDE([Total Quantity], [Total Orders])` | Average units per order |
| Sales per Customer | `DIVIDE([Total Sales], [Total Customers])` | Average revenue per customer ID |
| Shipping Cost Ratio % | `DIVIDE([Total Shipment Cost], [Total Sales])` | Shipping burden on revenue |

### Contribution and ranking

| Measure | Formula | Purpose |
|---|---|---|
| % of Total Sales | `DIVIDE([Total Sales], CALCULATE([Total Sales], ALLSELECTED(Sales)))` | Each slice's share of the selected total |
| Sales Rank by Sub-Category | `RANKX(ALLSELECTED(Sales[sub_category]), [Total Sales], , DESC)` | Sub-category ranking, drives Top N |

### Time intelligence

| Measure | Formula | Purpose |
|---|---|---|
| Sales 7D Moving Avg | `AVERAGEX(DATESINPERIOD(DimDate[Date], MAX(DimDate[Date]), -7, DAY), [Total Sales])` | Weekly rolling average, smooths daily noise |
| Sales YTD | `TOTALYTD([Total Sales], DimDate[Date])` | Cumulative sales from the start of the year |

> Month-over-month and year-over-year measures were deliberately omitted: the
> dataset spans only 41 days, so period-over-period comparison would be
> computed against a single incomplete month.

## Dashboard Components

| Visual | Type | Fields |
|---|---|---|
| KPI row | Card (×4) | Total Sales · Total Profit · Total Quantity · Total Shipment Cost |
| Category mix | Treemap | Group `category`, Details `sub_category`, Values Total Sales |
| Geographic profit | Map | Location `DimRegion[state_code]`, Size Total Profit |
| Category trend | Area chart | Axis `DimDate[Week No]`, Values Total Sales, Small multiples `category` |
| Payment breakdown | Matrix | Rows `category`, Columns `payment_method`, Values Total Sales + Total Profit |
| Regional comparison | Clustered column | Axis `DimRegion[region]`, Values Total Sales + Total Profit |
| Discount analysis | Clustered bar | Axis `sub_category`, Values Avg Discount % |
| Slicers | Slicer (×3) | `DimDate[Date]` range, `category`, `region` |

## Limitations

- **Profit figures are modeled, not measured.** The source data contains no
  cost column. Every profit number depends on the three cost ratios in
  `DimMargin`; changing them changes the profit ranking of categories.
- **The 41-day window** prevents seasonality analysis and makes the
  year-on-year view in comparable dashboards inapplicable here.
- **Customer-level analysis is unreliable** given that 46% of customer IDs
  resolve to more than one name.
- **Findings are not generalisable** to real Amazon operations; the dataset is
  synthetic and several distributions are close to uniform by construction.

## Repository Files

| File | Description |
|---|---|
| `amazon_sales_dataset_clean.csv` | Source transaction data, 10,000 rows |
| `dim_region_usa.csv` | State-to-region mapping, 50 rows |
| `dim_margin.csv` | Category cost assumptions, 3 rows |
| `README.md` | This document |

## Tools

Microsoft Power BI Desktop · DAX · Python (pandas) for profiling
