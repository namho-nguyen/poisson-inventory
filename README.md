# Poisson Inventory Models with Many Items: An Empirical Bayes Approach

Companion data and item-level results for the paper by **Edward Anderson,
Nam Ho-Nguyen and Peter Radchenko**.

The paper studies which items to stock and how much to order when each item
has Poisson demand and limited historical observations. Empirical Bayes uses
information across items to estimate demand distributions and construct
inventory decisions. The grouping study examines when it is beneficial to
estimate demand separately for two populations of items.

This repository contains the simulation datasets, anonymous book-sales data
and two-group datasets, together with the saved inventory decisions and
profit evaluations. The files let readers connect individual decisions to
the paper's aggregate comparisons and examine how those comparisons vary
with sample size, demand level, fixed stocking cost and grouping decisions.

## Connection to the paper

Section and table references below follow the accompanying manuscript.

| Experiment | Data and results | Context in the paper |
| --- | --- | --- |
| Simulation | [data/simulation](data/simulation), [results/simulation](results/simulation) | Sections 6.1–6.4 compare five estimation methods with a known-prior oracle. Figures 1–2 show performance as the number of items increases; Tables 1–2 examine demand scale and fixed cost. Table S.3 examines the stocking decisions behind the profit differences. |
| Book sales | [data/book](data/book), [results/book](results/book) | Section 6.5 and Table 3 evaluate inventory decisions using 2022 sales to predict 2023 demand. The supplementary book study compares observation distributions and breaks down stocking decisions and realised profits by 2022 sales group (Table S.6). |
| Grouping | [data/group](data/group), [results/group](results/group) | Section 7 and Table 4 compare pooling both groups with fitting them separately, selected using a likelihood-ratio test. Table S.7 compares the best splitting cutoffs across group sizes. |

## Reading the saved results

Every saved quantity and profit is recorded **before the fixed stocking cost
`b` is applied**. This allows the same fitted decisions to be evaluated for
different fixed costs.

The common item inputs are:

| Field | Meaning |
| --- | --- |
| `lambda` | Simulated latent Poisson demand rate. |
| `r` | Revenue per unit sold. |
| `c` | Variable cost per unit ordered. |
| `obs` | Historical demand observation available to the estimation method. |

Simulation input datasets use the equivalent names `lamb`, `rev` and `cost`;
their result files use `lambda`, `r` and `c`. Group inputs and results use
`lambda`, `r` and `c` throughout.

For each simulation or grouping method, the result columns use these suffixes:

| Column suffix | Meaning |
| --- | --- |
| `_quantity_pre_b` | Inventory quantity selected using the method's predictive distribution. |
| `_estimated_profit_pre_b` | Expected profit of that quantity according to the fitted predictive distribution; this determines whether to stock. |
| `_true_conditional_profit_pre_b` | Expected profit of the same quantity under the true predictive distribution conditional on the item's observation and, in the grouping study, its group. |

The `optimal_conditional` benchmark uses the known prior distribution to
choose the optimal conditional inventory quantity. Its estimated and true
conditional profits coincide. It provides the reference profit used in the
paper's relative performance comparisons.

### Applying the fixed stocking cost

For an item and method, let `q0` be the saved quantity, `e0` its estimated
pre-`b` profit, and `v0` its evaluation profit. For simulation and grouping,
`v0` is true conditional expected profit; for books it is realised 2023 profit.

```text
stock = (e0 > b)
quantity_at_b = q0 if stock else 0
estimated_profit_at_b = e0 - b if stock else 0
evaluation_profit_at_b = v0 - b if stock else 0
```

Use this rule for any nonnegative `b`, choosing not to stock at equality.
The fixed charge affects whether an item is stocked, while the selected
positive quantity stays the same. For the oracle, use its true conditional
profit as both `e0` and `v0`.

## Simulation study

The study compares methods across seven demand scenarios:

- Weibull priors with shape `1.8` and scales `2`, `3`, `4`, `5`.
- Two-point priors with lower rate `1`, upper rate `3`, `4` or `5`, and
  probability `0.5` on each rate.

Each scenario has 50 runs, numbered `0` through `49`, with 50,000 items in
each input dataset. The nine fitted sample sizes are `50`, `100`, `250`,
`500`, `1000`, `5000`, `10000`, `25000` and `50000`. Smaller samples are
nested prefixes of the same run's input dataset.

There are **3,150 result CSVs**, one per scenario, run and sample size. Each
has exactly `n` rows. The zero-based `item_index` identifies the corresponding
row of the run's input dataset, and the result's `n` is the fitted sample size.

