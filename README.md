# E-Wallet Payment & Transaction Analysis: EDA and Data Wrangling

## I. Introduction

### 📖 What is this project about?

An e-wallet company wants to understand the status of its payments and transactions. The key goal is to understand payment and transaction statuses in order to improve operational efficiency, user experience and fraud detection.

### ❓ Business Questions

Part 1: Exploratory Data Analysis (EDA) integrates and refines the payment and product datasets, identifies and fixes missing values, duplicates and incorrect data types, and checks the integrity of the transaction data.

Part 2: Data Wrangling refines and analyzes the datasets to answer business questions: which products perform best, whether product ownership is consistent, which team is weakest, where refunds come from, and how transactions split by type (transfers, top-ups, withdrawals, payments, split bills).

### 📂 Dataset

Three CSV files:

Payment report (`payment_report.csv`)

| Field Name | Data Type |
|---|---|
| report_month | DATETIME |
| payment_group | STRING |
| product_id | INT |
| source_id | INT |
| volume | INT |

Product (`product.csv`)

| Field Name | Data Type |
|---|---|
| product_id | INT |
| category | STRING |
| team_own | STRING |

Transactions (`transactions.csv`)

| Field Name | Data Type |
|---|---|
| transaction_id | INT |
| merchant_id | INT |
| volume | INT |
| transType | INT |
| transStatus | INT |
| sender_id | INT |
| receiver_id | INT |
| extra_info | STRING |
| timeStamp | INT |

### 📌 Part 1: Exploratory Data Analysis

- Df `payment_enriched` (merge `payment_report.csv` with `product.csv`)
- Df `transactions`

### 📌 Part 2: Data Wrangling

Using `payment_report.csv` and `product.csv`

1. Top 3 product_ids with the highest volume.
2. Given that 1 product_id is owned by only 1 team, are there any abnormal products against this rule?
3. Find the team with the lowest performance (lowest volume) since Q2.2023, and the category that contributes the least to that team.
4. Find the contribution of source_ids to refund transactions (payment_group = 'refund'). Which source_id contributes the most?

Using `transactions.csv`

5. Define the type of each transaction (`transaction_type`):
   - transType = 2 & merchant_id = 1205: Bank Transfer Transaction
   - transType = 2 & merchant_id = 2260: Withdraw Money Transaction
   - transType = 2 & merchant_id = 2270: Top Up Money Transaction
   - transType = 2 & other merchant_id: Payment Transaction
   - transType = 8 & merchant_id = 2250: Transfer Money Transaction
   - transType = 8 & other merchant_id: Split Bill Transaction
   - All remaining cases: Invalid Transaction
6. For each transaction type (excluding invalid), find the number of transactions, volume, senders and receivers.

## II. EDA and Wrangling

Tools: Python (pandas, numpy) in Google Colab.

## 📌 EDA

### 1. Payment report

**Read data**

```python
payment_report = pd.read_csv('payment_report.csv', encoding='utf-8')
payment_report.head()
```

Result: The first 5 rows load correctly, and `report_month` is still text (2023-01).

| | report_month | payment_group | product_id | source_id | volume |
|---|---|---|---|---|---|
| 0 | 2023-01 | payment | 12 | 45 | 624110375 |
| 1 | 2023-01 | payment | 17 | 45 | 335715113 |
| 2 | 2023-01 | payment | 18 | 45 | 737784466 |
| 3 | 2023-01 | payment | 19 | 45 | 120963069 |
| 4 | 2023-01 | payment | 20 | 45 | 319653158 |

**Check Dtype**

```python
payment_report.info()
```

Result: 919 rows and 5 columns, `report_month` and `payment_group` are object -> Action: convert to datetime and string.

```
RangeIndex: 919 entries, 0 to 918
Data columns (total 5 columns):
 #   Column         Non-Null Count  Dtype
---  ------         --------------  -----
 0   report_month   919 non-null    object
 1   payment_group  919 non-null    object
 2   product_id     919 non-null    int64
 3   source_id      919 non-null    int64
 4   volume         919 non-null    int64
dtypes: int64(3), object(2)
```

**Convert Dtype**

```python
payment_report['report_month'] = pd.to_datetime(payment_report['report_month'], format='%Y-%m')
payment_report['payment_group'] = payment_report['payment_group'].astype('string')
payment_report.info()
```

