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
            .rolling(width, min_periods=width)
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
