<!-- iluozhiqiang/hermes-aimaxia:skills/merge-bank-statements/SKILL.md | https://github.com/iluozhiqiang/hermes-aimaxia/blob/HEAD/skills/merge-bank-statements/SKILL.md -->

---
name: merge-bank-statements
description: Merge and analyze bank statements from multiple sources (万里汇, ipaylinks, etc.) into unified financial data with deduplication and statistics
category: productivity
---

# Merge and Analyze Bank Statements

Process and merge bank statements from multiple sources (万里汇/WorldFirst, ipaylinks, etc.) into a unified dataset for financial analysis.

## Use Cases
- Merge bank statements from different banks/time periods
- Remove duplicate transactions by transaction ID
- Generate revenue/expense statistics by time period, bank, and currency
- Prepare data for audit or bookkeeping

## Steps

### 1. Load and Parse Data
```python
import pandas as pd

# Read Excel files
df1 = pd.read_excel("万里汇_2025.xlsx")
df2 = pd.read_excel("万里汇_2026.xlsx")
df3 = pd.read_excel("ipaylinks.xls", engine='openpyxl')

# Parse time - different banks have different formats
def parse_wlw_time(t):
    """Parse 万里汇 time format: 2025/12/31, 23:22:48 (GMT+08:00)"""
    if pd.isna(t):
        return pd.NaT
    try:
        t = str(t).replace(' (GMT+08:00)', '')
        return pd.to_datetime(t, format='%Y/%m/%d, %H:%M:%S')
    except:
        return pd.NaT

df1['交易时间'] = df1['交易创建时间'].apply(parse_wlw_time)
df2['交易时间'] = df2['交易创建时间'].apply(parse_wlw_time)
df3['交易时间'] = pd.to_datetime(df3['创建时间'], errors='coerce')
```

### 2. Merge and Deduplicate
```python
# Combine same-bank data
wlw = pd.concat([df1, df2], ignore_index=True)
wlw = wlw.drop_duplicates(subset=['交易号'], keep='first')

# Add account identifier
wlw['账户'] = '万里汇'
df3['账户'] = 'ipaylinks'

# Standardize column names
wlw_cols = ['交易时间', '动账类型', '描述', '币种', '借记金额', '贷记金额', '余额', '交易号', '收款人名称', '付款人名称', '电商平台/支付网关', '账户']
wlw_final = wlw[wlw_cols].copy()
wlw_final = wlw_final.rename(columns={
    '动账类型': '类型', '描述': '摘要', '借记金额': '支出', '贷记金额': '收入',
    '交易号': '订单号', '收款人名称': '收款人', '付款人名称': '付款人', '电商平台/支付网关': '平台'
})

# Do same for other banks...
all_data = pd.concat([wlw_final, df_other], ignore_index=True)
all_data = all_data.sort_values('交易时间')
```

### 3. Save Merged Data
```python
output_path = "/opt/data/财务数据/合并资金记录.xlsx"
all_data.to_excel(output_path, index=False)
```

### 4. Generate Statistics
```python
# Filter income
income = df[df['收入'].notna() & (df['收入'] > 0)]

# By bank + currency
for account in income['账户'].unique():
    acc_income = income[income['账户'] == account]
    by_currency = acc_income.groupby('币种')['收入'].sum()
    print(f"{account}: {by_currency.to_dict()}")

# By month
income['月份'] = income['交易时间'].dt.to_period('M')
monthly = income.groupby(['月份', '账户'])['收入'].sum().unstack()
```

## Handling Overlapping Data

When files have overlapping time periods:
1. Check for duplicate transaction IDs between files
2. Filter by date range to avoid overlap, OR
3. Use `drop_duplicates(subset=['交易号'], keep='last')` to keep latest version

```python
# Example: Remove old overlapping data
jan_feb_old = df2[(df2['交易时间'] >= '2026-01-01') & (df2['交易时间'] < '2026-03-01')]
march_new = df3  # New March data

wlw = pd.concat([df1, jan_feb_old, march_new], ignore_index=True)
wlw = wlw.drop_duplicates(subset=['交易号'], keep='last')
```

## Transaction Type Filtering

Different transaction types must be filtered differently:

| Type | Treatment | Column |
|------|-----------|--------|
| 收款 (Sales) | Count as income | 万里汇 |
| 平台收款 (Platform Collection) | Count as income | ipaylinks |
| 转账 (Transfer) | Supplier/personal payments | 排除 from 业务支出 |
| 货币兑换 (Currency Exchange) | Exclude from income | Not real revenue |
| 外汇兑换 (FX Exchange) | Exclude from income | Not real revenue |
| 归集 (Aggregation) | Exclude | Internal transfer |
| 单笔付款 | Supplier payment | ipaylinks |

### Correct Income Filter
```python
# Real sales income only
income = df[(df['收入'].notna()) & (df['收入'] > 0) & 
            (df['类型'].isin(['收款', '平台收款']))]
```

### Correct Supplier Payment Filter
```python
# Supplier transfers
payments = df[(df['支出'].notna()) & (df['支出'] > 0) & 
              (df['类型'].isin(['转账', '单笔付款']))]
```

## Notes
- 万里汇 uses unique transaction ID in '交易号' column for deduplication
- ipaylinks uses '业务订单号' as transaction ID
- Different banks have different time ranges - check before merging
- Currency codes differ: CNH (离岸人民币) vs CNY, etc.
- WebDAV connections may fail if server has SSL issues (check TLS compatibility)
- **Important**: Currency exchange (货币兑换) is NOT income - it's just currency conversion

## Output
- Merged Excel file with standardized columns
- Revenue/expense statistics by time period, bank, and currency