Result: Dtype is now correct -> Next step: No action.

```
 #   Column         Non-Null Count  Dtype
---  ------         --------------  -----
 0   report_month   919 non-null    datetime64[ns]
 1   payment_group  919 non-null    string
 2   product_id     919 non-null    int64
 3   source_id      919 non-null    int64
 4   volume         919 non-null    int64
dtypes: datetime64[ns](1), int64(3), string(1)
```

**Check missing values**

```python
payment_report.isnull().sum()
```

Result: Missing: 0 in every column -> Next step: No action.

| Column | Missing |
|---|---|
| report_month | 0 |
| payment_group | 0 |
| product_id | 0 |
| source_id | 0 |
| volume | 0 |

**Check duplicate values**

```python
duplicates_payment = payment_report[payment_report.duplicated(
    subset=['report_month','payment_group','product_id','source_id'], keep=False)]
duplicates_payment.head()
```

Result: The table has no primary key, so `report_month + payment_group + product_id + source_id` is used as the key, and it repeats with different volumes (for example product 1976, source 39) while `duplicated()` over all columns returns 0 -> Next step: group by the key and sum `volume`.

| | report_month | payment_group | product_id | source_id | volume |
|---|---|---|---|---|---|
| 194 | 2023-01-01 | refund | 1976 | 39 | 443387276 |
| 195 | 2023-01-01 | refund | 1976 | 39 | 111281678 |
| 409 | 2023-02-01 | refund | 1976 | 39 | 111500 |
| 412 | 2023-02-01 | refund | 1976 | 39 | 910819764 |
| 413 | 2023-02-01 | refund | 1976 | 39 | 2675894726 |

**Handle duplicate values**

```python
cleaning_payment = payment_report.groupby(
    ['report_month','payment_group','product_id','source_id'], as_index=False
).agg({'volume': 'sum', 'payment_group': 'first', 'source_id': 'first'})
cleaning_payment.head()
```

Result: Duplicate keys are merged and their volumes summed, 919 rows -> 914 rows.

| | report_month | product_id | volume | payment_group | source_id |
|---|---|---|---|---|---|
| 0 | 2023-01-01 | 12 | 624110375 | payment | 45 |
| 1 | 2023-01-01 | 17 | 335715113 | payment | 45 |
| 2 | 2023-01-01 | 18 | 737784466 | payment | 45 |
| 3 | 2023-01-01 | 19 | 120963069 | payment | 45 |
| 4 | 2023-01-01 | 20 | 319653158 | payment | 45 |

**Verify product_id = 1976 after cleaning**

```python
cleaning_payment[cleaning_payment['product_id'] == 1976]
```

Result: Each month now has exactly one row per source (37, 38, 39) -> duplicates resolved.

| | report_month | product_id | volume | payment_group | source_id |
|---|---|---|---|---|---|
| 192 | 2023-01-01 | 1976 | 852173308 | refund | 37 |
| 193 | 2023-01-01 | 1976 | 3520890417 | refund | 38 |
| 194 | 2023-01-01 | 1976 | 554668954 | refund | 39 |
| 408 | 2023-02-01 | 1976 | 1794436949 | refund | 37 |
| 409 | 2023-02-01 | 1976 | 7522440976 | refund | 38 |
| 410 | 2023-02-01 | 1976 | 3586825990 | refund | 39 |
| 631 | 2023-03-01 | 1976 | 2962187382 | refund | 37 |
| 632 | 2023-03-01 | 1976 | 11652415177 | refund | 38 |
| 633 | 2023-03-01 | 1976 | 6392207104 | refund | 39 |
| 911 | 2023-04-01 | 1976 | 3542271587 | refund | 37 |
| 912 | 2023-04-01 | 1976 | 13831708189 | refund | 38 |
| 913 | 2023-04-01 | 1976 | 5585357614 | refund | 39 |

**Final payment report**

```python
final_payment_report = cleaning_payment[['report_month','payment_group','product_id','source_id','volume']]
final_payment_report.head()
```

Result: Original column order restored, 914 clean rows.

