# Power Query Portfolio: Retail Sales ETL (README)

This repository contains a complete **Power Query (M)** ETL pipeline for Retail Sales data. The project demonstrates parameterized ingestion, schema standardization, currency normalization, and star‑schema output suitable for Power BI or Excel.

---

## 🧩 Overview
This project simulates a small‑scale **Retail Sales ETL** solution built entirely with **Power Query (M)**. It ingests CSV files from a folder, applies transformations, enriches data with a date dimension and exchange rates, and produces a clean, analytics‑ready model.

---

## 📁 Folder Structure
```
/Retail-Sales-ETL-PowerQuery/
├── README.md
├── /Queries/
│   ├── Parameters.pq
│   ├── fnCleanColumnNames.pq
│   ├── Staging_Sales.pq
│   ├── Dim_Calendar.pq
│   ├── ExchangeRates.pq
│   ├── Sales_With_Currency.pq
│   ├── Sales_Summary.pq
├── /Data/
│   ├── sales_jan.csv
│   ├── sales_feb.csv
│   ├── sales_mar.csv
└── RetailSalesETL.pbix (optional Power BI demo)
```

---

## 🚀 How to Use
1. Open **Power BI Desktop** or **Excel Power Query Editor**.
2. Create a **Blank Query** for each `.pq` file in the `/Queries` folder.
3. Copy and paste the M script from each file.
4. Update the path in `Parameters.pq` to your local folder of CSVs.
5. Load and refresh to generate the star‑schema tables.

---

## ⚙️ Features
- Folder ingestion with schema drift handling
- Parameterized configuration (folder path, fiscal month, base currency)
- Reusable cleaning function
- Robust typing and error handling with `try ... otherwise`
- Calendar dimension with fiscal year logic
- Web JSON ingestion for exchange rates (with fallback)
- Currency normalization and star‑schema aggregation

---

## 🧱 Example Queries
### Parameters.pq
```m
let
    SalesFolderPath = "C:\\Data\\Portfolio\\RetailSales",
    FiscalStartMonth = 7,
    BaseCurrency = "USD"
in
    [ SalesFolderPath = SalesFolderPath, FiscalStartMonth = FiscalStartMonth, BaseCurrency = BaseCurrency ]
```

### fnCleanColumnNames.pq
```m
let fnCleanColumnNames = (table as table) as table =>
let
    src = Table.TransformColumnNames(
        table,
        each Text.Replace(Text.Replace(Text.Lower(_), " ", "_"), "-", "_")
    )
in
    src
in
    fnCleanColumnNames
```

### Staging_Sales.pq
```m
let
    p = Parameters,
    Source = Folder.Files(p[SalesFolderPath]),
    Filtered = Table.SelectRows(Source, each Text.Lower([Extension]) = ".csv"),
    AddTables = Table.AddColumn(Filtered, "Data", each try Csv.Document(File.Contents([Folder Path] & [Name]), [Delimiter=",", Columns=7, Encoding=65001, QuoteStyle=QuoteStyle.Csv]) otherwise null),
    Kept = Table.SelectRows(AddTables, each [Data] <> null),
    Promoted = Table.TransformColumns(Kept, {{"Data", each Table.PromoteHeaders(_, [PromoteAllScalars=true])}}),
    Cleaned = Table.TransformColumns(Promoted, {{"Data", each fnCleanColumnNames(_)}}),
    Expanded = Table.ExpandTableColumn(Cleaned, "Data", {"date","orderid","store","sku","qty","unitprice","currency"}, {"date","orderid","store","sku","qty","unitprice","currency"}),
    Typed = Table.TransformColumnTypes(Expanded,{{"date", type date},{"orderid", type text},{"store", type text},{"sku", type text},{"qty", Int64.Type},{"unitprice", type number},{"currency", type text}}),
    AddedAmount = Table.AddColumn(Typed, "amount", each [qty] * [unitprice], type number),
    CleanedCurrency = Table.TransformColumns(AddedAmount, {{"currency", each Text.Upper(_), type text}})
in
    CleanedCurrency
```