Examples:

```text
results/simulation/weibull/weibull_scale3_run0_n1000.csv
results/simulation/twopoint/twopoint_low1_high3_p0.5_run0_n1000.csv
```

Weibull results include `weibull_shape` and `weibull_scale`; two-point results
include `lambda_low`, `lambda_high` and `mixture_probability`, the probability
on the lower rate. Weibull CSVs have 30 columns and two-point CSVs have 31.

The six result prefixes are:

| Prefix | Role in the comparison |
| --- | --- |
| `optimal_conditional` | Known-prior oracle benchmark. |
| `single_item` | Uses each item's observation as its Poisson-rate estimate. |
| `shrinkage` | Adjusts item-rate estimates using information across items. |
| `fmodel_plugin` | Uses marginal-distribution estimation with a plug-in rate estimate. |
| `fmodel_full` | Uses marginal-distribution estimation to construct the full predictive distribution. |
| `gmodel` | Estimates the mixing distribution of Poisson rates and uses its full predictive distribution. |

Each prefix has the three result suffixes defined above. The
`fmodel_plugin_cdf_valid` and `fmodel_full_cdf_valid` flags describe the fitted
CDFs. The paper's comparisons use all saved method results, including flagged
CDFs and negative evaluated profits.

### Mean profits and stocking comparisons

For Tables 1–2, apply each fixed cost `b = 0, 0.1, ..., 0.6` and first average
profits across the items in each run. At `n=100`, Table 1 averages equally
over the 50 runs and seven fixed costs for each Weibull scale; Table 2 averages
equally over the 50 runs and four Weibull scales for each fixed cost.

Table S.3 uses Weibull scale `3` and `n=10000`, pooling 500,000 item records
from the 50 runs. For each `b`, classify items by whether the oracle stocks
them, then calculate each method's stocking percentage within those oracle
subsets. The displayed method profit and quantity averages are conditional
on the method stocking the item within the relevant subset.

### Figures 1–2 and bootstrap confidence bands

Use `b=0.2`. For each scenario, sample size and method, compute mean profits
within each run. In numeric run order `0,...,49`, form

```text
x_j = 100 * (oracle_mean_profit_j - method_mean_profit_j)
            / oracle_mean_profit_j
```

Plot the mean of these 50 relative gaps. The confidence bands use 10,000
studentized (bootstrap-t) resamples of the runs, shared across methods and
sample sizes within each scenario.

To recover the same numerical bands, use NumPy
`SeedSequence(20260806).spawn(7)` in this scenario order: Weibull scales
`2,3,4,5`, then two-point upper rates `3,4,5`. For each scenario create
`default_rng(child_seed)` and draw multinomial counts with 50 trials,
50 equal probabilities and `size=10000`. Draw this count matrix once per
scenario and use it for every sample-size/method combination.

For each vector `x` of 50 run gaps, let `m` be its mean, `s` its sample standard
deviation (`ddof=1`) and `SE=s/sqrt(50)`. For each count vector `w`, compute:

```text
mb = sum(w*x) / 50
sb = sqrt(max(sum(w*x*x) - 50*mb*mb, 0) / 49)
SEb = sb / sqrt(50)
tb = (mb - m) / SEb
CI = [m - q97.5*SE, m - q2.5*SE]
```

Here `q2.5` and `q97.5` are the corresponding quantiles of `tb`, using linear
interpolation. The standard-deviation threshold is
`100 * machine_epsilon * max(1, abs(m), s, max(abs(x)))`. Treat resamples at
or below this threshold as NaN and omit them from the quantiles; when the
observed `s` is at or below the threshold, use `tb=0`. At least 99% of resamples
are valid in the reported calculations.

The figures use a vertical range of `0` to `100`. The single-item curve is
omitted for Weibull scale `2` and two-point upper rate `3`.

## Book-sales study

The book study examines inventory decisions for **178 titles**, using 2022
sales as the historical observation and 2023 sales to evaluate the resulting
decisions. [book_sales_prices.csv](data/book/book_sales_prices.csv) contains
`item_id`, `sales_2022`, `sales_2023` and `price`. IDs are `0` through `177`.

[book_decisions_pre_b.csv](results/book/book_decisions_pre_b.csv) has 534 rows:
one per title and method (`gmodel`, `shrinkage`, `single_item`). The same
`item_id` joins the input data to each method's decision. The result columns
are `item_id`, `method`, `sales_2022`, `sales_2023`, `price`, `r`, `c`,
`quantity_pre_b`, `estimated_profit_pre_b` and `realized_profit_pre_b`.

