# BB48 P2000T memory and floppy-controller design

## Purpose and current status

This board is intended to combine two mostly independent functions:

1. provide banked static RAM to the P2000T; and
2. replace the NEBO floppy-controller electronics with an RP2350-based Olimex BB48 while retaining a physical 34-pin floppy-drive connection.

The schematic currently establishes the signal paths and voltage boundaries for both functions. It does **not** yet contain the ATF1502 equations or BB48 firmware, so address decoding, bank-register behavior, bus-cycle handshaking, and uPD765 emulation are requirements rather than finished logic.

In this document `~{NAME}` means an active-low signal. Internal BB48 floppy signals such as `FDC_STEP` use asserted-high logic; the 74LS06 converts them to the active-low open-collector signals used by the drive.

## High-level architecture

```text
                                  +-----------------------+
 P2000T expansion port J2         | U6 ATF1502AS CPLD     |
                                  |                       |
 A0..A14 ------------------------>| decode and bank latch |---- RAM_A15/A16
 D0..D7 ------------------------->|                       |---- ~RAM_CS
 RAMS2, ~MRQ, ~IORQ,              |                       |---- ~FDC_CS
 ~RD, ~WR, ~M1, ~RES, ~RFSH ---->|                       |---- WAIT_ASSERT
                                  +-----------------------+
            |                                  ^
            |                                  | MCU_READY
            |                                  |
            |       +--------------------------+------------------+
            |       |                                             |
            v       v                                             v
     +------------------+                                  +---------------+
     | U4 128 KiB SRAM  |                                  | A1 Olimex     |
     |                  |                                  | BB48 / RP2350 |
     | A0..A14 direct   |                                  +---------------+
     | A15/A16 from CPLD|                                     ^         |
     | D0..D7 direct    |                                     |         |
     +------------------+                                     |         |
                                                              |         |
 P2000 D0..D7 <====== U5 bidirectional 5 V/3.3 V translator ==+         |
 P2000 controls ------ U7 fixed 5 V-to-3.3 V translator ------+         |
                                                                        |
                    FDD status --> U3 74LVC14 --------------------------+
                    FDD control <-- U1/U2 74LS06 <-----------------------+
```

The memory path does not require the BB48 during normal reads and writes. The floppy path does: the CPLD detects a relevant P2000 I/O cycle, the translators expose it to the BB48, and firmware supplies the emulated controller response.

## Memory expansion

### Required functionality

The memory expansion needs the following functions:

- recognize when the P2000T is accessing the expansion-memory window;
- present static RAM on `D0..D7` only during a valid selected cycle;
- route the CPU address to the RAM;
- turn the P2000T's 15 address bits into a larger physical RAM address by adding bank bits;
- provide a software-writable bank register;
- generate correct SRAM chip-select, output-enable, and write-enable timing;
- reset to a deterministic bank;
- avoid driving the data bus during I/O, refresh, and unrelated memory cycles; and
- tolerate the P2000T's 5 V logic levels.

The original NEBO uses a 32 KiB CY62256 and a one-bit bank latch. Its glue logic decodes writes to ports `94h`-`97h`, mirrored at `D4h`-`D7h`, and uses `D0` as the latched bank control. The new board has a 128 KiB SRAM, so it can retain the NEBO-compatible bit while adding a second bank bit for the extra capacity.

### Current implementation

| Function | Schematic implementation |
| --- | --- |
| Physical memory | U4, `CY62128Exx-xxS`, is a 128 KiB × 8 SRAM powered at 5 V. |
| Low address | P2000T `A0..A14` connect directly to SRAM `A0..A14`. |
| High address | U6 generates `RAM_A15` and `RAM_A16`, connected to SRAM `A15` and `A16`. |
| Data | P2000T `D0..D7` connect directly to U4 and also enter U6 for bank-register writes. |
| Window selection | `RAMS2`, `~{MRQ}`, `~{RFSH}`, and the relevant address/control signals enter U6. U6 must generate `~{RAM_CS}`. |
| Read timing | P2000T `~{RD}` connects directly to SRAM `~{OE}`. |
| Write timing | P2000T `~{WR}` connects directly to SRAM `~{WE}`. |
| Positive chip select | SRAM `CS2` is tied to `+5V`. |
| Reset and clocking | `~{RES}`, `T4M`, and `PHI2` are available to U6. |
| Programming | J4 exposes the ATF1502 JTAG signals. |

