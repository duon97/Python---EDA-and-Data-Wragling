# E-Wallet Payment & Transaction Analysis: EDA and Data Wrangling

## I. Introduction

### 📖 What is this project about?

An e-wallet company wants to understand the status of its payments and transactions. The key goal is to understand payment and transaction statuses in order to improve operational efficiency, user experience and fraud detection.

### ❓ Business Questions

Part 1: Exploratory Data Analysis (EDA) integrates and refines the payment and product datasets, identifies and fixes missing values, duplicates and incorrect data types, and checks the integrity of the transaction data.

Part 2: Data Wrangling refines and analyzes the datasets to answer business questions: which products perform best, whether product ownership is consistent, which team is weakest, where refunds come from, and how transactions split by type (transfers, top-ups, withdrawals, payments, split bills).

### 📂 Dataset

Three CSV files:

Payment report (`payment_report.csv`, 919 rows)

| Field Name | Data Type |
|---|---|
| report_month | DATETIME |
| payment_group | STRING |
| product_id | INT |
| source_id | INT |
| volume | INT |

Product (`product.csv`, 492 rows)

| Field Name | Data Type |
|---|---|
| product_id | INT |
| category | STRING |
| team_own | STRING |

Transactions (`transactions.csv`, 1,324,002 rows)

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

### 1. Read data

**Read payment_report**

```python
payment_report = pd.read_csv('payment_report.csv', encoding='utf-8')
payment_report.head()
```

Result: The first 5 rows load correctly, 919 rows and 5 columns in total.

| | report_month | payment_group | product_id | source_id | volume |
|---|---|---|---|---|---|
| 0 | 2023-01 | payment | 12 | 45 | 624110375 |
| 1 | 2023-01 | payment | 17 | 45 | 335715113 |
| 2 | 2023-01 | payment | 18 | 45 | 737784466 |
| 3 | 2023-01 | payment | 19 | 45 | 120963069 |
| 4 | 2023-01 | payment | 20 | 45 | 319653158 |

**Read product**

```python
product = pd.read_csv('product.csv', encoding='utf-8')
product.head()
```

Result: The first 5 rows load correctly, 492 rows and 3 columns in total.

| | product_id | category | team_own |
|---|---|---|---|
| 0 | 17 | PXXXXXB | ASD |
| 1 | 18 | PXXXXXB | ASD |
| 2 | 20 | PXXXXXB | ASD |
| 3 | 287 | PXXXXXB | ASD |
| 4 | 372 | PXXXXXB | ASD |

**Read transactions**

```python
transactions = pd.read_csv('transactions.csv', encoding='utf-8')
transactions.head()
```

Result: The first 5 rows load correctly, 1,324,002 rows and 9 columns in total, and `sender_id` already contains NaN.

| | transaction_id | merchant_id | volume | transType | transStatus | sender_id | receiver_id | extra_info | timeStamp |
|---|---|---|---|---|---|---|---|---|---|
| 0 | 3002692434 | 5 | 100000 | 24 | 1 | 10199794.0 | 199794.0 | NaN | 1682932054455 |
| 1 | 3002692437 | 305 | 20000 | 2 | 1 | 14022211.0 | 14022211.0 | NaN | 1682932054912 |
| 2 | 3001960110 | 7255 | 48605 | 22 | 1 | NaN | 10530940.0 | NaN | 1682932055000 |
| 3 | 3002680710 | 2270 | 1500000 | 2 | 1 | 10059206.0 | 59206.0 | NaN | 1682932055622 |
| 4 | 3002680713 | 2275 | 90000 | 2 | 1 | 10004711.0 | 4711.0 | NaN | 1682932056197 |

### 2. Payment_enriched

**Merge payment_report with product**

```python
payment_enriched = pd.merge(payment_report, product, how='left', on='product_id')
payment_enriched.head()
```

Result: Left join keeps all 919 payment rows and adds `category` and `team_own`, 7 columns in total.

| | report_month | payment_group | product_id | source_id | volume | category | team_own |
|---|---|---|---|---|---|---|---|
| 0 | 2023-01 | payment | 12 | 45 | 624110375 | PXXXXXT | ASD |
| 1 | 2023-01 | payment | 17 | 45 | 335715113 | PXXXXXB | ASD |
| 2 | 2023-01 | payment | 18 | 45 | 737784466 | PXXXXXB | ASD |
| 3 | 2023-01 | payment | 19 | 45 | 120963069 | PXXXXXM2 | ASD |
| 4 | 2023-01 | payment | 20 | 45 | 319653158 | PXXXXXB | ASD |