| | report_month | payment_group | product_id | source_id | volume |
|---|---|---|---|---|---|
| 0 | 2023-01-01 | payment | 12 | 45 | 624110375 |
| 1 | 2023-01-01 | payment | 17 | 45 | 335715113 |
| 2 | 2023-01-01 | payment | 18 | 45 | 737784466 |
| 3 | 2023-01-01 | payment | 19 | 45 | 120963069 |
| 4 | 2023-01-01 | payment | 20 | 45 | 319653158 |

### 2. Product

**Read data**

```python
product = pd.read_csv('product.csv', encoding='utf-8')
product.head()
```

Result: The first 5 rows load correctly, each product belongs to one category and one team.

| | product_id | category | team_own |
|---|---|---|---|
| 0 | 17 | PXXXXXB | ASD |
| 1 | 18 | PXXXXXB | ASD |
| 2 | 20 | PXXXXXB | ASD |
| 3 | 287 | PXXXXXB | ASD |
| 4 | 372 | PXXXXXB | ASD |

**Check Dtype**

```python
product.info()
```

Result: 492 rows and 3 columns, `category` and `team_own` are object -> Action: convert to string.

```
RangeIndex: 492 entries, 0 to 491
Data columns (total 3 columns):
 #   Column      Non-Null Count  Dtype
---  ------      --------------  -----
 0   product_id  492 non-null    int64
 1   category    492 non-null    object
 2   team_own    492 non-null    object
dtypes: int64(1), object(2)
```

**Convert Dtype**

```python
product['category'] = product['category'].astype('string')
product['team_own'] = product['team_own'].astype('string')
product.info()
```

Result: Dtype is now correct -> Next step: No action.

```
 #   Column      Non-Null Count  Dtype
---  ------      --------------  -----
 0   product_id  492 non-null    int64
 1   category    492 non-null    string
 2   team_own    492 non-null    string
dtypes: int64(1), string(2)
```

**Check missing values**

```python
product.isnull().sum()
```

Result: Missing: 0 in every column -> Next step: No action.

| Column | Missing |
|---|---|
| product_id | 0 |
| category | 0 |
| team_own | 0 |

**Check duplicate values**

```python
duplicates_product = product[product.duplicated(subset=['product_id'], keep=False)]
duplicates_product.head()
```

Result: Primary key `product_id` has no duplicate -> Next step: No action.

```
Empty DataFrame
Columns: [product_id, category, team_own]
Index: []
```

### 3. Payment_enriched

**Merge payment_report with product**

```python
payment_enriched = final_payment_report.merge(product, on='product_id', how='left')
payment_enriched.head()
```

Result: Left join keeps all 914 payment rows and adds `category` and `team_own`.

| | report_month | payment_group | product_id | source_id | volume | category | team_own |
|---|---|---|---|---|---|---|---|
| 0 | 2023-01-01 | payment | 12 | 45 | 624110375 | PXXXXXT | ASD |
| 1 | 2023-01-01 | payment | 17 | 45 | 335715113 | PXXXXXB | ASD |
| 2 | 2023-01-01 | payment | 18 | 45 | 737784466 | PXXXXXB | ASD |
| 3 | 2023-01-01 | payment | 19 | 45 | 120963069 | PXXXXXM2 | ASD |
| 4 | 2023-01-01 | payment | 20 | 45 | 319653158 | PXXXXXB | ASD |

**Check Dtype**

```python
payment_enriched.info()
```

Result: Dtype is correct -> No action, but `category` and `team_own` have 897 non-null out of 914 rows -> check missing values.

```
RangeIndex: 914 entries, 0 to 913
Data columns (total 7 columns):
 #   Column         Non-Null Count  Dtype
---  ------         --------------  -----
 0   report_month   914 non-null    datetime64[ns]
 1   payment_group  914 non-null    string
 2   product_id     914 non-null    int64
 3   source_id      914 non-null    int64
 4   volume         914 non-null    int64
 5   category       897 non-null    string
 6   team_own       897 non-null    string
dtypes: datetime64[ns](1), int64(3), string(3)
```

**Check missing values**

```python
payment_enriched.isnull().sum()
```

Result: `category` and `team_own` each miss 17 rows, the other columns are complete.

| Column | Missing |
|---|---|
| report_month | 0 |
| payment_group | 0 |
| product_id | 0 |
| source_id | 0 |
| volume | 0 |
| category | 17 |
| team_own | 17 |

**Inspect missing rows**

