# SPI Notes

## Scope

These notes define the host-side SPI contract between an ATmega328P and an
MFRC522. They cover electrical assumptions, ATmega328P peripheral settings, and
the MFRC522 register transaction format. RF framing is covered in
[`protocol-notes.md`](protocol-notes.md).

In this document, **controller** refers to the ATmega328P and **peripheral**
refers to the MFRC522. The MFRC522 datasheet uses the older terms master and
slave.

## Electrical Preconditions

- The MFRC522's main analog, digital, and transmitter supplies operate from
  2.5 V to 3.6 V, with 3.3 V typical.
- The MFRC522 digital pin supply, `PVDD`, may be 1.6 V to 3.6 V and must not be
  higher than `DVDD`. A breakout board will normally tie it to 3.3 V.
- MFRC522 input pins must remain between `PVSS - 0.5 V` and `PVDD + 0.5 V`.
  Driving them directly from a 5 V ATmega328P violates that limit.
- The ATmega328P speed-grade curve places the nominal 16 MHz boundary at about
  3.78 V, before allowing for supply and clock tolerances. A 16 MHz MCU at
  3.3 V is outside the guaranteed area. Use logic-level translation between a
  common 5 V MCU configuration and a 3.3 V MFRC522.
- Translate `SCK`, `MOSI`, `NSS`, and any MCU-driven `NRSTPD` signal toward the
  MFRC522. Also verify the `MISO` and optional `IRQ` high levels against the
  ATmega328P input-high requirement; use a suitable buffer if they are not
  guaranteed.
- Both devices must share a ground.
- Do not assume an RC522 module's onboard regulator also translates logic
  levels. Most simple modules do not.

## Signal Connections

| ATmega328P signal | Port pin | MFRC522 signal | Direction during transfer |
| --- | --- | --- | --- |
| `SS` or another GPIO | `PB2` or chosen GPIO | `NSS` | ATmega328P to MFRC522 |
| `MOSI` | `PB3` | `MOSI` | ATmega328P to MFRC522 |
| `MISO` | `PB4` | `MISO` | MFRC522 to ATmega328P |
| `SCK` | `PB5` | `SCK` | ATmega328P to MFRC522 |
| Optional GPIO | project-defined | `NRSTPD` | ATmega328P to MFRC522 |
| Optional interrupt input | project-defined | `IRQ` | MFRC522 to ATmega328P |

`NSS` is active low. The ATmega328P hardware does not automatically drive a
peripheral chip-select in controller mode, so firmware must assert and release
it around each complete MFRC522 transaction.

The bare MFRC522 automatically detects its host interface after power-on or a
hard reset. SPI requires the MFRC522 `I2C` configuration pin low and `EA` high
during that detection. These are static straps, not SPI traffic signals. Many
breakout boards wire them internally, but a custom board must provide and
verify both levels before releasing `NRSTPD`.

Even when another GPIO drives MFRC522 `NSS`, configure the ATmega328P hardware
`SS` pin (`PB2`) as an output and keep it high. If `PB2` remains an input and is
driven low, hardware clears `MSTR`, changes the SPI peripheral to target mode,
and sets `SPIF`.

## Required SPI Mode

The MFRC522 sends bytes most-significant bit first. Data is stable on each
rising edge and may change on each falling edge. Configure the ATmega328P as:

| Setting | Value | ATmega328P control |
| --- | --- | --- |
| Role | Controller | `MSTR = 1` |
| Data order | MSB first | `DORD = 0` |
| Clock idle level | Low | `CPOL = 0` |
| Sample edge | Rising | `CPHA = 0` |
| SPI mode | Mode 0 | `CPOL = 0`, `CPHA = 0` |
| Peripheral enabled | Yes | `SPE = 1` |

The MFRC522 supports an SPI clock up to 10 Mbit/s. At a 16 MHz CPU clock, all
ATmega328P controller divisors fit below that limit:

| `SPI2X` | `SPR1:SPR0` | Divider | SCK at 16 MHz |
| --- | --- | --- | --- |
| 0 | `00` | 4 | 4 MHz |
| 0 | `01` | 16 | 1 MHz |
| 0 | `10` | 64 | 250 kHz |
| 0 | `11` | 128 | 125 kHz |
| 1 | `00` | 2 | 8 MHz |
| 1 | `01` | 8 | 2 MHz |
| 1 | `10` | 32 | 500 kHz |
| 1 | `11` | 64 | 250 kHz |

Starting at 4 MHz is conservative and leaves margin for wiring and level
shifters. An 8 MHz SCK remains within the MFRC522 specification but should only
be selected after signal integrity and translator timing are verified.

## ATmega328P Transfer Semantics