**Check missing values**

```python
payment_enriched.isnull().sum()
```

Result: `category` and `team_own` each miss 22 rows, which belong to products 3, 1976 and 10033 (see Q2) -> Next step: No action, keep them as NaN.

| Column | Missing |
|---|---|
| report_month | 0 |
| payment_group | 0 |
| product_id | 0 |
| source_id | 0 |
| volume | 0 |
| category | 22 |
| team_own | 22 |

### 3. Transactions

**Check missing values**

```python
transactions.isnull().sum()
```

Result: `sender_id` misses 49,059 rows, `receiver_id` misses 164,795 rows and `extra_info` misses 1,317,907 rows -> Next step: ignore `extra_info`, delete rows missing both `sender_id` and `receiver_id`, then fill the remaining NaN.

| Column | Missing |
|---|---|
| transaction_id | 0 |
| merchant_id | 0 |
| volume | 0 |
| transType | 0 |
| transStatus | 0 |
| sender_id | 49,059 |
| receiver_id | 164,795 |
| extra_info | 1,317,907 |
| timeStamp | 0 |

**Delete rows missing both sender_id and receiver_id**

```python
null_both = transactions[transactions['sender_id'].isnull() & transactions['receiver_id'].isnull()]
transactions_cleaned = transactions.dropna(subset=['sender_id', 'receiver_id'], how='all')
```

Result: 2,587 rows are deleted, because they have neither a sender nor a receiver.

| | Rows |
|---|---|
| Before | 1,324,002 |
| After (`transactions_cleaned`) | 1,321,415 |

**Handle missing values**

```python
transactions_cleaned['sender_id'].fillna(-1, inplace=True)
transactions_cleaned['receiver_id'].fillna(-1, inplace=True)
transactions_cleaned.isnull().sum()
```

Result: The reason for the missing `sender_id` and `receiver_id` is unknown, so NaN is replaced by -1 and no missing value is left in them, while `extra_info` keeps its NaN (ignored).

| Column | Missing |
|---|---|
| transaction_id | 0 |
| merchant_id | 0 |
| volume | 0 |
| transType | 0 |
| transStatus | 0 |
| sender_id | 0 |
| receiver_id | 0 |
| extra_info | 1,315,320 |
| timeStamp | 0 |

**Check duplicate values**

```python
transactions_cleaned.info()
transactions_cleaned.nunique()
```

Result: `transaction_id` has 1,321,387 unique values in 1,321,415 rows, so 28 rows repeat a `transaction_id` -> Next step: drop the duplicates by `transaction_id`.

| | Count |
|---|---|
| Rows | 1,321,415 |
| Unique transaction_id | 1,321,387 |

**Handle duplicate values**

```python
transactions_final = transactions_cleaned.drop_duplicates('transaction_id')
```

Result: One row per `transaction_id` remains, 1,321,387 rows.

### 4. Incorrect data type and data value

**Check Dtype**

```python
payment_enriched.info()
```

Result: `report_month` and `payment_group` are object, the other columns have a suitable type -> Next step: convert `report_month` to datetime when it is needed (Q3).

```
RangeIndex: 919 entries, 0 to 918
Data columns (total 7 columns):
 #   Column         Non-Null Count  Dtype
---  ------         --------------  -----
 0   report_month   919 non-null    object
 1   payment_group  919 non-null    object
 2   product_id     919 non-null    int64
 3   source_id      919 non-null    int64
 4   volume         919 non-null    int64
 5   category       897 non-null    object
 6   team_own       897 non-null    object
dtypes: int64(3), object(4)
```

```python
transactions_final.info()
```

Result: `sender_id` and `receiver_id` are float64 because of NaN and the -1 fill, the other columns are int64 -> Next step: No action.

```
Index: 1321387 entries, 0 to 1324001
Data columns (total 9 columns):
 #   Column          Non-Null Count    Dtype
---  ------          --------------    -----
 0   transaction_id  1321387 non-null  int64
 1   merchant_id     1321387 non-null  int64
 2   volume          1321387 non-null  int64
 3   transType       1321387 non-null  int64
 4   transStatus     1321387 non-null  int64
 5   sender_id       1321387 non-null  float64
 6   receiver_id     1321387 non-null  float64
 7   extra_info      6095 non-null     object
 8   timeStamp       1321387 non-null  int64
dtypes: float64(2), int64(6), object(1)
```