```python
miss_payment_enriched = payment_enriched[payment_enriched['category'].isna() | payment_enriched['team_own'].isna()]
miss_payment_enriched.head()
```

Result: The missing rows are products in the payment report but not in `product.csv` (for example 10033 and 1976) -> Next step: replace NA with 'Unknown' and keep the rows, because they carry real volume.

| | report_month | payment_group | product_id | source_id | volume | category | team_own |
|---|---|---|---|---|---|---|---|
| 190 | 2023-01-01 | payment | 10033 | 45 | 5500 | NA | NA |
| 192 | 2023-01-01 | refund | 1976 | 37 | 852173308 | NA | NA |
| 193 | 2023-01-01 | refund | 1976 | 38 | 3520890417 | NA | NA |
| 194 | 2023-01-01 | refund | 1976 | 39 | 554668954 | NA | NA |
| 405 | 2023-02-01 | payment | 10033 | 45 | 114050 | NA | NA |

**Handle missing values**

```python
cleaned_payment_enriched = payment_enriched.replace(np.nan, 'Unknown')
cleaned_payment_enriched.tail()
```

Result: Missing values are replaced by 'Unknown', no row is lost.

| | report_month | payment_group | product_id | source_id | volume | category | team_own |
|---|---|---|---|---|---|---|---|
| 909 | 2023-04-01 | payment | 10039 | 45 | 7872489 | PXXXXXF | ASL |
| 910 | 2023-04-01 | payment | 15067 | 45 | 1504000 | PXXXXXR | ASL |
| 911 | 2023-04-01 | refund | 1976 | 37 | 3542271587 | Unknown | Unknown |
| 912 | 2023-04-01 | refund | 1976 | 38 | 13831708189 | Unknown | Unknown |
| 913 | 2023-04-01 | refund | 1976 | 39 | 5585357614 | Unknown | Unknown |

**Check duplicate values**

```python
duplicates_payment_enriched = cleaned_payment_enriched[cleaned_payment_enriched.duplicated(
    subset=['report_month','payment_group','product_id','source_id'], keep=False)]
duplicates_payment_enriched.head()
```

Result: No duplicate values after the merge -> Next step: No action.

```
Empty DataFrame
Columns: [report_month, payment_group, product_id, source_id, volume, category, team_own]
Index: []
```

**Final payment_enriched**

```python
final_payment_enriched = cleaned_payment_enriched[
    ['report_month','payment_group','product_id','source_id','volume','category','team_own']]
final_payment_enriched.head()
```

Result: 914 rows and 7 columns, clean.

| | report_month | payment_group | product_id | source_id | volume | category | team_own |
|---|---|---|---|---|---|---|---|
| 0 | 2023-01-01 | payment | 12 | 45 | 624110375 | PXXXXXT | ASD |
| 1 | 2023-01-01 | payment | 17 | 45 | 335715113 | PXXXXXB | ASD |
| 2 | 2023-01-01 | payment | 18 | 45 | 737784466 | PXXXXXB | ASD |
| 3 | 2023-01-01 | payment | 19 | 45 | 120963069 | PXXXXXM2 | ASD |
| 4 | 2023-01-01 | payment | 20 | 45 | 319653158 | PXXXXXB | ASD |

### 4. Transactions

**Read data**

```python
transactions = pd.read_csv('transactions.csv', encoding='utf-8')
transactions.head()
```

Result: The first 5 rows load correctly, `sender_id` and `receiver_id` are floats and contain NaN.

| | transaction_id | merchant_id | volume | transType | transStatus | sender_id | receiver_id | extra_info | timeStamp |
|---|---|---|---|---|---|---|---|---|---|
| 0 | 3002692434 | 5.0 | 100000.0 | 24.0 | 1.0 | 10199794.0 | 199794.0 | NaN | 1.682932e+12 |
| 1 | 3002692437 | 305.0 | 20000.0 | 2.0 | 1.0 | 14022211.0 | 14022211.0 | NaN | 1.682932e+12 |
| 2 | 3001960110 | 7255.0 | 48605.0 | 22.0 | 1.0 | NaN | 10530940.0 | NaN | 1.682932e+12 |
| 3 | 3002680710 | 2270.0 | 1500000.0 | 2.0 | 1.0 | 10059206.0 | 59206.0 | NaN | 1.682932e+12 |
| 4 | 3002680713 | 2275.0 | 90000.0 | 2.0 | 1.0 | 10004711.0 | 4711.0 | NaN | 1.682932e+12 |

