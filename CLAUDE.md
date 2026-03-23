# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

SNES_MiSTer is a cycle-accurate Super Nintendo / Super Famicom FPGA core for the MiSTer platform, written primarily in VHDL and SystemVerilog. It targets the Intel Cyclone V FPGA via Quartus v17.0.2.

## Build

Open `SNES.qpf` in Quartus v17.0.2 and run full compilation. Output is an `.rbf` file placed on the MiSTer SD card. There is no simulation or lint target — all verification is done on hardware.

When adding new RTL files, add them to `files.qip` (never through the Quartus IDE, which corrupts `.qsf`). Sub-projects (65C816, SPC700, chip coprocessors) have their own `.qip` files referenced from the top-level `files.qip`.

## Architecture

### Module Hierarchy (three layers)

1. **`SNES.sv`** (MiSTer integration) — PLL configuration, HPS I/O, ROM loading, save state slot management, controller input, video/audio output routing. This is the MiSTer framework glue.

2. **`rtl/main.v`** (coprocessor arbitration) — Instantiates the SNES core and all coprocessor map controllers. Multiplexes ROM/BSRAM buses based on `MAP_ACTIVE` (7-bit vector selecting the active coprocessor). Handles ROM word mode (8-bit vs 16-bit access).

3. **`rtl/SNES.vhd`** (core SNES logic) — Instantiates SCPU, SPPU, SMP, DSPN and connects internal buses. Manages WRAM, VRAM, ARAM interfaces, DMA/HDMA, and the cheat engine.

### Major Subsystems

| Subsystem | Key Files | Language |
|-----------|-----------|----------|
| 65C816 CPU + DMA | `rtl/CPU.vhd`, `rtl/65C816/P65C816.vhd` | VHDL |
| PPU (graphics) | `rtl/PPU.vhd`, `rtl/PPU_PKG.vhd` | VHDL |
| SMP (sound CPU) | `rtl/SMP.vhd`, `rtl/SPC700/` | VHDL |
| DSP (audio) | `rtl/DSP.vhd`, `rtl/DSP_PKG.vhd` | VHDL |
| SDRAM controller | `rtl/sdram.sv` | SystemVerilog |
| Save states | `rtl/savestates*.sv` | SystemVerilog |
| I/O & peripherals | `rtl/ioport.sv`, `rtl/lightgun.sv`, `rtl/miracle.sv` | SystemVerilog |

### Coprocessors (`rtl/chip/`)

Each coprocessor has a Map controller (bus mapping) and a core implementation:

| Chip | Games | Directory |
|------|-------|-----------|
| CX4 | Mega Man X2/X3 | `rtl/chip/CX4/` |
| GSU (SuperFX) | Star Fox, Yoshi's Island, Doom | `rtl/chip/GSU/` |
| SA1 | Super Mario RPG, Kirby Super Star | `rtl/chip/SA1/` |
| SDD1 | Star Ocean, Street Fighter Alpha 2 | `rtl/chip/SDD1/` |
| DSP-1/2/3/4 | Pilotwings, Mario Kart | `rtl/chip/DSP/` |
| SPC7110 | Far East of Eden Zero | `rtl/chip/SPC7110/` |
| MSU-1 | Modern streaming extension | `rtl/chip/MSU1/` |
| BSX | Satellaview | `rtl/chip/BSX/` |
| Sufami Turbo | Dual-cartridge system | `rtl/chip/Sufami/` |

### Clocking

PLL generates from 50 MHz input:
- `clk_mem`: 85.9 MHz (SDRAM)
- `CLK_VIDEO`: 42.95 MHz (video output)
- `clk_sys`: 21.477 MHz (main system clock, NTSC)

PAL mode uses dynamic PLL reconfiguration. Internal timing uses clock enables (`SYSCLKF_CE`, `SYSCLKR_CE`) rather than clock dividers, enabling turbo modes without PLL changes.

### Clock Domain Crossings

Two primary domains: `clk_sys` (21.5 MHz) and `clk_mem` (85.9 MHz). Use 2-FF synchronizers for CDC. GSU cache and CX4 memory use inverted clocks declared as asynchronous groups in `SNES.sdc`.

### Conventions

- Active-low signals use `_N` suffix (e.g., `CPURD_N`, `ROM_CE_N`)
- Core logic is VHDL; peripherals and MiSTer integration are SystemVerilog/Verilog
- `sys/` directory is the shared MiSTer framework — do not modify

## ROM Mapping

ROM type detection happens in `SNES.sv` during loading. Supports LoROM, HiROM, ExHiROM. ROM address mirroring handles non-power-of-2 ROM sizes automatically. The `rom_type` and `rom_mask` signals propagate through the hierarchy to configure memory mapping.
