# ATmega328P Notes

## Project Baseline

The initial host target is an ATmega328P clocked at 16 MHz and programmed in C11
with `avr-gcc`. The project uses peripheral and GPIO registers directly; it does
not depend on Arduino core functions or an MCU framework.

Only the MCU details that affect the MFRC522 driver are recorded here.

## Clock and Voltage Constraint

The ATmega328P speed-grade endpoints are:

- Up to 4 MHz from 1.8 V.
- Up to 10 MHz from 2.7 V.
- Up to 20 MHz from 4.5 V.

The maximum frequency changes linearly between 2.7 V at 10 MHz and 4.5 V at
20 MHz. Interpolation places the nominal 16 MHz boundary at approximately
3.78 V; a real design must also account for supply and clock tolerances.
Consequently, a 16 MHz ATmega328P is not guaranteed at 3.3 V. A 4.5 V supply is
the endpoint that guarantees operation through 20 MHz, not the exact minimum
for 16 MHz.

This conflicts with a common direct MFRC522 connection. The MFRC522 main supply
is 3.3 V typical with 3.6 V maximum, and its digital inputs are limited relative
to `PVDD`. A 16 MHz, 5 V ATmega328P therefore requires level translation. See
[`spi-notes.md`](spi-notes.md) for signal-level requirements.

Changing the MCU to 3.3 V would require reducing its clock to a value inside the
datasheet safe operating area; simply retaining 16 MHz at 3.3 V is an
out-of-specification design.

## SPI Pins

The ATmega328P hardware SPI peripheral is on port B:

| SPI role | Port bit | 28-pin PDIP | 32-pin TQFP |
| --- | --- | --- | --- |
| `SS` | `PB2` | 16 | 14 |
| `MOSI` | `PB3` | 17 | 15 |
| `MISO` | `PB4` | 18 | 16 |
| `SCK` | `PB5` | 19 | 17 |

Use port names in the driver rather than Arduino board labels. Physical package
pin numbers must remain a board-level concern.

Recommended directions in SPI controller mode are:

| Pin | Direction | Initial level |
| --- | --- | --- |
| `PB2/SS` | Output | High |
| `PB3/MOSI` | Output | Low unless the level translator requires otherwise |
| `PB4/MISO` | Input | No internal pull-up by default |
| `PB5/SCK` | Output | Low for SPI mode 0 |

The MFRC522 `NSS` may use `PB2` or another GPIO. Regardless of that choice,
`PB2` must remain an output in controller mode. If it is an input and reads low,
the ATmega328P clears `SPCR.MSTR` and switches to SPI target mode.

## Relevant GPIO Registers

| Register | Purpose |
| --- | --- |
| `DDRB` | Sets each port B pin as input (`0`) or output (`1`) |
| `PORTB` | Sets output level, or enables an input pull-up |
| `PINB` | Reads current pin levels; writing a one toggles the corresponding output latch |

Set safe output values in `PORTB` before or together with changing `DDRB`, so
`NSS` does not glitch low during startup. Avoid broad whole-register writes if
unrelated port B pins are used by other subsystems.

## SPI Registers

### `SPCR`: SPI Control Register

| Bit | Name | Required or relevant behavior |
| --- | --- | --- |
| 7 | `SPIE` | Enables SPI interrupt; clear for initial polling implementation |
| 6 | `SPE` | Must be set to enable SPI |
| 5 | `DORD` | Clear for MSB-first MFRC522 transfers |
| 4 | `MSTR` | Set for controller mode |
| 3 | `CPOL` | Clear so SCK is low while idle |
| 2 | `CPHA` | Clear to sample on the leading rising edge |
| 1:0 | `SPR1:SPR0` | Selects the base SCK divider |

The required MFRC522 mode is therefore mode 0, MSB first.

### `SPSR`: SPI Status Register

| Bit | Name | Behavior |
| --- | --- | --- |
| 7 | `SPIF` | Set when an eight-bit transfer completes |
| 6 | `WCOL` | Set if `SPDR` is written while a transfer is active |
| 0 | `SPI2X` | Doubles SCK for the selected `SPR1:SPR0` setting |

`SPIF` and `WCOL` are cleared by reading `SPSR` while the flag is set, followed
by accessing `SPDR`. A transfer primitive must preserve this sequence.

