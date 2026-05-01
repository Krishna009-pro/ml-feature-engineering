# My Understanding — How to Read `df.describe()`

`df.describe()` gives a **statistical summary of all numerical columns** in any dataset.
Use it as your **first diagnostic tool** after loading data.

---

## The 8 Statistics — What Each One Tells You

| Statistic | What it means | What to check |
|---|---|---|
| `count` | Number of non-null values | Is it less than total rows? → Missing values exist |
| `mean` | Average of all values | Compare with median — are they close or far apart? |
| `std` | How spread out values are from the mean | High std = high variability |
| `min` | Smallest value | Is it realistic? Could it be an error or outlier? |
| `25%` | 25th percentile (Q1) | 25% of data falls below this value |
| `50%` | Median — the true middle value | Compare with mean to detect skew |
| `75%` | 75th percentile (Q3) | 75% of data falls below this value |
| `max` | Largest value | Is it realistic? Could it be an outlier? |

---

## Rules to Apply on Every Column

### 1. Check for Missing Values → `count`
```
If count < total_rows  →  that column has missing values
Missing % = (total_rows - count) / total_rows * 100
```
| Missing % | Severity | Typical Action |
|---|---|---|
| < 5% | Low | Safe to fill with mean/median/mode |
| 5% – 20% | Medium | Fill carefully (use grouped median or model-based imputation) |
| 20% – 50% | High | Fill with caution, consider creating a "was_missing" flag column |
| > 50% | Very High | Consider dropping the column entirely |

---

### 2. Detect Skewness → `mean` vs `50%` (median)

```
mean ≈ median       →  Symmetric distribution (Normal) — safe to use as-is
mean >> median      →  Right-skewed (positive skew) — a few very large values pulling mean up
mean << median      →  Left-skewed (negative skew) — a few very small values pulling mean down
```

| Situation | What to do |
|---|---|
| Right-skewed (mean >> median) | Apply log transform: `np.log1p(column)` |
| Left-skewed (mean << median) | Apply square root or reflect + log |
| Symmetric | No transformation needed |

---

### 3. Detect Outliers → `min`, `max`, `std`

```
IQR = 75% - 25%
Lower fence = 25% - 1.5 * IQR
Upper fence = 75% + 1.5 * IQR

If min  < Lower fence  →  potential lower outliers
If max  > Upper fence  →  potential upper outliers
```

Also check:
- Is `min` = 0 or negative when it shouldn't be? (e.g., age = 0, price = -5)
- Is `max` unrealistically large? (e.g., age = 999, salary = 99999999)
- Is `std` very large compared to the `mean`? → High spread, likely outliers

---

### 4. Understand the Spread → `std` and percentiles

```
Small std (relative to mean)  →  values are clustered tightly around the mean
Large std (relative to mean)  →  values are widely spread
```

The **interquartile range (IQR)** tells you where the middle 50% of data lives:
```
IQR = 75% - 25%
Narrow IQR  →  most values are similar
Wide IQR    →  high variability in the middle of the data
```

---

### 5. Understand the Range → `min` to `max`

```
Range = max - min
```
- A very large range with a small IQR = **extreme outliers** at the tails
- A large range with a large IQR = **genuinely high variability**

### Visual Proof

**How to read the charts below:**
- **Range** = distance from `min` to `max` (the full spread)
- **IQR** = distance from `Q1` to `Q3` (the middle 50% of data, the green zone)
- **Red zones** in the histogram = outlier territory (beyond 1.5×IQR from Q1/Q3)

![Range vs IQR — Boxplot and Histogram](range_vs_iqr.png)

| | Case 1 (Top row) | Case 2 (Bottom row) |
|---|---|---|
| **Range** | Very large (~99) | Large (~99) |
| **IQR** | Very small (~6) | Large (~50) |
| **Box in boxplot** | Tiny — squeezed in middle | Wide — spans most of the range |
| **Histogram shape** | Tall spike + scattered dots at tails | Flat, even spread |
| **Conclusion** | ⚠️ Extreme outliers exist | ✅ Genuinely high variability |

> **Key insight:** Both cases have a similar Range (~99), but the IQR tells a completely different story.
> The boxplot's box size instantly reveals which case you're dealing with.

![Range vs IQR — Side-by-Side Comparison](range_iqr_comparison.png)

---

### 6. Identify Column Type from the Numbers

| What you see | What it likely means |
|---|---|
| min=0, max=1, mean between 0–1 | **Binary column** (e.g., yes/no, survived/not) |
| Only a few unique values (e.g., 1, 2, 3) | **Categorical encoded as number** (e.g., class, rating) |
| min=1, max=N, mean=N/2 | Likely just a **row ID** — drop it |
| Very large range, right-skewed | Likely a **monetary/count column** (price, salary, views) |
| Values around 0–100 | Could be **age, percentage, score** |

---

## Quick Checklist — What to Do After Reading `df.describe()`

```
[ ] count < total rows?           →  Find and handle missing values
[ ] mean >> median?               →  Apply log transformation
[ ] mean << median?               →  Apply sqrt or reflect transformation
[ ] min or max look wrong?        →  Investigate and handle outliers
[ ] std is very large vs mean?    →  Check for outliers, consider scaling
[ ] min = max (or std = 0)?       →  Column has only one value → drop it (useless)
[ ] mean ≈ 0 or 1 with tiny std?  →  Likely a binary or near-constant column
[ ] min = 1, max = N, mean = N/2? →  This is an ID column → drop it
```

---

## How to Cross-Check Skewness in Code

```python
# Check skewness of all numerical columns
df.skew()

# Skewness interpretation:
# -0.5 to +0.5  →  Roughly symmetric
# +0.5 to +1.0  →  Moderately right-skewed
# > +1.0        →  Highly right-skewed (apply log transform)
# < -1.0        →  Highly left-skewed
```

---

## How to Find Outlier Boundaries in Code

```python
Q1 = df['column'].quantile(0.25)
Q3 = df['column'].quantile(0.75)
IQR = Q3 - Q1

lower = Q1 - 1.5 * IQR
upper = Q3 + 1.5 * IQR

outliers = df[(df['column'] < lower) | (df['column'] > upper)]
print(f"Outliers found: {len(outliers)}")
```

---

*This is a general guide — applies to any dataset.*