With the current direct address wiring, the natural physical address is:

```text
SRAM address = { RAM_A16, RAM_A15, A14..A0 }
```

This divides 128 KiB into four 32 KiB physical banks. That is electrically simple, but it is not automatically identical to the original NEBO's address remapping. The CPLD equations must explicitly choose between:

- exact NEBO-compatible mapping;
- a simpler four-bank, 32 KiB window; or
- a compatibility mode plus an extended banking mode.

### CPLD logic still required

A suitable first implementation for U6 would:

1. reset the two-bit bank register to zero on `~{RES}`;
2. detect an I/O write to the NEBO bank port (`94h`-`97h`, optionally including its mirror);
3. latch `D0` into the compatibility bank bit and `D1` into the extension bank bit;
4. assert `~{RAM_CS}` only when `RAMS2` and a valid memory request select expansion RAM;
5. inhibit RAM selection during refresh and all unrelated cycles; and
6. drive `RAM_A15` and `RAM_A16` from the latched bank value or from compatibility-remapping equations.

The SRAM is fast enough that normal memory reads and writes should not require the BB48 or `~{WAIT}`. Keeping this path completely in the SRAM/CPLD domain makes it deterministic and avoids microcontroller latency.

## FDC emulation

### What must be emulated

To the P2000T, the board must behave like the software-visible part of a NEBO controller. At minimum that includes:

- the uPD765-compatible Main Status Register and data register;
- command, execution, and result phases;
- the `RQM`, `DIO`, `EXM`, and `BUSY` status bits;
- controller reset and the board motor/control latch;
- interrupt generation and acknowledgement;
- data-request or wait-state behavior during byte transfers; and
- the commands required by the P2000T bootstrap and disk software.

The local NEBO reverse-engineering identifies these original I/O regions:

| Ports | Original function |
| --- | --- |
| `88h`-`8Bh` | Z80 CTC |
| `8Ch`-`8Fh` | uPD765 FDC window |
| `90h`-`93h` | NEBO control/status latch |
| `94h`-`97h` | RAM-bank latch, writes only |

The diagnostic cartridge uses `IN 8Ch` for the Main Status Register, `IN/OUT 8Dh` for command/result data, and `OUT 90h` for reset and motor control. Values `04h` and `0Ch` respectively mean reset released with the motor off and reset released with the motor on.

Full software compatibility may also require emulating the parts of the original Z80 CTC that software observes. On the NEBO, the CTC participates in floppy interrupt handling and timing; it is not merely incidental address-decode logic.

### P2000T bus interface

U6 is responsible for recognizing an FDC-related I/O cycle from `A0..A7`, `~{IORQ}`, `~{M1}`, `~{RD}`, and `~{WR}`. It generates `~{FDC_CS}` and can stretch a transaction with `WAIT_ASSERT` until the BB48 reports `MCU_READY`.

U5 is the bidirectional eight-bit data translator:

- its A side is the 5 V P2000T `D0..D7` bus;
- its B side is the 3.3 V `MCU_D0..MCU_D7` bus;
- `~{FDC_CS}` enables it only for selected transactions; and
- `~{RD}` controls its direction.

During a P2000 write, `~{RD}` is high and U5 transfers P2000 A-side data toward the BB48 B side. During a P2000 read, `~{RD}` is low and U5 transfers the BB48 response toward the P2000T.

U7 is permanently enabled in the P2000T-to-BB48 direction. It translates:

- `A0` and `A1`;
- `~{RD}` and `~{WR}`;
- `~{RES}`;
- `PHI`;
- `~{FDC_CS}`; and
- `~{DEW}`.

