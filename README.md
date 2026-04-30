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
