# Monitor "D1" — Firmware Reverse Engineering (Realtek RTD2525BE / 8051)

Hardware security / right-to-repair research on a no-name OEM USB-C monitor
("D1"). Two real defects were investigated from the firmware side: a USB-C
Power-Delivery charging bug and a start-up backlight bug where the panel boots
at maximum brightness. The main work is a deep reverse-engineering pass on the
monitor's **Realtek RTD2525BE scaler (8051 core)** firmware, dumped from its
external SPI flash.

Done on a monitor I own, with verified firmware backups kept at every step.

---

## TL;DR

An **honest, in-depth RE case study with a documented negative result.** The
brightness bug was traced deep into the 8051 firmware: I dumped the SPI flash,
reconstructed the boot chain in Ghidra, **decoded the chip's bank-switching
mechanism**, identified the brightness variable, and formed a well-supported
root-cause hypothesis — but two hardware patch attempts did **not** fix it, and
I document exactly why (including catching that one of my own tests was a no-op
and therefore proved nothing). The remaining explanation is a hardware-autonomous
backlight default that the firmware never overrides at boot. The USB-C charging
bug was diagnosed but not fixed.

## Board / silicon

| Chip | Role | Notes |
|---|---|---|
| **Realtek RTD2525BE** | Video scaler, LVDS panel driver | 8051 core for OSD/system mgmt; target of the brightness RE |
| **Chipsea CPW3220** | USB-C PD controller | ARM Cortex-M0 (CS32G02x family); governs the charging bug |
| **Genesys GL850G** | USB 2.0 hub | Not relevant to the bugs |
| **Zbit ZB25VQ80** | External SPI flash, 1 MB, 3 V | Holds the 8051 firmware; **dumped successfully** |

## Bug 1 — USB-C PD charging (CPW3220): diagnosed, not fixed

The laptop would not charge from the monitor's USB-C. Findings: the port's
`power_role` was stuck on `source` instead of `sink` (worked around by
re-plugging — a UCSI/kernel negotiation glitch), after which PD stayed pinned at
~250 mA and never completed a real power contract. This is consistent with the
CPW3220 not advertising its PDOs correctly, matching the kernel log
`ucsi_acpi: GET_CONNECTOR_CAPABILITY failed (-110)`. Fixing it would require
dumping/patching the CPW3220 (updatable via Chipsea's USB-C tool, SWD, or a
proprietary serial interface) — not attempted.

## Bug 2 — boot brightness (RTD2525BE 8051): the main investigation

### Fault isolation
Reproduced with **HDMI only, no USB-C and no host involvement** → the bug is
entirely internal to the monitor firmware, triggered on video-signal lock. Host,
driver, DDC-CI and USB-C were ruled out.

### Flash dump
CH341A (black ZIF variant) + SOP8 clip + SOIC8→DIP8 adapter, read with
`flashrom -p ch341a_spi`. **Two independent dumps with identical SHA-256**
(1,048,576 bytes, ~66% empty `0xFF`) — a stable, trustworthy read.

### Static analysis (Ghidra, 8051 big-endian)
- Reset vector `LJMP 0x26EB`; reconstructed the startup → main-superloop chain;
  ~600 functions; **no ASCII strings** (the OSD uses bitmap glyphs).
- **Bank-switching decoded** — the key unlock: XDATA `0xFFFF` is the
  bank-select register; a family of call-gates (`0x2200/0x2210/…`) and
  trampolines (`0x03f7/…`) implement cross-bank calls by selecting a 64 KB flash
  page and saving/restoring the caller's bank on the stack. (An earlier guess
  that this family was a generic watchdog/state tag was wrong and corrected.)
- Identified the brightness variable **`DAT_EXTMEM_e65f`**, set by three
  independent OSD/remote step paths (coarse steps of 12: 0x91, 0x9d, …, **0xc1
  = 193**, …) and reset to the factory default 0x91 (145) by the firmware's own
  "restore defaults" routine; the XINIT table initializes it to 145 at boot.

### Hardware patch tests — both failed, documented
1. **Patch #1** — force `ff10/12/14/16` from `0xFF` to `0x82` (the only four
   registers written once at boot by `boot_time_peripheral_init_ONCE`): no
   visible change.
2. **Patch #2** — a code-cave hook forcing `e65f = 145` at boot: no change — and
   later found to be a **no-op**, because the XINIT table already sets `e65f =
   145`. That "failed" test therefore proves nothing about whether `e65f` drives
   the backlight — explicitly noted so a wrong conclusion isn't drawn from it.

### Root-cause hypothesis (most consistent with all symptoms)
The backlight **apply/dispatch chain only runs when the user changes the value**
(OSD/remote), never at boot. At power-on the LED driver sits at its hardware
default (bright, read as ~193); `e65f` is re-initialized to 145 each boot, never
persisted and never auto-applied. A correct fix must **trigger the apply
dispatch**, not merely set the variable — which is the additional reason patch
#2 could not have worked.

### Decisive next test
A single-byte change to the `e65f` XINIT default (file offset `0x1f85`) to a very
different value, reflashed, disambiguates the three remaining hypotheses: whether
`e65f` is the OSD-displayed value, whether it drives the physical backlight, or
whether 193 is a pure hardware default and `e65f` is only a shadow.

## Why this is interesting

Full hardware-to-firmware RE **without source, SDK or headers**: physical chip
identification, SPI flash extraction, 8051 static analysis, and an 8051
bank-switching scheme decoded from scratch — then hypothesis-driven **binary
patching** (hooking via a code cave, no addresses shifted) validated on real
hardware with a verified rollback backup. Just as important, the write-up is
honest about two failed patches and self-corrects a flawed test rather than
claiming a result.

## Tools

CH341A + flashrom · SOP8 clip / SOIC8→DIP8 · Ghidra (8051:BE) · GhidraMCP ·
multimeter

## Repository layout

```
notes.md                 Full reverse-engineering work log (findings, boot chain, patches)
ghidra_code/             Decompilation notes and analysis (no vendor firmware binaries)
ch341-i2c-spi-gpio/      CH341A I2C/SPI kernel driver used for the ISP header work
RTDMultiProg/            RTD2660/2662 programmer tool (not used in the end)
```

> **Not included (copyright):** the raw vendor firmware dumps
> (`dumps/*.bin`, `bank0.bin`) and the patched images are OEM/Realtek firmware
> and are **not** redistributed. The analysis, the reconstructed logic, and the
> exact patch offset/byte tables are in `notes.md`; the dumps are reproducible
> from the device with the setup above.

## Ethics & scope

Performed on a monitor I own, for repair/research purposes, with verified
backups of the original firmware at every step. No vendor firmware is
redistributed.

## License

See [`LICENSE`](LICENSE).
