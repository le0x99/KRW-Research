sub = TH // 2

check = TB * df["avg_ask_benchmark_interval"]

for offset in range(0, TB, sub):
    width = min(sub, TB - offset)

    if width == sub:
        # full subwindow chunk
        chunk = group["avg_ask_0"].shift(-offset)
    else:
        # final partial tail chunk
        chunk = (
            group["ask"]
            .rolling(width, 

sub = TH // 2

check = TB * df["avg_ask_benchmark_interval"]

for offset in range(0, TB, sub):
    width = min(sub, TB - offset)

    if width == sub:
        # full subwindow chunk
        chunk = group["avg_ask_0"].shift(-offset)
    else:
        # final partial tail chunk
        chunk = (
            group["ask"]
            .rolling(width, 



















import numpy as np
import pandas as pd


def avg_prices_at_offsets(
    df: pd.DataFrame,
    offsets,
    price_cols,
    *,
    date_col: str = "Date",
    split_col: str = "Split",
    time_col: str = "Time",
    suffix: str = "_avg",
) -> pd.DataFrame:
    if isinstance(price_cols, str):
        price_cols = [price_cols]
    else:
        price_cols = list(price_cols)

    offsets = tuple(offsets)

    if len(offsets) == 0:
        raise ValueError("offsets must contain at least one value")

    if len(price_cols) == 0:
        raise ValueError("price_cols must contain at least one column")

    key_cols = [date_col, split_col, time_col]
    required_cols = key_cols + price_cols

    missing_cols = [c for c in required_cols if c not in df.columns]
    if missing_cols:
        raise ValueError(f"Missing required columns: {missing_cols}")

    if df.empty:
        return pd.DataFrame(
            index=df.index,
            columns=[f"{c}{suffix}" for c in price_cols],
            dtype="float64",
        )

    if df[key_cols].isna().any().any():
        bad_cols = df[key_cols].columns[df[key_cols].isna().any()].tolist()
        raise ValueError(f"Key columns must not contain missing values: {bad_cols}")

    if df.duplicated(key_cols).any():
        dupes = df.loc[df.duplicated(key_cols, keep=False), key_cols]
        raise ValueError(
            "Duplicate (Date, Split, Time) keys exist; lookup is ambiguous.\n"
            f"{dupes.head(20)}"
        )

    if not pd.api.types.is_integer_dtype(df[time_col]):
        raise TypeError(f"{time_col} must have an integer dtype")

    for col in price_cols:
        if not pd.api.types.is_numeric_dtype(df[col]):
            raise TypeError(f"{col} must be numeric")

    offsets_arr = np.asarray(offsets, dtype=np.int64)
    times = df[time_col].to_numpy(dtype=np.int64)

    int64_min = np.iinfo(np.int64).min
    int64_max = np.iinfo(np.int64).max

    min_target = int(times.min()) + int(offsets_arr.min())
    max_target = int(times.max()) + int(offsets_arr.max())

    if min_target < int64_min or max_target > int64_max:
        raise OverflowError("Time + offset would overflow int64")

    lookup = df.set_index(key_cols)[price_cols]

    looked_up = []

    for offset in offsets_arr:
        target_index = pd.MultiIndex.from_arrays(
            [
                df[date_col].to_numpy(),
                df[split_col].to_numpy(),
                times + offset,
            ],
            names=key_cols,
        )

        values = lookup.reindex(target_index).to_numpy(
            dtype=np.float64,
            na_value=np.nan,
        )

        looked_up.append(values)

    # Shape: n_offsets x n_rows x n_price_cols
    arr = np.stack(looked_up, axis=0)

    # If any required timestamp is missing, reindex gives NaN.
    # If a required price exists but is NaN, result is also NaN for that column.
    invalid = np.isnan(arr).any(axis=0)

    result = arr.mean(axis=0)
    result[invalid] = np.nan

    return pd.DataFrame(
        result,
        index=df.index,
        columns=[f"{c}{suffix}" for c in price_cols],
    )



min_periods=width)
            .mean()
            .shift(-TB)
        )

    check -= width * chunk

check.abs().describe()


Yes — purely algebraic, using only already-computed columns.

Let:

sub = TH // 2

1. Local sanity check

This must always hold:

TH * avg_*_interval
=
sub * avg_*_0 + sub * avg_*_1

Code:

check_interval = (
    TH * df["avg_ask_interval"]
    - sub * df["avg_ask_0"]
    - sub * df["avg_ask_1"]
)
check_interval.abs().describe()

Same for bid.

⸻

2. Inter-row sanity check

This must also always hold:

avg_*_1(t) = avg_*_0(t + sub)

Code:

check_shift = (
    df["avg_ask_1"]
    - group["avg_ask_0"].shift(-sub)
)
check_shift.abs().describe()

Same for bid.

⸻

3. Benchmark decomposition check

The benchmark should equal the weighted sum of all atomic sub windows.

check_benchmark = TB * df["avg_ask_benchmark_interval"]
for offset in range(0, TB, sub):
    width = min(sub, TB - offset)
    check_benchmark -= width * group["avg_ask_0"].shift(-offset)
check_benchmark.abs().describe()

This is fully algebraic only if TB % sub == 0.

Example:

TB = 15
TH = 2
sub = 1

Then it uses:

avg_ask_0.shift(0)
avg_ask_0.shift(-1)
avg_ask_0.shift(-2)
...
avg_ask_0.shift(-14)

and should be zero.

⸻

If TB % sub != 0, then the last chunk is shorter than sub, and avg_*_0.shift(...) is too long for the final piece. Then you need one additional precomputed column:

avg_*_final_tail

meaning:

[TB - remainder, TB]

Then:

r = TB % sub
check_benchmark = TB * df["avg_ask_benchmark_interval"]
for offset in range(0, TB - r, sub):
    check_benchmark -= sub * group["avg_ask_0"].shift(-offset)
if r > 0:
    check_benchmark -= r * df["avg_ask_final_tail"]
check_benchmark.abs().describe()

So the universal algebraic checks are:

1. TH * interval = sub * avg_0 + sub * avg_1
2. avg_1(t) = avg_0(t + sub)
3. TB * benchmark = weighted sum of shifted avg_0 chunks, plus final_tail if needed

No raw ask/bid needed.