These become `MCU_A0`, `MCU_A1`, `~{MCU_RD}`, `~{MCU_WR}`, `~{MCU_RES}`, `MCU_PHI`, `~{MCU_FDC_CS}`, and `~{MCU_DEW}` on the BB48.

The handshake back to the P2000T is:

```text
BB48 MCU_READY --> U6 CPLD --> WAIT_ASSERT --> U2 74LS06 --> P2000T ~WAIT
BB48 INT_ASSERT -----------------------------> U2 74LS06 --> P2000T ~INT
```

The 74LS06 outputs are open collector, matching the shared active-low host signals. The necessary pull-up arrangement must be verified against the P2000T backplane and completed in the hardware design.

### Important decode limitation in the current wiring

The BB48 currently receives one decoded select, `~{MCU_FDC_CS}`, and only address bits `MCU_A0:A1`. That is enough to select registers within one four-port window, but it cannot distinguish all of `88h`-`8Bh`, `8Ch`-`8Fh`, and `90h`-`93h` if U6 combines them behind the same select.

Before firmware is finalized, the schematic should therefore provide one of these:

- separate translated selects such as `~{MCU_CTC_CS}`, `~{MCU_FDC_CS}`, and `~{MCU_CTRL_CS}`;
- one broader select plus another translated address bit that uniquely identifies the blocks; or
- an encoded register-block identifier generated by U6 and safely level-shifted to the BB48.

One possible trade is to replace a less important always-translated signal, such as `~{DEW}`, with an additional address or select signal. That choice should be made only after determining how much CTC/DEW compatibility the target software requires.

### Physical floppy-drive interface

The BB48 replaces the original uPD765, WD9216 data separator, write-clock generation, and drive-control logic. Its firmware and RP2350 PIO/state machines must perform the real-time floppy work.

#### Outputs to the drive

BB48 asserted-high signals pass through U1 and U2, both 74LS06 open-collector inverters:

| BB48-side signal | 34-pin drive signal |
| --- | --- |
| `FDC_DENSEL` | `~{FDD_DENSEL}` |
| `FDC_DRVSA` | `~{FDD_DRVSA}` |
| `FDC_DRVSB` | `~{FDD_DRVSB}` |
| `FDC_MOTEA` | `~{FDD_MOTEA}` |
| `FDC_MOTEB` | `~{FDD_MOTEB}` |
| `FDC_DIR` | `~{FDD_DIR}` |
| `FDC_STEP` | `~{FDD_STEP}` |
| `FDC_WDATA` | `~{FDD_WDATA}` |
| `FDC_WGATE` | `~{FDD_WGATE}` |
| `FDC_SIDE1` | `~{FDD_SIDE1}` |

The present connector follows the common PC-style two-drive assignment with `DRVSA/DRVSB` and `MOTEA/MOTEB`. This differs from the original NEBO drawing's four decoded drive-select outputs and should be accounted for in firmware and cabling expectations.

#### Inputs from the drive

Drive status and read-data signals pass through series resistors and U3, a 74LVC14 Schmitt-trigger inverter:

| 34-pin drive signal | BB48-side signal |
| --- | --- |
| `~{FDD_INDEX}` | `FDC_INDEX` |
| `~{FDD_TRK0}` | `FDC_TRK0` |
| `~{FDD_RDATA}` | `FDC_RDATA` |
| `~{FDD_DSKCHG}` | `FDC_DSKCHG` |
| `~{FDD_WPT}` | `FDC_WPT` |

The Schmitt inputs clean up asynchronous and potentially slow/noisy drive signals. The exact 74LVC14 device fitted must guarantee 5 V-tolerant inputs when powered from 3.3 V. U3's unused sixth inverter is tied to a defined input level.

### Required BB48 firmware

The firmware naturally splits into two layers.

The P2000T-facing layer must:

- service decoded bus reads and writes without violating P2000 timing;
- expose uPD765-compatible registers and state transitions;
- implement the NEBO control/status register;
- emulate the required CTC-visible behavior or document why it is unnecessary;
- coordinate `MCU_READY`, `WAIT_ASSERT`, and `INT_ASSERT`; and
- reset to the state expected by the P2000 monitor ROM.