**Check data values**

```python
payment_enriched.describe()
```

Result: `volume` ranges from 5,500 to about 13.8B with no zero or negative value -> Next step: No action.

| | product_id | source_id | volume |
|---|---|---|---|
| count | 919 | 919 | 919 |
| mean | 1192.52 | 44.88 | 1.978574e+08 |
| std | 1293.46 | 0.91 | 8.367595e+08 |
| min | 3 | 37 | 5.5e+03 |
| 25% | 640 | 45 | 1.25e+06 |
| 50% | 1059 | 45 | 7.982015e+06 |
| 75% | 1585 | 45 | 5.447599e+07 |
| max | 15067 | 45 | 1.383171e+10 |

```python
transactions_final.describe()
```

Result: `volume` is always positive (1 to 78.7M), but `transStatus` goes down to -1,333 and `receiver_id` goes down to -63 -> Next step: No action in this analysis, the negative values are kept as they are (the -1 in `sender_id` is the fill value).

| | volume | transType | transStatus | sender_id | receiver_id |
|---|---|---|---|---|---|
| count | 1.321387e+06 | 1.321387e+06 | 1.321387e+06 | 1.321387e+06 | 1.321387e+06 |
| mean | 2.382381e+05 | 6.934112 | -1.193622e+01 | 9.975608e+07 | 1.828956e+08 |
| min | 1 | 2 | -1333 | -1 | -63 |
| 25% | 1.0e+04 | 2 | 1 | 1.004134e+07 | 4.2014e+04 |
| 50% | 3.0e+04 | 2 | 1 | 1.057732e+07 | 3.531248e+06 |
| 75% | 1.0e+05 | 8 | 1 | 2.100212e+07 | 2.451698e+07 |
| max | 7.869148e+07 | 30 | 1 | 6.993439e+09 | 2.1e+10 |

## 📌 Data Wrangling

Questions 1 to 4 use `payment_enriched`, and questions 5 and 6 use `transactions`.

### 🔥 1. Top 3 product_id with the highest volume

```python
volume_by_product = payment_enriched.groupby('product_id', as_index=False).agg({'volume': 'sum'})
top_3_product = volume_by_product.nlargest(3, 'volume')
top_3_product
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

```python
unique_team_by_product = payment_enriched.groupby('product_id', as_index=False).agg({'team_own': 'nunique'})
unique_team_by_product[unique_team_by_product['team_own'] != 1]
```

Result: Only products 3, 1976 and 10033 break the rule, with 0 teams each.

| product_id | team_own (nunique) |
|---|---|
| 3 | 0 |
| 1976 | 0 |
| 10033 | 0 |

Insight:
- No product is owned by two or more teams, because no product returns a count above 1.
- The abnormal products are 3, 1976 and 10033. Their count is 0 because they have no team at all (they are missing from `product.csv`), not because they have several teams.
- These products make up the 22 rows with missing `category` and `team_own` found in EDA. The product table should be completed for them.

### 🔥 3. Find the team with the lowest performance since Q2.2023, and the category that contributes the least to that team

```python
# Edit date_time value
payment_enriched['report_month'] = pd.to_datetime(payment_enriched['report_month'])
payment_enriched['year'] = payment_enriched['report_month'].dt.year
payment_enriched['quarter'] = payment_enriched['report_month'].dt.quarter

# Filter data since Q2.2023
from_q2_2023 = payment_enriched[(payment_enriched['year'] >= 2023) & (payment_enriched['quarter'] >= 2)]

# Total volume by team, then the team with the lowest total volume
team_performance = from_q2_2023.groupby('team_own').agg({'volume': 'sum'})
lowest_performance_team = team_performance['volume'].idxmin()
print(f"The lowest performing team since Q2.2023 is: {lowest_performance_team}")