**Check Dtype and missing values**

```python
transactions.info()
```

Result: 159,689 rows and 9 columns, `sender_id` and `receiver_id` are float -> convert to int, `extra_info` is object -> convert to string, `sender_id` (5,857 missing) and `receiver_id` (18,531 missing) -> replace NaN with 0, and `extra_info` has only 565 non-null -> ignore.

```
RangeIndex: 159689 entries, 0 to 159688
Data columns (total 9 columns):
 #   Column          Non-Null Count   Dtype
---  ------          --------------   -----
 0   transaction_id  159689 non-null  int64
 1   merchant_id     159688 non-null  float64
 2   volume          159688 non-null  float64
 3   transType       159688 non-null  float64
 4   transStatus     159688 non-null  float64
 5   sender_id       153832 non-null  float64
 6   receiver_id     141158 non-null  float64
 7   extra_info      565 non-null     object
 8   timeStamp       159688 non-null  float64
dtypes: float64(7), int64(1), object(1)
```

**Convert Dtype and handle missing values**

```python
transactions['sender_id'] = transactions['sender_id'].fillna(0).astype('int')
transactions['receiver_id'] = transactions['receiver_id'].fillna(0).astype('int')
transactions['extra_info'] = transactions['extra_info'].astype('string')
transactions.info()
```

Result: `sender_id` and `receiver_id` are now int64 with no missing values, `extra_info` is string.

```
 #   Column          Non-Null Count   Dtype
---  ------          --------------   -----
 0   transaction_id  159689 non-null  int64
 1   merchant_id     159688 non-null  float64
 2   volume          159688 non-null  float64
 3   transType       159688 non-null  float64
 4   transStatus     159688 non-null  float64
 5   sender_id       159689 non-null  int64
 6   receiver_id     159689 non-null  int64
 7   extra_info      565 non-null     string
 8   timeStamp       159688 non-null  float64
dtypes: float64(5), int64(3), string(1)
```

**Check duplicate values**

```python
dup_transaction = transactions[transactions.duplicated(subset=['transaction_id'], keep=False)]
dup_transaction.head()
```

Result: Primary key `transaction_id` is duplicated and the rows shown are identical in every column -> Action: drop.

| | transaction_id | merchant_id | volume | transType | transStatus | sender_id | receiver_id | extra_info | timeStamp |
|---|---|---|---|---|---|---|---|---|---|
| 149734 | 3000871152 | 75.0 | 193000.0 | 2.0 | 1.0 | 17027174 | 0 | NA | 1.682989e+12 |
| 149735 | 3000871152 | 75.0 | 193000.0 | 2.0 | 1.0 | 17027174 | 0 | NA | 1.682989e+12 |
| 149738 | 3000871161 | 2270.0 | 50000.0 | 2.0 | 1.0 | 10009352 | 9352 | NA | 1.682989e+12 |
| 149739 | 3000871161 | 2270.0 | 50000.0 | 2.0 | 1.0 | 10009352 | 9352 | NA | 1.682989e+12 |
| 149752 | 3000687030 | 2250.0 | 165000.0 | 8.0 | 1.0 | 10074844 | 38548300 | NA | 1.682989e+12 |

**Handle duplicate values**

```python
no_dup_transactions = transactions.drop_duplicates(subset='transaction_id', ignore_index=False)
no_dup_transactions.head()
```

Result: Duplicate transactions are dropped, one row per `transaction_id`.

| | transaction_id | merchant_id | volume | transType | transStatus | sender_id | receiver_id | extra_info | timeStamp |
|---|---|---|---|---|---|---|---|---|---|
| 0 | 3002692434 | 5.0 | 100000.0 | 24.0 | 1.0 | 10199794 | 199794 | NA | 1.682932e+12 |
| 1 | 3002692437 | 305.0 | 20000.0 | 2.0 | 1.0 | 14022211 | 14022211 | NA | 1.682932e+12 |
| 2 | 3001960110 | 7255.0 | 48605.0 | 22.0 | 1.0 | 0 | 10530940 | NA | 1.682932e+12 |
| 3 | 3002680710 | 2270.0 | 1500000.0 | 2.0 | 1.0 | 10059206 | 59206 | NA | 1.682932e+12 |
| 4 | 3002680713 | 2275.0 | 90000.0 | 2.0 | 1.0 | 10004711 | 4711 | NA | 1.682932e+12 |

