```py
import math
import matplotlib.pyplot as plt

SYS_CLK_HZ = 60_000_000
SPI_CLK_HZ = 20_000_000
CLKS_PER_SCK = SYS_CLK_HZ // SPI_CLK_HZ  # 3 sys clocks per SCK
BITS_PER_SCK = 2                         # Dual SPI

def sim_fifo(fifo_bits, t_bit_clks, turnaround_clks=180, burst_words=4, word_bits=16, margin_bits=16, mode="dynamic_ae", sim_clks=200_000):
    burst_bits = burst_words * word_bits
    if mode == "empty":
        ae_thresh = 0
    else:
        ae_thresh = min(
            math.ceil(turnaround_clks / t_bit_clks) + margin_bits,
            max(0, fifo_bits - burst_bits),
        )

    occ = fifo_bits // 2
    bit_timer = starved = ovf_bits = occ_sum = bursts = 0
    state, spi_timer, rem_bits, sck_cnt = "IDLE", 0, 0, 0

    for _ in range(sim_clks):
        if occ > 0:
            bit_timer += 1
            if bit_timer >= t_bit_clks:
                bit_timer = 0
                occ -= 1
        else:
            starved += 1

        if state == "IDLE" and occ <= ae_thresh:
            state, spi_timer = "TURNAROUND", turnaround_clks
        elif state == "TURNAROUND":
            spi_timer -= 1
            if spi_timer <= 0:
                state, rem_bits, sck_cnt = "SHIFTING", burst_bits, 0
        elif state == "SHIFTING":
            sck_cnt += 1
            if sck_cnt >= CLKS_PER_SCK:
                sck_cnt = 0
                rem_bits -= BITS_PER_SCK
                if rem_bits % word_bits == 0:
                    if occ + word_bits <= fifo_bits:
                        occ += word_bits
                    else:
                        ovf_bits += word_bits
                if rem_bits <= 0:
                    state = "IDLE"
                    bursts += 1

        occ_sum += occ

    return {
        "ae": ae_thresh,
        "avg_fill": 100.0 * (occ_sum / sim_clks) / fifo_bits,
        "starve": 100.0 * starved / sim_clks,
        "ovf_bits": ovf_bits,
        "bursts": bursts,
    }

scenarios = [("10BASE-T (T=6)", 6), ("2 Mbps (T=30)", 30), ("I2C 400k (T=150)", 150), ("UART 115k (T=521)", 521)]

for fifo_bits in [128, 256]:
    b_words = 2 if fifo_bits == 128 else 4
    print(f"\n--- FIFO={fifo_bits}b ({fifo_bits//16} words), burst={b_words} words ---")
    empty_s, dyn_s = [], []
    for label, t_bit in scenarios:
        r_e = sim_fifo(fifo_bits, t_bit, burst_words=b_words, mode="empty")
        r_d = sim_fifo(fifo_bits, t_bit, burst_words=b_words, mode="dynamic_ae")
        empty_s.append(r_e["starve"])
        dyn_s.append(r_d["starve"])
        print(f"{label:18s} | EMPTY starve={r_e['starve']:5.2f}% | DYN(AE={r_d['ae']:3d}b) fill={r_d['avg_fill']:5.1f}% starve={r_d['starve']:5.2f}% ovf={r_d['ovf_bits']}b bursts={r_d['bursts']}")

    plt.figure(figsize=(7, 3.5))
    x = range(len(scenarios))
    plt.bar([i - 0.18 for i in x], empty_s, width=0.36, label="Wait for Empty")
    plt.bar([i + 0.18 for i in x], dyn_s, width=0.36, label="Dynamic Almost-Empty")
    plt.xticks(x, [s[0] for s in scenarios])
    plt.ylabel("Starvation (%)")
    plt.title(f"TX FIFO Starvation ({fifo_bits}-bit FIFO, 60 MHz Sys / 20 MHz Dual SPI)")
    plt.legend()
    plt.tight_layout()
    plt.show()
```

Result

```
--- FIFO=128b (8 words), burst=2 words ---
10BASE-T (T=6)     | EMPTY starve=51.42% | DYN(AE= 46b) fill=  9.5% starve=16.10% ovf=0b bursts=872
2 Mbps (T=30)      | EMPTY starve=17.44% | DYN(AE= 22b) fill= 24.5% starve= 0.00% ovf=0b bursts=207
I2C 400k (T=150)   | EMPTY starve= 3.98% | DYN(AE= 18b) fill= 26.1% starve= 0.00% ovf=0b bursts=41
UART 115k (T=521)  | EMPTY starve= 1.02% | DYN(AE= 17b) fill= 26.9% starve= 0.00% ovf=0b bursts=11
```

```
--- FIFO=256b (16 words), burst=4 words ---
10BASE-T (T=6)     | EMPTY starve=34.58% | DYN(AE= 46b) fill= 15.1% starve= 0.00% ovf=0b bursts=519
2 Mbps (T=30)      | EMPTY starve= 9.49% | DYN(AE= 22b) fill= 18.4% starve= 0.00% ovf=0b bursts=103
I2C 400k (T=150)   | EMPTY starve= 1.94% | DYN(AE= 18b) fill= 19.9% starve= 0.00% ovf=0b bursts=20
UART 115k (T=521)  | EMPTY starve= 0.41% | DYN(AE= 17b) fill= 22.2% starve= 0.00% ovf=0b bursts=5
```


Thanks to gemini colab to run this and got the result and bar chart. 
This is a simulation proof that dynamic empty will work at higher speed it will not starve our FIFO

For now just did a sim with size as 128 and 256 bits. Need to explore tradeoff memory area vs dynamic size calculation area and arrive at a sweet spot.

Those are the two main extremes—**logic/timing smarts (dynamic `almost_empty`)** versus **storage cushion (bigger FIFO)**—though you also have a **middle ground** and can even combine a small threshold with a moderate FIFO.

### 1. Dynamic `almost_empty` (Smarter Trigger, Smaller FIFO)

* **What it does:** Keeps the FIFO smaller and adjusts when `almost_empty` fires based on `T_bit` (and known SPI timing), giving the RP2350 more lead time on fast protocols (`10BASE-T`) and less on slow ones (`UART`/`I2C`).
* **Trade-off:** Saves flip-flops and read-mux area, while adding a small comparator/lookup block and needing accurate SPI turnaround assumptions.

### 2. Big FIFO (More Cushion, Simpler Trigger)

* **What it does:** Gives `TX_FIFO` enough capacity to absorb burst latency and slow firmware/SPI jitter without needing smart `T_bit`-aware thresholds.
* **Trade-off:** Saves comparator logic, but costs **standard-cell flip-flops and mux area**—which can get tight on Tiny Tapeout alongside your dual sequencers and instruction memories.

### 3. The Middle Ground (Often Best on Tiny Tapeout)

* You don't have to force a binary choice between "tiny FIFO + complex math" and "huge FIFO."
* A **moderate FIFO** (such as `128` or `256` bits) paired with a **simple 2- or 3-tier `almost_empty` watermark** (e.g., high watermark for fast `T_bit`, low watermark for slow `T_bit`) usually balances flip-flop count, low logic overhead, and zero/low starvation without bloating tile usage.