# Category that contributes the least to that team
data_team = from_q2_2023[from_q2_2023['team_own'] == lowest_performance_team]
category_contribute = data_team.groupby('category').agg({'volume': 'sum'})
lowest_category_contribute = category_contribute['volume'].idxmin()
print(f"Category that contributes the least to {lowest_performance_team} is: {lowest_category_contribute}")
```

Result: APS is the lowest performing team since Q2.2023, and PXXXXXE is the category that contributes the least to APS.

| Question | Answer |
|---|---|
| Lowest performing team since Q2.2023 | APS |
| Category contributing the least to APS | PXXXXXE |

Insight:
- The data ends in April 2023, so "since Q2.2023" covers a single month.
- Volume is summed per team first, then the weakest category is searched inside that team, which follows the question.
- The code prints the names only, so the gap between APS and the next team is not visible yet.
- Rows with no team (products 3, 1976, 10033) are excluded by the groupby, although product 1976 alone has a very large volume. The ranking may change once these products get an owner team.

### 🔥 4. Contribution of source_ids to refund transactions

```python
# Filter payment_group = refund
refund_payment = payment_enriched[payment_enriched['payment_group'] == 'refund']

# Total volume group by source_id
source_id_contribute = refund_payment.groupby('source_id')['volume'].sum()

# Highest contribution source_id
highest_source_id_contribute = source_id_contribute.idxmax()
print(f"Source ID with the highest contribution to refund transactions is: {highest_source_id_contribute}")
```

Result: Source 38 has the highest contribution to refund transactions.

| Question | Answer |
|---|---|
| source_id with the highest refund contribution | 38 |

Insight:
- Source 38 holds about 59% of the refund volume of product 1976 (36.5B), against about 15% for source 37 and about 26% for source 39.
- Refunds are concentrated in one source, which makes source 38 the first place to check for refund causes.

### 🔥 5. Define the type of each transaction

```python
def define_transaction_type(row):
    if row['transType'] == 2 and row['merchant_id'] == 1205:
        return 'Bank Transfer Transaction'
    elif row['transType'] == 2 and row['merchant_id'] == 2260:
        return 'Withdraw Money Transaction'
    elif row['transType'] == 2 and row['merchant_id'] == 2270:
        return 'Top Up Money Transaction'
    elif row['transType'] == 2:
        return 'Payment Transaction'
    elif row['transType'] == 8 and row['merchant_id'] == 2250:
        return 'Transfer Money Transaction'
    elif row['transType'] == 8:
        return 'Split Bill Transaction'
    else:
        return 'Invalid Transactions'

transactions['transaction_types'] = transactions.apply(define_transaction_type, axis=1)
transactions.head()
```

Result: Each transaction now has a `transaction_types`, and the first rows are Invalid, Payment and Top Up.

| | transaction_id | merchant_id | volume | transType | transStatus | sender_id | receiver_id | extra_info | timeStamp | transaction_types |
|---|---|---|---|---|---|---|---|---|---|---|
| 0 | 3002692434 | 5 | 100000 | 24 | 1 | 10199794.0 | 199794.0 | NaN | 1682932054455 | Invalid Transactions |
| 1 | 3002692437 | 305 | 20000 | 2 | 1 | 14022211.0 | 14022211.0 | NaN | 1682932054912 | Payment Transaction |
| 2 | 3001960110 | 7255 | 48605 | 22 | 1 | NaN | 10530940.0 | NaN | 1682932055000 | Invalid Transactions |
| 3 | 3002680710 | 2270 | 1500000 | 2 | 1 | 10059206.0 | 59206.0 | NaN | 1682932055622 | Top Up Money Transaction |
| 4 | 3002680713 | 2275 | 90000 | 2 | 1 | 10004711.0 | 4711.0 | NaN | 1682932056197 | Payment Transaction |

Insight:
- The conditions are checked from top to bottom, so the specific merchant rules must come before the general `transType == 2` rule. Reordering them would mislabel Bank Transfer, Withdraw and Top Up as Payment.
- Only transType 2 and 8 are valid. Other codes (22 and 24 appear in the first rows) become Invalid, about 220,700 of 1,324,002 rows (about 16.7%).
- This step runs on the raw `transactions` table, not on `transactions_final`, so duplicate `transaction_id` rows (at least 28) and rows missing both `sender_id` and `receiver_id` (2,587) are included, as well as every `transStatus` value.

### 🔥 6. Number of transactions, volume, senders and receivers per type (excluding invalid)

```python
# Excluding invalid transactions
filter_transactions = transactions[transactions['transaction_types'] != 'Invalid Transactions']

# Number of transactions, volume, senders and receivers group by transaction type
transaction_type_summary = filter_transactions.groupby('transaction_types').agg(
    num_transactions=('transaction_id', 'count'),
    total_volume=('volume', 'sum'),
    num_senders=('sender_id', 'nunique'),
    num_receivers=('receiver_id', 'nunique')).reset_index()