## 📌 Data Wrangling

**Base table: payment report joined with product**

```python
Data_wrangling = payment_report.merge(product, on='product_id', how='left')
Data_wrangling.head()
```

Result: Every payment row is kept and joined with its `category` and `team_own`.

| | report_month | payment_group | product_id | source_id | volume | category | team_own |
|---|---|---|---|---|---|---|---|
| 0 | 2023-01-01 | payment | 12 | 45 | 624110375 | PXXXXXT | ASD |
| 1 | 2023-01-01 | payment | 17 | 45 | 335715113 | PXXXXXB | ASD |
| 2 | 2023-01-01 | payment | 18 | 45 | 737784466 | PXXXXXB | ASD |
| 3 | 2023-01-01 | payment | 19 | 45 | 120963069 | PXXXXXM2 | ASD |
| 4 | 2023-01-01 | payment | 20 | 45 | 319653158 | PXXXXXB | ASD |

Note: built from `payment_report` before the duplicate groupby (919 rows). Totals are not affected, because the duplicate keys are summed anyway.

### 🔥 1. Top 3 product_id with the highest volume

```python
Data_wrangling.groupby('product_id')['volume'].sum().sort_values(ascending=False).head(3)
```

Result: Product 1976 has the highest volume, followed by 429 and 372.

| product_id | volume |
|---|---|
| 1976 | 61,797,583,647 |
| 429 | 14,667,676,567 |
| 372 | 13,713,658,515 |

Insight:
- The top 3 products hold about half of the total volume (about 90.2B of roughly 181.8B).
- Product 1976 alone is about 34% of total volume and 4.2 times the second product.
- Product 1976 has only refund rows and no team in `product.csv`. The ranking mixes payment and refund volume, so the top product is a refund product, not a best-selling product. A ranking of payment volume only would be a better measure of product performance.

### 🔥 2. Given that 1 product_id is owned by only 1 team, are there any abnormal products against this rule?

**Count distinct teams per product**

```python
Data_wrangling['product_team_own'] = (
    Data_wrangling.groupby(['product_id','source_id'])['team_own'].transform('nunique')
)
Data_wrangling.head()
```

Result: The new column `product_team_own` is 1 for the first rows, which follows the rule.

| | report_month | payment_group | product_id | source_id | volume | category | team_own | product_team_own |
|---|---|---|---|---|---|---|---|---|
| 0 | 2023-01-01 | payment | 12 | 45 | 624110375 | PXXXXXT | ASD | 1 |
| 1 | 2023-01-01 | payment | 17 | 45 | 335715113 | PXXXXXB | ASD | 1 |
| 2 | 2023-01-01 | payment | 18 | 45 | 737784466 | PXXXXXB | ASD | 1 |
| 3 | 2023-01-01 | payment | 19 | 45 | 120963069 | PXXXXXM2 | ASD | 1 |
| 4 | 2023-01-01 | payment | 20 | 45 | 319653158 | PXXXXXB | ASD | 1 |

**Filter products that break the rule**

```python
Non_Team = Data_wrangling[Data_wrangling['product_team_own'] != 1].reset_index()
Non_Team.groupby('product_id')['team_own'].nunique()
```

Result: Products 3, 1976 and 10033 break the rule, with 0 teams each.

| product_id | nunique team_own |
|---|---|
| 3 | 0 |
| 1976 | 0 |
| 10033 | 0 |

Insight:
- No product is owned by two or more teams. This is expected, because `product_id` is unique in `product.csv`.
- The abnormal products are 3, 1976 and 10033. Their count is 0 because they have no team at all (they are missing from `product.csv`), not because they have several teams.
- These are the same rows replaced with 'Unknown' in EDA. The product table should be completed for them.

### 🔥 3. Find the team with the lowest performance since Q2.2023, and the category that contributes the least to that team

```python
Data_wrangling[Data_wrangling['report_month'] >= '2023-04-01'] \
    .groupby(['team_own','category'])['volume'].sum() \
    .sort_values(ascending=True).head(1)
```