Writing a byte to `SPDR` starts one eight-clock transfer. Completion sets
`SPSR.SPIF`; the received byte is then available in `SPDR`. Every transmitted
byte also receives a byte, even if the caller discards it.

A polling transfer follows this sequence:

1. Write the outgoing byte to `SPDR`.
2. Wait until `SPSR.SPIF` is set.
3. Read `SPDR`, even when the received value is not needed.

Reading `SPSR` while `SPIF` is set and then accessing `SPDR` clears `SPIF`.
Writing `SPDR` before the active transfer finishes sets `SPSR.WCOL` and does not
replace the byte being shifted.

## MFRC522 Address Byte

The first byte transferred after `NSS` goes low selects the operation and the
six-bit register address:

```text
bit:       7       6 5 4 3 2 1       0
         R/W        address           0
```

- Bit 7 is `1` for a read and `0` for a write.
- Bits 6:1 contain the MFRC522 register address (`0x00` to `0x3F`).
- Bit 0 is always `0`.

Equivalent expressions are:

```text
write_address = (register_address << 1) & 0x7E
read_address  = ((register_address << 1) & 0x7E) | 0x80
```

Examples:

| Operation | Register | Register address | SPI address byte |
| --- | --- | --- | --- |
| Write | `TxControlReg` | `0x14` | `0x28` |
| Read | `VersionReg` | `0x37` | `0xEE` |

The address byte is an SPI transport field. It is not written to the MFRC522
FIFO and must not be confused with an RF protocol command.

## Register Write Transaction

For one byte:

```text
NSS low -> write address -> data -> NSS high
```

The returned MISO bytes are unspecified and must be ignored. The MFRC522 also
supports writing several bytes after one address byte:

```text
NSS low -> write address -> data 0 -> data 1 -> ... -> data n -> NSS high
```

Every data byte goes to the same selected register. This is useful for filling
`FIFODataReg`, because each write advances the FIFO's internal write pointer.
It is not general register-address auto-increment.

## Register Read Transaction

SPI is full duplex, so the first returned byte is not the selected register's
value. A single-byte read is:

```text
NSS low -> read address -> 0x00 -> NSS high
MISO:       ignored        data
```

The second MOSI byte is only a clock-generating dummy byte.

The datasheet defines a pipelined multi-register read:

```text
MOSI: address 0, address 1, address 2, ..., address n, 0x00
MISO: ignored,   data 0,    data 1,  ..., data n-1, data n
```

To read several bytes from `FIFODataReg`, repeatedly send its encoded read
address and finish with `0x00`. Each completed FIFO register read advances the
FIFO read pointer. Do not infer ordinary address auto-increment from this
behavior.

## Chip-Select Rules

- Drive `NSS` high while idle and before enabling the SPI peripheral.
- Pull `NSS` low before the address byte.
- Keep `NSS` low for the entire address-plus-data transaction.
- Return `NSS` high only after `SPIF` confirms the final byte is complete.
- Do not toggle `NSS` between an address byte and its data or dummy byte.
- On a shared SPI bus, ensure all other peripheral selects remain inactive.

## Initialization Order

The intended hardware initialization order is:

1. Verify the MFRC522 interface straps are `I2C = low` and `EA = high`.
2. Establish safe inactive output levels, especially `NSS = high`.
3. Configure `PB2`, `PB3`, and `PB5` as outputs and `PB4` as input.
4. Enable SPI controller mode, MSB first, mode 0, at the selected divider.
5. Release MFRC522 `NRSTPD`, if controlled by firmware, and wait for oscillator
   startup.
6. Read `VersionReg` as the first communication check.

For a genuine MFRC522, `VersionReg` is `0x91` for version 1.0 or `0x92` for
version 2.0. A correct value confirms basic SPI communication, not antenna or RF
operation.

## Failure Checklist

| Symptom | Checks |
| --- | --- |
| Always `0x00` | Power, common ground, `NRSTPD`, `NSS`, MISO continuity |
| Always `0xFF` | MISO floating, wrong chip select, peripheral unpowered |
| Bit-shifted values | SPI mode, data order, edge quality, excessive SCK |
| Controller unexpectedly stops | `PB2/SS` direction and `SPCR.MSTR` |
| Reads lag or look like addresses | Missing dummy byte or wrong read pipeline |
| Works slowly but fails at 8 MHz | Wiring, breadboard capacitance, translator bandwidth |
| Device becomes hot or unreliable | 5 V logic applied to MFRC522 pins; disconnect power and fix levels |

## References

- NXP, *MFRC522 Standard performance MIFARE and NTAG frontend*, Rev. 3.9,
  sections 8.1.1, 8.1.2, 12, 13, and 14.1.
- Microchip, *ATmega48A/PA/88A/PA/168A/PA/328/P Data Sheet*, DS40002061A,
  sections 19 and 29.