### `SPDR`: SPI Data Register

Writing `SPDR` starts a transfer. After `SPIF` is set, reading `SPDR` returns the
byte shifted in from MISO. The transmit side is single-buffered, so software
must not write the next byte before the current transfer completes.

## SCK Selection at 16 MHz

| `SPI2X` | `SPR1:SPR0` | SCK |
| --- | --- | --- |
| 0 | `00` | 4 MHz |
| 0 | `01` | 1 MHz |
| 0 | `10` | 250 kHz |
| 0 | `11` | 125 kHz |
| 1 | `00` | 8 MHz |
| 1 | `01` | 2 MHz |
| 1 | `10` | 500 kHz |
| 1 | `11` | 250 kHz |

The MFRC522 maximum is 10 Mbit/s, making both 4 MHz and 8 MHz valid at the chip
interface. Use 4 MHz initially. Level translators and long wires can impose a
lower practical limit than either device's SPI peripheral.

## Peripheral Power Reduction

`PRR.PRSPI` must be clear for the SPI peripheral clock to run. If power
management later disables SPI, re-enable its clock before accessing `SPCR`,
`SPSR`, or `SPDR` and restore the intended configuration as needed.

The MFRC522's soft/hard power states are independent of the ATmega328P's SPI
power reduction. A working MCU SPI peripheral does not imply that the MFRC522
oscillator is ready.

## Polling and Interrupt Choices

The first driver phase should use bounded polling for ATmega328P `SPIF` and
MFRC522 status flags. Polling keeps ownership and sequencing explicit while the
basic transport is validated.

Every polling loop still needs a finite bound or a higher-level timeout. A
wiring fault should return an error rather than permanently blocking the MCU.
The MFRC522 timer can bound RF waits, but it cannot detect a failed SPI bus;
that requires a host-side bound.

An interrupt-driven revision would involve two independent sources:

- ATmega328P SPI transfer-complete interrupt (`SPI_STC_vect`).
- A GPIO/external or pin-change interrupt connected to MFRC522 `IRQ`.

Those interrupts should not be introduced until the synchronous register path
and its state ownership are stable.

## Resource and Concurrency Rules

- Treat the hardware SPI peripheral as a shared resource if other devices are
  present. Each transaction must establish mode, speed, and chip select without
  disturbing another owner.
- Never allow two chip-select outputs to be active simultaneously.
- Do not change `CPOL`, `CPHA`, `DORD`, or divider bits during a transfer.
- If SPI is used from both main code and an ISR, protect complete transactions,
  not only individual bytes.
- Avoid disabling interrupts for the duration of RF waits. An SPI register
  transaction is short; a card response can be comparatively long.
- Keep card protocol state above the byte-transfer primitive. The SPI layer
  should know nothing about REQA, UIDs, or MIFARE blocks.

## Programming Interface Conflict

The ATmega328P uses `PB3/MOSI`, `PB4/MISO`, and `PB5/SCK` for in-system serial
programming as well as runtime SPI. A permanently attached MFRC522 and its level
translator must not contend with the programmer.

Board design must ensure that:

- MFRC522 `NSS` remains inactive during programming.
- The translator permits the programming direction and voltage levels, or can
  be isolated.
- No peripheral strongly drives MISO while deselected.
- Programmer and target grounds are common.

Series resistors, translator output-enable control, or a disconnect mechanism
may be necessary depending on the circuit.

## Hardware Bring-Up Checks

1. Verify ATmega328P VCC and clock before connecting the MFRC522.
2. Verify the 3.3 V rail under RF load; the antenna transmitter causes a much
   larger load than SPI-only idle operation.
3. Confirm translated `NSS` is high and translated `SCK` is low at idle.
4. Confirm `PB2` is an output and `SPCR.MSTR` remains set.
5. Check a known SPI byte on a logic analyzer using mode 0 and MSB-first decode.
6. Read MFRC522 `VersionReg`.
7. Only after SPI is stable, enable the antenna drivers and begin RF protocol
   work.

## References

- Microchip, *ATmega48A/PA/88A/PA/168A/PA/328/P Data Sheet*, DS40002061A,
  sections 1, 14, 19, and 29.
- NXP, *MFRC522 Standard performance MIFARE and NTAG frontend*, Rev. 3.9,
  sections 4, 8.1.2, 11 through 14.