Result: The smallest team-category pair since Q2.2023 is ASD / PXXXXXP at 300,000.

| team_own | category | volume |
|---|---|---|
| ASD | PXXXXXP | 300,000 |

Insight:
- The data ends in April 2023, so "since Q2.2023" covers a single month.
- The smallest team-category combination is ASD / PXXXXXP at 300,000, a negligible amount next to the monthly volumes of other categories.
- This query ranks team-category pairs. It does not first sum volume per team to find the weakest team, and rows with no team (`Unknown`) are excluded by the groupby. A team-level total is needed to confirm that ASD is the lowest team.

### 🔥 4. Contribution of source_ids to refund transactions

```python
Data_wrangling[Data_wrangling['payment_group'] == 'refund'] \
    .groupby('source_id')['volume'].sum() \
    .sort_values(ascending=False).head(1)
```

Result: Source 38 has the highest refund volume at 36,527,454,759.

| source_id | volume |
|---|---|
| 38 | 36,527,454,759 |

Insight:
- Source 38 contributes the most refund volume, about 59% of the refund volume of product 1976 (source 37 about 15%, source 39 about 26%).
- Refunds are concentrated in one source, which makes source 38 the first place to check for refund causes.

### 🔥 5. Define the type of each transaction

```python
transactions = pd.read_csv('transactions.csv')

def transaction_type(row):
    if row['transType'] == 2 and row['merchant_id'] == 1205:
        return 'Bank Transfer Transaction'
    elif row['transType'] == 2 and row['merchant_id'] == 2260:
        return 'Withdraw Money Transaction'
    elif row['transType'] == 2 and row['merchant_id'] == 2270:
        return 'Top Up Money Transaction'
    elif row['transType'] == 2 and row['merchant_id'] not in [1205, 2260, 2270]:
        return 'Payment Transaction'
    elif row['transType'] == 8 and row['merchant_id'] == 2250:
        return 'Transfer Money Transaction'
    elif row['transType'] == 8:
        return 'Split Bill Transaction'
    else:
        return 'Invalid Transaction'

transactions['Type'] = transactions.apply(transaction_type, axis=1)
transactions.head()
```

Result: Each transaction now has a `Type`, and the first rows are Invalid, Payment and Top Up.

| | transaction_id | merchant_id | volume | transType | transStatus | sender_id | receiver_id | extra_info | timeStamp | Type |
|---|---|---|---|---|---|---|---|---|---|---|
| 0 | 3002692434 | 5 | 100000.0 | 24.0 | 1.0 | 10199794.0 | 199794.0 | NaN | 1.682932e+12 | Invalid Transaction |
| 1 | 3002692437 | 305 | 20000.0 | 2.0 | 1.0 | 14022211.0 | 14022211.0 | NaN | 1.682932e+12 | Payment Transaction |
| 2 | 3001960110 | 7255 | 48605.0 | 22.0 | 1.0 | NaN | 10530940.0 | NaN | 1.682932e+12 | Invalid Transaction |
| 3 | 3002680710 | 2270 | 1500000.0 | 2.0 | 1.0 | 10059206.0 | 59206.0 | NaN | 1.682932e+12 | Top Up Money Transaction |
| 4 | 3002680713 | 2275 | 90000.0 | 2.0 | 1.0 | 10004711.0 | 4711.0 | NaN | 1.682932e+12 | Payment Transaction |

Insight:
- The conditions are checked from top to bottom, so the specific merchant rules must come before the "other merchant" rules. Reordering them would mislabel Bank Transfer, Withdraw and Top Up as Payment.
- Only transType 2 and 8 are valid. Other codes (22 and 24 appear in the first rows) and the empty row become Invalid, about 11,400 of 159,689 rows (about 7%).
- This step reads the raw file, so duplicate `transaction_id` rows and all `transStatus` values are included. Part 2 counts are therefore not limited to successful, de-duplicated transactions.

### 🔥 6. Number of transactions, volume, senders and receivers per type (excluding invalid)

```python
transactions[transactions['Type'] != 'Invalid Transaction'].groupby('Type').agg({
    'transaction_id': 'count',
    'volume': 'sum',
    'sender_id': 'nunique',
    'receiver_id': 'nunique'
})
```