transaction_type_summary
```

Result: Payment has the most transactions, and Top Up has the highest volume.

| transaction_types | num_transactions | total_volume | num_senders | num_receivers |
|---|---|---|---|---|
| Bank Transfer Transaction | 37,879 | 50,605,806,190 | 23,156 | 9,271 |
| Payment Transaction | 398,677 | 71,851,515,181 | 139,583 | 113,298 |
| Split Bill Transaction | 1,376 | 4,901,464 | 1,323 | 572 |
| Top Up Money Transaction | 290,502 | 108,606,478,829 | 110,409 | 110,409 |
| Transfer Money Transaction | 341,177 | 37,033,171,492 | 39,021 | 34,585 |
| Withdraw Money Transaction | 33,725 | 23,418,181,420 | 24,814 | 24,814 |

Insight:
- By count, Payment (36.1%), Transfer Money (30.9%) and Top Up (26.3%) make up 93% of valid transactions. By volume, Top Up leads with 37.3%, then Payment 24.6% and Bank Transfer 17.4%.
- Average value per transaction differs a lot: Bank Transfer about 1.34M, Withdraw about 694K, Top Up about 374K, Payment about 180K, Transfer Money about 109K, Split Bill about 3.6K. Bank Transfer is only 3.4% of transactions but 17.4% of volume.
- Top Up and Withdraw have the same number of distinct senders and receivers (110,409 and 24,814). This suggests users act on their own wallet in these two types. It is a hypothesis to verify by checking `sender_id` against `receiver_id`.
- Split Bill is almost negligible (1,376 transactions, 0.002% of volume), so it has little influence on the business.
- Payment has more senders than receivers (139,583 vs 113,298), and Bank Transfer even more (23,156 vs 9,271), which means receivers are concentrated on fewer accounts.
- `nunique` ignores NaN, so rows without a sender or receiver are not counted in those two columns.

## III. Conclusion

| # | Question | Insight | Recommendation |
|---|---|---|---|
| 1 | Q1: Top 3 products | The top 3 products hold about half of total volume. Product 1976 alone is about 34% and 4.2 times the second product, but it is a refund product, so the ranking mixes payment and refund volume. | Rank products on payment volume only and track refund volume separately. Monitor concentration, since a few products drive most of the volume. |
| 2 | Q2: Product ownership | No product is owned by two teams. The only abnormal products are 3, 1976 and 10033, which have no owner team because they are missing from `product.csv`. | Assign an owner team to these 3 products and keep ownership in one maintained table. Flag any new `product_id` in the payment report that has no product record. |
| 3 | Q3: Weakest team | APS is the weakest team since Q2.2023 (April 2023 only), and PXXXXXE is its least contributing category. The gap with the other teams is not shown, and products without a team are excluded. | Print the volume per team to see how far APS is from the next team. Review whether category PXXXXXE needs support. Assign owners to products 3, 1976 and 10033 and rerun the ranking. |
| 4 | Q4: Refund sources | Source 38 holds about 59% of the refund volume of product 1976 (source 37 about 15%, source 39 about 26%). | Investigate source 38 first for refund causes. Track the refund-to-payment ratio per source every month. |
| 5 | Q5: Transaction types | About 16.7% of rows (about 220,700) are invalid, because their `transType` is not 2 or 8 (for example 22 and 24). | Check what the other `transType` codes mean before discarding them. If they are valid types missing from the mapping, their volume is left out of the analysis. |
| 6 | Q6: Transaction mix | Payment, Transfer Money and Top Up make up 93% of valid transactions. Top Up has the largest volume (37.3%). | Prioritize the reliability of Top Up, the main funding channel. |
| 7 | Q6: Bank Transfer | Bank Transfer is 3.4% of transactions but 17.4% of volume, with the highest average value (about 1.34M). | Apply closer monitoring and fraud rules to Bank Transfer, because each transaction is large. |
| 8 | Q6: User behavior | Top Up and Withdraw have identical sender and receiver counts (110,409 and 24,814), which suggests users act on their own wallet. This is a hypothesis. | Verify by checking `sender_id` against `receiver_id`. If confirmed, treat a mismatch in these two types as an anomaly signal for fraud detection. |
| 9 | Q6: Split Bill | Split Bill has 1,376 transactions and 0.002% of volume. | Check whether users can find the feature easily. If adoption stays low, deprioritize it. |