### Dim_Calendar.pq
```m
let
    p = Parameters,
    TryMin = try List.Min(Table.Column(Staging_Sales, "date")) otherwise #date(2024,1,1),
    TryMax = try List.Max(Table.Column(Staging_Sales, "date")) otherwise #date(2026,12,31),
    StartDate = Date.StartOfMonth(TryMin),
    EndDate = Date.EndOfMonth(TryMax),
    DayCount = Duration.Days(EndDate - StartDate) + 1,
    DatesList = List.Dates(StartDate, DayCount, #duration(1,0,0,0)),
    ToTable = Table.FromList(DatesList, Splitter.SplitByNothing(), {"Date"}),
    AddYear = Table.AddColumn(ToTable, "Year", each Date.Year([Date]), Int64.Type),
    AddMonth = Table.AddColumn(AddYear, "Month", each Date.Month([Date]), Int64.Type),
    AddMonthName = Table.AddColumn(AddMonth, "MonthName", each Date.ToText([Date], "MMM"), type text),
    AddQuarter = Table.AddColumn(AddMonthName, "Quarter", each Date.QuarterOfYear([Date]), Int64.Type),
    AddWeek = Table.AddColumn(AddQuarter, "Week", each Date.WeekOfYear([Date]), Int64.Type),
    fsm = p[FiscalStartMonth],
    AddFiscalYear = Table.AddColumn(AddWeek, "FiscalYear", each let m = [Month] in if m >= fsm then [Year] else [Year]-1, Int64.Type),
    AddFiscalPeriod = Table.AddColumn(AddFiscalYear, "FiscalPeriod", each let m=[Month] in if m >= fsm then m - fsm + 1 else m + (12 - fsm + 1), Int64.Type)
in
    AddFiscalPeriod
```

### ExchangeRates.pq
```m
let
    p = Parameters,
    TryWeb = try Json.Document(Web.Contents("https://raw.githubusercontent.com/PublicAPIs-Example/data/main/sample_fx.json", [Timeout=#duration(0,0,30,0)])) otherwise null,
    WebToTable = if TryWeb <> null and Record.HasFields(TryWeb, {"rates"}) then let rates = TryWeb[rates], toList = Record.ToTable(rates), Renamed = Table.RenameColumns(toList, {{"Name","currency"},{"Value","rate_to_" & p[BaseCurrency]}}), Typed = Table.TransformColumnTypes(Renamed, {{"currency", type text}, {"rate_to_" & p[BaseCurrency], type number}}) in Typed else null,
    Fallback = #table({"currency", "rate_to_" & p[BaseCurrency]}, {{"USD", 1.0},{"EUR", 1.08},{"GBP", 1.27},{"JPY", 0.0066},{"CAD", 0.73}}),
    Output = if WebToTable <> null then WebToTable else Fallback
in
    Output
```

### Sales_With_Currency.pq
```m
let
    p = Parameters,
    Rates = ExchangeRates,
    Joined = Table.NestedJoin(Staging_Sales, {"currency"}, Rates, {"currency"}, "fx", JoinKind.LeftOuter),
    Expanded = Table.ExpandTableColumn(Joined, "fx", {"rate_to_" & p[BaseCurrency]}, {"rate_to_base"}),
    WithDefaultRate = Table.ReplaceValue(Expanded, null, 1.0, Replacer.ReplaceValue, {"rate_to_base"}),
    AddedAmountBase = Table.AddColumn(WithDefaultRate, "amount_" & Text.Lower(p[BaseCurrency]), each [amount] * [rate_to_base], type number)
in
    AddedAmountBase
```

### Sales_Summary.pq
```m
let
    p = Parameters,
    MergedDate = Table.NestedJoin(Sales_With_Currency, {"date"}, Dim_Calendar, {"Date"}, "d", JoinKind.LeftOuter),
    ExpandedDate = Table.ExpandTableColumn(MergedDate, "d", {"Year","Quarter","Month","MonthName","FiscalYear","FiscalPeriod"}, {"Year","Quarter","Month","MonthName","FiscalYear","FiscalPeriod"}),
    Grouped = Table.Group(ExpandedDate, {"Year","FiscalYear","FiscalPeriod","Month","MonthName","store","sku"}, {{"Orders", each List.NonNullCount([orderid]), Int64.Type}, {"Units", each List.Sum([qty]), Int64.Type}, {"Sales", each List.Sum([amount]), type number}, {"Sales_" & Text.Upper(p[BaseCurrency]), each List.Sum([amount_" & Text.Lower(p[BaseCurrency]) & "]), type number}}),
    Sorted = Table.Sort(Grouped, {{"FiscalYear", Order.Ascending}, {"FiscalPeriod", Order.Ascending}, {"store", Order.Ascending}, {"sku", Order.Ascending}})
in
    Sorted
```

---

## 🏁 Portfolio Talking Points
- Built a **parameterized ETL** using Power Query that ingests CSVs and normalizes schema.
- Implemented **robust error handling** and **type-safe transformations**.
- Designed a **Dim_Calendar** with fiscal logic.
- Created a **Sales Summary** fact table with currency normalization.
- Demonstrated **JSON web ingestion with offline fallback**.

---

### 👤 Author
**Rodger Roberts**  
[LinkedIn](https://www.linkedin.com/in/rodger-roberts/) | [GitHub](https://github.com/) | [Portfolio](https://github.com/)