The drive-facing layer must:

- select and spin the requested drive;
- step and track the head position;
- monitor index, track zero, write protect, and disk change;
- recover FM/MFM data from `FDC_RDATA` with deterministic timing;
- generate correctly timed `FDC_WDATA` and `FDC_WGATE` for writes; and
- implement sector search, CRC, read, write, format, and error reporting as required by the uPD765 command set.

PIO and DMA are preferable for read-data capture and write-data generation. Servicing raw floppy pulses solely from ordinary CPU interrupts would introduce too much timing jitter.

### Typical emulated I/O transaction

1. The P2000T starts an I/O cycle.
2. U6 decodes the address and asserts the appropriate controller select.
3. U6 asserts `WAIT_ASSERT` if the BB48 response is not ready.
4. U5 is enabled and points in the direction selected by `~{RD}`.
5. The BB48 consumes a written byte or places a read byte on `MCU_D0..MCU_D7`.
6. The BB48 asserts `MCU_READY`; U6 releases `~{WAIT}` at a bus-safe point.
7. When a command completes or requires service, the BB48 raises `INT_ASSERT`, causing the 74LS06 to pull P2000T `~{INT}` low.

The exact timing of steps 3-6 belongs in the CPLD equations. The CPLD should capture the request and provide deterministic setup/hold timing; firmware should not be expected to notice a short, unstretched Z80 bus pulse reliably.

## Responsibility summary

| Part | Responsibility |
| --- | --- |
| J2 | P2000T expansion-bus connection |
| U4 | 128 KiB 5 V SRAM |
| U6 | Address decoding, RAM banking, FDC transaction capture, and wait-state generation |
| U5 | Bidirectional P2000T/BB48 data-bus level translation |
| U7 | Fixed-direction P2000T-to-BB48 control translation |
| A1 | uPD765/NEBO behavior and physical floppy data processing |
| U1, U2 | Open-collector drive controls plus host `~{INT}` and `~{WAIT}` |
| U3 | Schmitt-trigger conditioning and 5 V-to-3.3 V drive-status input path |
| J1 | Standard 34-pin floppy-drive connection |
| J4 | ATF1502 JTAG programming connection |

## Implementation checklist

- [ ] Decide exact NEBO-compatible versus extended SRAM address mapping.
- [ ] Write and simulate the ATF1502 bank-register and `~{RAM_CS}` equations.
- [ ] Define separate or encoded BB48 selects for the CTC, FDC, and control windows.
- [ ] Write and simulate the CPLD request/ready/`~{WAIT}` state machine.
- [ ] Confirm reset levels and safe power-up behavior for every CPLD output.
- [ ] Verify U5 direction and output-enable timing on read, write, and deselection.
- [ ] Verify the selected 74LVC14 input-voltage rating at a 3.3 V supply.
- [ ] Determine where open-collector pull-ups for the host and floppy buses reside.
- [ ] Implement the uPD765 register/command state machine in BB48 firmware.
- [ ] Implement the NEBO control latch and required CTC compatibility.
- [ ] Implement and test PIO/DMA floppy read and write timing.
- [ ] Validate first with the timeout-protected NEBO diagnostic cartridge, then with the P2000 monitor bootstrap and JWS-DOS.

## Reference behavior used for this design

The functional requirements above were cross-checked against the local `p2000t-nebo-reverse-engineering` repository, particularly its reconstructed NEBO schematic, 82S123 decode analysis, and FDC diagnostic cartridge. The diagnostic's minimum proven interface is:

```text
IN  8Ch       read uPD765 Main Status Register
IN/OUT 8Dh   read result bytes or write command bytes
OUT 90h,04h  release FDC reset, motor off
OUT 90h,0Ch  release FDC reset, motor on
OUT 94h      original NEBO RAM-bank latch family
```

This is a useful first compatibility target, but successful diagnostics do not by themselves prove full JWS-DOS compatibility. Full compatibility depends on command coverage, interrupts, transfer timing, CTC behavior, and the original memory mapping.
