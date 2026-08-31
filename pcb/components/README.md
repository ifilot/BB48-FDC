# Project-local KiCad component libraries

This directory contains project-local symbols and carrier-board footprints for
the two Olimex modules:

- `Olimex_RP2350_PICO2_BB48`: base module without PSRAM or microSD
- `Olimex_RP2350_PICO2_BB48R`: module with 8 MB PSRAM and microSD

The project-local `sym-lib-table` and `fp-lib-table` in `pcb/` register both
libraries under the nickname `BB48`.

## Pin numbering

Pad/pin names follow the labels on the Olimex board instead of inventing a
single 1-through-54 sequence:

- `A1` through `A27` correspond to `EXT1.1` through `EXT1.27`.
- `B1` through `B27` correspond to `EXT2.1` through `EXT2.27`.

This keeps schematic pins, footprint pads, and the manufacturer's pinout easy
to cross-check.

| Header | Pins | Signals |
| --- | --- | --- |
| EXT1 | A1-A3 | 3V3, GND, 3V3_EN |
| EXT1 | A4-A27 | GPIO0-GPIO23 |
| EXT2 | B1-B3 | VBUS, VSYS, GND |
| EXT2 | B4-B27 | GPIO24-GPIO47 |

## On-board sharing

The symbols name shared pins explicitly. They remain electrically connected to
the header, but firmware and external circuitry must account for the on-board
load.

| GPIO | Both variants | BB48R additionally |
| --- | --- | --- |
| 0-1 | UEXT UART | - |
| 2-3 | UEXT and Qwiic/Stemma I2C | - |
| 4-7 | UEXT SPI | - |
| 8 | - | PSRAM `QMI_CS1n` |
| 9 | - | microSD `CS` |
| 10 | - | microSD `CLK` |
| 11 | - | microSD `CMD/MOSI` |
| 24 | - | microSD `DAT0/MISO` |
| 25 | User LED | - |

## Mechanical assumptions

- Board outline: 69.0 x 18.3 mm
- Header pitch: 2.54 mm
- Row spacing: 15.24 mm (0.6 inch)
- 27 positions per row
- Carrier drill: 1.0 mm with 1.8 mm annular pads

The origin is the module center. USB-C is at the negative-Y end; header position
1 is at the positive-Y end. The BB48R footprint shows the bottom-side microSD
socket and its keep-clear reminder on documentation/fabrication layers.

Sources: [Olimex product page](https://www.olimex.com/Products/RaspberryPi/PICO/RP2350-PICO2-BB48/open-source-hardware),
[official Rev. A KiCad design](https://github.com/OLIMEX/RP2350-PICO2-BB48/tree/main/HARDWARE/RP2350-PICO2-BB48.REV.A).

## Interface symbols

- `FDC_Interface:FDD_34_HOST` models the host/controller side of a 34-pin
  PC/Shugart floppy connector. Its physical ground contacts are a KiCad pin
  stack: one GND pin is drawn and the remaining ground contacts are hidden at
  the same connection point.
- `P2000T:P2000T_EXPANSION_PORT` models the 40-pin CPU-to-video/extension-board
  header used by the NEBO floppy controller. It follows section 3.8.1 of the
  Field Support Manual and uses the same 1–40 mapping as the supplied NEBO
  reverse-engineering schematic; see the
  [Field Support Manual](https://electrickery.hosting.philpem.me.uk/comp/p2000c/doc/P2000MT_FSupp.pdf).