Revenue is `r = 0.2 * price` and variable cost is `c = 0.08 * price`. The
estimated profit uses 2022 information, while the realised evaluation is

```text
realized_profit_pre_b = r * min(sales_2023, quantity_pre_b)
                       - c * quantity_pre_b
```

Apply the common fixed-cost rule using `estimated_profit_pre_b` to determine
stocking and `realized_profit_pre_b` to evaluate the decision. Table 3 reports
total realised profit across the 178 titles. Table S.6 groups titles by 2022
sales (`0`, `1–2`, `at least 3`) and reports total titles stocked, units ordered
and realised profits at `b=0,4,5`.

The supplementary histogram compares all 178 book observations with the first
178 observations from each of the 350 simulation datasets. Use separate bins
for counts `0` through `14` and a final bin for `15` or more.

## Grouping study

The grouping study examines the trade-off between pooling more observations
and allowing different demand distributions for two groups. The 540 input
datasets combine larger-group Weibull scales `2`, `4`, `8` with smaller-group
scale ratios `0.6`, `0.8`, `1`, `1.2`, `1.4`, `1.6`, across 30 runs per scale
pair. Each input dataset contains 10,000 larger-group items and 5,000
smaller-group items. Fitted samples use `1000`, `5000` or `10000` larger-group
items and `10%`, `20%` or `50%` as many smaller-group items. The nine size
combinations produce
**4,860 result CSVs**.

`gmodel_combined` estimates one distribution from both groups;
`gmodel_split` estimates a separate distribution for each group. Their saved
quantities and profits allow the pooling decision to be evaluated for
different likelihood-ratio cutoffs and fixed stocking costs.

### Columns and item identity

The 19 result columns are ordered in these blocks:

1. Prior and identifiers: `weibull_shape`, `weibull_scale`, `run`, `group`.
2. Item inputs: `lambda`, `r`, `c`, `obs`.
3. Oracle: `optimal_conditional_quantity_pre_b`,
   `optimal_conditional_true_conditional_profit_pre_b`.
4. Pooled fit: `gmodel_combined_quantity_pre_b`,
   `gmodel_combined_estimated_profit_pre_b`,
   `gmodel_combined_true_conditional_profit_pre_b`.
5. Separate fits: `gmodel_split_quantity_pre_b`,
   `gmodel_split_estimated_profit_pre_b`,
   `gmodel_split_true_conditional_profit_pre_b`.
6. Likelihood-ratio test: `LR_stat`, `LR_DF`, `LR_pval`.

For either method, use its quantity, estimated profit and true conditional
profit in the common fixed-cost rule. For the oracle, the saved true
conditional profit serves as both the estimated and evaluation profit.

The filenames encode the two scales, run and group sizes. In the files,
`group=0` is the larger group and `group=1` the smaller group; the paper uses
size notation `n_1` for the larger group and `n_0` for the smaller group.
Each fitted sample uses the required prefix of its group's input dataset.
Row order is preserved: identify an item by its group and its zero-based
position within that group in the corresponding input dataset.

### Cutoff comparisons

For cutoff `a`, select the split fit when `LR_pval < a`, and otherwise select
the pooled fit. Apply the fixed stocking cost to the selected method. The
reported comparisons use cutoffs
`0, 0.000005, 0.00005, 0.0005, 0.005, 0.05, 0.5, 1` and fixed costs
`b = 0, 0.1, ..., 0.6`.

For each fitted configuration and fixed cost, average profits across the
smaller group's items. Then average equally across the 18 scale pairs,
30 runs and seven fixed costs. Table 4 uses the smaller/larger size pair
`200/1000`; Table S.7 selects the best cutoff for each of the nine size pairs.

Table 4's conditional losses use only configuration/fixed-cost rows whose
split/pool choice differs from the best cutoff. Their standard errors equal
the population standard deviation of those losses (`ddof=0`) divided by the
square root of the number of such rows.

## True conditional-distribution tables

The files under `data/simulation/conditional_distributions` and
`data/group/conditional_distributions` contain the true conditional CDFs used
to evaluate saved decisions and form the oracle benchmark. Entry `(k,s)` is
`P(future demand <= k | observed demand = s)`: the CSV header gives the
observation `s`, and data-row positions `k` start at zero.

The grouping cache has 14 files, one per scale. Integer scales use filenames
without a decimal suffix, such as `weibull_scale2_shape1.8_conditional.csv`,
and similarly for scales `4` and `8`. Other scales use their usual decimal
representation. The retained tables cover every saved group observation and
inventory quantity.
