# SNES_MiSTer Timing Issues Analysis

Analysis of the two timing problems reported by a core maintainer: **-4ns slack** and a **new async clock**.

---

## 1. New Async Clock — Subcarrier Generation (Framework Update)

**Introduced by:** Framework update `a085205` (Dec 6, 2025) — "Update framework (#449)"

The framework update added a subcarrier generator for external composite/S-Video encoders in `sys/sys_top.v:1446-1451`:

```verilog
reg [39:0] sub_accum;
always @(posedge clk_vid) sub_accum <= sub_accum + PhaseInc;

reg subcarrier_out;
always @(posedge clk_vid) subcarrier_out <= ~(subcarrier & csync_en & ...) | sub_accum[39];
```

`subcarrier_out` is then ANDed into `VGA_VS` (line 1506), directly affecting analog video output.

### The CDC violation

`PhaseInc` is a **40-bit register written in the `clk_sys` domain** (`sys_top.v:513-515`, inside an `always @(posedge clk_sys)` block driven by the HPS SPI interface):

```verilog
1: PhaseInc[15:0]  <= io_din;
2: PhaseInc[31:16] <= io_din;
3: PhaseInc[39:32] <= io_din[7:0];
```

It is **read in the `clk_vid` domain** (the 40-bit accumulator addition above). These two clocks are in **different exclusive clock groups** per `sys_top.sdc:13-22`:

```tcl
set_clock_groups -exclusive \
   -group [get_clocks { *|pll|pll_inst|altera_pll_i|*[*].*|divclk}] \  # core PLL → clk_vid
   ...
   -group [get_clocks { *|h2f_user0_clk}] \                            # HPS → clk_sys
```

The SDC file does declare `set_false_path -from {PhaseInc}` (line 63), which silences the timing tool. However, the hardware CDC is **unprotected** — there is no synchronizer, gray-code handshake, or hold register on the 40-bit bus. If `PhaseInc` is sampled mid-update across the three successive 16-bit writes, the accumulator will produce a glitched output for one or more `clk_vid` cycles.

In practice this may be benign (PhaseInc is written once at startup and rarely changes), but it is technically an unsafe CDC crossing on a wide bus, and Quartus may flag it as a new unconstrained async path contributing to timing degradation.

### Where it lives

This is in `sys/` (shared MiSTer framework) — the fix needs to happen upstream, not in the core.

---

## 2. Big Slack (-4ns) — SDRAM Path Regression

The -4ns slack is on the `clk_sys` → `clk_mem` crossing path between the core logic and the SDRAM controller. Multiple changes have combined to degrade this path:

### Contributing changes

#### a) Save states added significant logic to the datapath

**Commit:** `7e9ec63` (Jul 7, 2025) — "Add Save states"

This was a +5057/-267 line change touching 28 files. It added:
- `ddram.sv` — new DDR3 interface module (189 lines)
- Save state arbitration muxing in `main.v` (+236 lines)
- State capture/restore registers in DSP, GSU, SA1, PPU, SMP, SPC700
- `savestates.sv`, `savestates_regs.sv`, `savestates_map.sv`, `savestates_sa1.sv`

All save state logic runs in `clk_sys` and shares the `emu|main|*` register namespace that feeds into the SDRAM controller — adding combinational depth and fanout to the constrained path.

#### b) SDC constraint was tightened

In the same timeframe, `SNES.sdc:15` was changed:

```diff
-set_max_delay 23 -from [get_registers { emu|hps_io|* \
+set_max_delay 20 -from [get_registers { emu|hps_io|* \
                                         emu|main|* \
                                         emu|bsram|* \
                                         emu|rom_mask[*] \
                                         emu|rom_type[*] }] \
                  -to   [get_registers { emu|sdram|* }]
```

The max-delay budget was reduced from **23ns to 20ns** while more logic was being added to the same path.

#### c) ROM address mirroring added combinational logic

**Commit:** `3839622` (Mar 2, 2026) — "Add rom address mirroring when rom size is not a power of 2"

This added a conditional mux directly in the SDRAM address path (`SNES.sv:908-912`):

```verilog
wire        rom_in_range  = !(|(ROM_ADDR & ~rom_mask));
wire        rom_in_mirror = rom_mirror_en & rom_in_range & |(ROM_ADDR & rom_mirror_split);
wire [23:0] rom_addr_out  = rom_in_mirror
    ? (rom_mirror_split | (ROM_ADDR & rom_mirror_rmask))
    : ROM_ADDR;
```

This is purely combinational logic (no register stage) sitting between `emu|main|*` outputs and `emu|sdram|addr` inputs — adding to the critical path delay.

### The math

With the original 23ns budget, the path had ~4ns of positive slack. The combination of:
- Constraint tightened by 3ns (23 → 20)
- Additional combinational depth from ROM mirroring (~1-2ns)
- Additional fanout/routing from save state muxing (~1-2ns)

Results in roughly **-4ns slack** (was +4ns, lost ~8ns total).

---

## Potential Remediation

### For the slack issue (actionable in this repo)

1. **Revert the max-delay to 23ns** — The original constraint worked. The 4:1 frequency ratio between `clk_mem` (85.9 MHz) and `clk_sys` (21.5 MHz) provides inherent multi-cycle margin. The constraint could safely be 23ns or even relaxed further, since SDRAM transactions are initiated on `clk_sys` edges and the SDRAM controller has multiple `clk_mem` cycles to sample them.

2. **Register the ROM mirror address** — Add a pipeline register for `rom_addr_out` in `clk_sys` to break the combinational path:
   ```verilog
   reg [23:0] rom_addr_out_r;
   always @(posedge clk_sys) rom_addr_out_r <= rom_addr_out;
   ```
   This adds one `clk_sys` cycle of latency (46ns) but eliminates the combinational depth. The SDRAM read result already has latency tolerance built in.

3. **Both** — Register the address AND keep the constraint at 20ns for additional margin.

### For the async clock issue (requires upstream fix in `sys/`)

The `PhaseInc` CDC in `sys/sys_top.v` should use either:
- A register stage with a valid/acknowledge handshake
- A shadow register that loads atomically after all three partial writes complete
- Gray-code encoding (impractical for a 40-bit phase increment)

Since `PhaseInc` is only written during configuration (not at runtime), the simplest fix is a shadow register pattern: accumulate the three 16-bit writes into a staging register in `clk_sys`, then pulse a synchronized load signal to latch the full 40-bit value in `clk_vid`.

---

## Files Involved

| File | Role in Issue |
|------|--------------|
| `sys/sys_top.v:1440-1451` | Subcarrier accumulator — async clock source |
| `sys/sys_top.v:505-518` | PhaseInc writes in clk_sys domain |
| `sys/sys_top.sdc:13-22` | Clock group exclusions |
| `SNES.sdc:15-32` | Max-delay constraint (tightened 23→20) |
| `SNES.sv:908-912` | ROM mirror combinational mux in SDRAM address path |
| `rtl/main.v` | Save state arbitration adding to emu|main|* fanout |
| `rtl/sdram.sv` | SDRAM controller (destination of constrained path) |
| `rtl/savestates*.sv` | Save state logic in clk_sys domain |

## Relevant Commits

| Hash | Date | Description |
|------|------|-------------|
| `a085205` | 2025-12-06 | Update framework — introduces subcarrier/async clock |
| `7e9ec63` | 2025-07-07 | Add Save states — major logic addition to clk_sys domain |
| `85b76d3` | 2025-09-09 | ROM address path optimization + SDC tightening (23→20ns) |
| `3839622` | 2026-03-02 | ROM address mirroring — combinational logic in SDRAM path |