Result: Payment has the most transactions, and Top Up has the highest volume.

| Type | transaction_id | volume | sender_id | receiver_id |
|---|---|---|---|---|
| Bank Transfer Transaction | 5185 | 7.440799e+09 | 4338 | 1743 |
| Payment Transaction | 52622 | 8.667976e+09 | 37092 | 27020 |
| Split Bill Transaction | 171 | 6.398060e+05 | 166 | 110 |
| Top Up Money Transaction | 39284 | 1.422743e+10 | 28411 | 28411 |
| Transfer Money Transaction | 46710 | 5.098194e+09 | 11853 | 8212 |
| Withdraw Money Transaction | 4327 | 2.766309e+09 | 4051 | 4051 |

Insight:
- By count, Payment (35.5%), Transfer Money (31.5%) and Top Up (26.5%) make up 93% of valid transactions. By volume, Top Up leads with 37.2%, then Payment 22.7% and Bank Transfer 19.5%.
- Average value per transaction differs a lot: Bank Transfer about 1.44M, Withdraw about 639K, Top Up about 362K, Payment about 165K, Transfer Money about 109K, Split Bill about 3.7K. Bank Transfer is only 3.5% of transactions but 19.5% of volume.
- Top Up and Withdraw have the same number of distinct senders and receivers (28,411 and 4,051). This suggests users act on their own wallet in these two types. It is a hypothesis to verify by checking `sender_id` against `receiver_id`.
- Split Bill is almost negligible (171 transactions, 0.002% of volume), so it has little influence on the business.
- Payment has many more senders than receivers (37,092 vs 27,020), and Bank Transfer even more (4,338 vs 1,743), which means receivers are concentrated on fewer accounts.
- `sender_id` and `receiver_id` are counted with NaN excluded here, because the raw file is re-read in this step.

## III. Conclusion

| # | Question | Insight | Recommendation |
|---|---|---|---|
| 1 | Q1: Top 3 products | The top 3 products hold about half of total volume. Product 1976 alone is about 34% and 4.2 times the second product, but it is a refund product, so the ranking mixes payment and refund volume. | Rank products on payment volume only and track refund volume separately. Monitor concentration, since a few products drive most of the volume. |
| 2 | Q2: Product ownership | No product is owned by two teams. The only abnormal products are 3, 1976 and 10033, which have no owner team because they are missing from `product.csv`. | Assign an owner team to these 3 products and keep ownership in one maintained table. Flag any new `product_id` in the payment report that has no product record. |
| 3 | Q3: Weakest team | In April 2023 the smallest team-category pair is ASD / PXXXXXP at 300,000. The query ranks pairs, not teams, so the weakest team is not confirmed yet. | Sum volume per team first and confirm the weakest team. Then review whether category PXXXXXP needs support or can be dropped. |
| 4 | Q4: Refund sources | Source 38 holds about 59% of the refund volume of product 1976 (source 37 about 15%, source 39 about 26%). | Investigate source 38 first for refund causes. Track the refund-to-payment ratio per source every month. |
| 5 | Q5: Transaction types | About 7% of rows (about 11,400) are invalid, because their `transType` is not 2 or 8 (for example 22 and 24) or the row is empty. | Check what `transType` 22 and 24 mean before discarding them. If they are valid types missing from the mapping, their volume is left out of the analysis. |
| 6 | Q6: Transaction mix | Payment, Transfer Money and Top Up make up 93% of valid transactions. Top Up has the largest volume (37%). | Prioritize the reliability of Top Up, the main funding channel. |
| 7 | Q6: Bank Transfer | Bank Transfer is 3.5% of transactions but 19.5% of volume, with the highest average value (about 1.44M). | Apply closer monitoring and fraud rules to Bank Transfer, because each transaction is large. |
| 8 | Q6: User behavior | Top Up and Withdraw have identical sender and receiver counts (28,411 and 4,051), which suggests users act on their own wallet. This is a hypothesis. | Verify by checking `sender_id` against `receiver_id`. If confirmed, treat a mismatch in these two types as an anomaly signal for fraud detection. |
| 9 | Q6: Split Bill | Split Bill has 171 transactions and 0.002% of volume. | Check whether users can find the feature easily. If adoption stays low, deprioritize it. |

