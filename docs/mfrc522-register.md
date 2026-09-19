# MFRC522 Register Notes

## Scope

The MFRC522 exposes 64 byte-addressed registers from `0x00` through `0x3F`.
They configure the host interface, command state machine, FIFO, contactless
UART, analog front end, timer, and test functions. This document is a design
index for the future driver; the NXP datasheet remains authoritative for every
bit definition and electrical condition.

Register addresses are independent of the SPI read/write address byte. See
[`spi-notes.md`](spi-notes.md) for the required address encoding.

## Register Access Behavior

The datasheet classifies bits by behavior:

| Mark | Meaning |
| --- | --- |
| `R/W` | Host-readable and host-writable control bit |
| `D` | Dynamic; readable/writable and possibly changed by an internal state machine |
| `R` | Read-only status |
| `W` | Write-only action; reads return zero |
| Reserved | Do not modify; write zero where a write cannot be avoided |
| `RFT` | Reserved for future use or production test; do not modify |

Do not apply a generic read-modify-write operation to action, dynamic,
interrupt, or reserved fields. In particular, interrupt flags use special
set/clear semantics and FIFO reads and writes have side effects.

## Complete Register Map

### Page 0: Command and Status

| Address | Register | Purpose |
| --- | --- | --- |
| `0x00` | Reserved | Do not use |
| `0x01` | `CommandReg` | Starts/stops commands; receiver and soft-power controls |
| `0x02` | `ComIEnReg` | Enables communication interrupt sources on `IRQ` |
| `0x03` | `DivIEnReg` | Enables auxiliary interrupt sources and configures `IRQ` output |
| `0x04` | `ComIrqReg` | Communication interrupt flags |
| `0x05` | `DivIrqReg` | CRC and MFIN interrupt flags |
| `0x06` | `ErrorReg` | Error status from the last command |
| `0x07` | `Status1Reg` | CRC, IRQ, timer, and FIFO status |
| `0x08` | `Status2Reg` | Crypto1 and modem state |
| `0x09` | `FIFODataReg` | Data port for the 64-byte FIFO |
| `0x0A` | `FIFOLevelReg` | FIFO count and flush action |
| `0x0B` | `WaterLevelReg` | FIFO high/low warning threshold |
| `0x0C` | `ControlReg` | Timer actions and received trailing-bit count |
| `0x0D` | `BitFramingReg` | Starts transceive and controls partial-bit frames |
| `0x0E` | `CollReg` | First RF collision position and collision handling |
| `0x0F` | Reserved | Do not use |

### Page 1: Communication

| Address | Register | Purpose |
| --- | --- | --- |
| `0x10` | Reserved | Do not use |
| `0x11` | `ModeReg` | General transmit/receive and CRC preset settings |
| `0x12` | `TxModeReg` | Transmit data rate, framing CRC, and modulation inversion |
| `0x13` | `RxModeReg` | Receive data rate, framing CRC, and receive behavior |
| `0x14` | `TxControlReg` | Antenna-driver enable and polarity controls |
| `0x15` | `TxASKReg` | Transmit modulation, including forced 100% ASK |
| `0x16` | `TxSelReg` | Selects antenna-driver and `MFOUT` sources |
| `0x17` | `RxSelReg` | Selects receiver source and post-transmit receive delay |
| `0x18` | `RxThresholdReg` | Decoder minimum and collision thresholds |
| `0x19` | `DemodReg` | Demodulator and PLL settings |
| `0x1A` | Reserved | Do not use |
| `0x1B` | Reserved | Do not use |
| `0x1C` | `MfTxReg` | MIFARE transmit timing |
| `0x1D` | `MfRxReg` | MIFARE parity behavior |
| `0x1E` | Reserved | Do not use |
| `0x1F` | `SerialSpeedReg` | UART host-interface speed; irrelevant to SPI |

### Page 2: Configuration

| Address | Register | Purpose |
| --- | --- | --- |
| `0x20` | Reserved | Do not use |
| `0x21` | `CRCResultRegH` | High byte of CRC coprocessor result |
| `0x22` | `CRCResultRegL` | Low byte of CRC coprocessor result |
| `0x23` | Reserved | Do not use |
| `0x24` | `ModWidthReg` | Modulation width at 106 kBd |
| `0x25` | Reserved | Do not use |
| `0x26` | `RFCfgReg` | Receiver gain |
| `0x27` | `GsNReg` | Antenna-driver conductance during modulation |
| `0x28` | `CWGsPReg` | P-driver conductance without modulation |
| `0x29` | `ModGsPReg` | P-driver conductance during modulation |
| `0x2A` | `TModeReg` | Timer mode and high prescaler bits |
| `0x2B` | `TPrescalerReg` | Timer low prescaler byte |
| `0x2C` | `TReloadRegH` | Timer reload high byte |
| `0x2D` | `TReloadRegL` | Timer reload low byte |
| `0x2E` | `TCounterValRegH` | Current timer high byte |
| `0x2F` | `TCounterValRegL` | Current timer low byte |

### Page 3: Test Registers

| Address | Register | Purpose |
| --- | --- | --- |
| `0x30` | Reserved | Do not use |
| `0x31` | `TestSel1Reg` | Test-signal selection |
| `0x32` | `TestSel2Reg` | Test-signal and PRBS configuration |
| `0x33` | `TestPinEnReg` | Test output enables for parallel pins |
| `0x34` | `TestPinValueReg` | Test values for parallel pins |
| `0x35` | `TestBusReg` | Internal test-bus status |
| `0x36` | `AutoTestReg` | Digital self-test control |
| `0x37` | `VersionReg` | Chip type and silicon version |
| `0x38` | `AnalogTestReg` | `AUX1` and `AUX2` test routing |
| `0x39` | `TestDAC1Reg` | Test DAC 1 value |
| `0x3A` | `TestDAC2Reg` | Test DAC 2 value |
| `0x3B` | `TestADCReg` | I/Q ADC test result |
| `0x3C`-`0x3F` | Reserved | Production test; do not use |

Normal operation should avoid page 3 except for reading `VersionReg` and a
deliberately implemented self-test procedure.

## Command Codes

Write a command code to `CommandReg.Command[3:0]`:

| Code | Command | Behavior |
| --- | --- | --- |
| `0x0` | `Idle` | No action; cancels the current command |
| `0x1` | `Mem` | Transfers 25 bytes between FIFO and internal buffer |
| `0x2` | `GenerateRandomID` | Generates a 10-byte random ID in the internal buffer |
| `0x3` | `CalcCRC` | Runs the CRC coprocessor over FIFO data |
| `0x4` | `Transmit` | Sends FIFO data |
| `0x7` | `NoCmdChange` | Changes non-command `CommandReg` bits without stopping a command |
| `0x8` | `Receive` | Enables the receiver and waits for a frame |
| `0xC` | `Transceive` | Transmits FIFO data, then receives a response |
| `0xE` | `MFAuthent` | Executes MIFARE Classic authentication |
| `0xF` | `SoftReset` | Resets registers and the command state machine |

`0x5`, `0x6`, and `0x9` through `0xD` other than `0xC` are not commands for
normal use. `0xD` is explicitly reserved.

The FIFO is not automatically cleared when a command starts. The driver must
flush or preserve it intentionally.

## Registers Required by the Core Driver

### `CommandReg` (`0x01`)

- `RcvOff` at bit 5 disables the analog receiver when set.
- `PowerDown` at bit 4 enters soft power-down. Clearing it begins wake-up; it
  remains set until the oscillator and device are ready.
- `Command[3:0]` selects the active command.

Reset value: `0x20`.

### Interrupt Registers (`0x02` to `0x05`)

`ComIrqReg` flags are, from bit 6 to bit 0: `TxIRq`, `RxIRq`, `IdleIRq`,
`HiAlertIRq`, `LoAlertIRq`, `ErrIRq`, and `TimerIRq`. `DivIrqReg.CRCIRq` is bit
2.

For either IRQ flag register, bit 7 (`Set1` or `Set2`) selects software action:

- Write a flag mask with bit 7 clear to clear the marked flags.
- Write a flag mask with bit 7 set to set the marked flags.

Do not clear these registers by writing back an unmodified value that was read;
that can apply the action to every set flag.

The enable registers control propagation to the physical `IRQ` pin. Status
flags themselves can still be polled without enabling the pin.

### `ErrorReg` (`0x06`)

| Bit | Name | Meaning |
| --- | --- | --- |
| 7 | `WrErr` | Invalid FIFO write timing or FIFO access during authentication |
| 6 | `TempErr` | Overtemperature; antenna drivers are switched off |
| 4 | `BufferOvfl` | Attempt to write to a full FIFO |
| 3 | `CollErr` | RF collision at 106 kBd |
| 2 | `CRCErr` | Received CRC did not match while CRC checking was enabled |
| 1 | `ParityErr` | Parity check failed at 106 kBd |
| 0 | `ProtocolErr` | Framing/protocol error |

Command execution clears all error bits except `TempErr`. Do not treat an IRQ
completion alone as success; inspect `ErrorReg` and the expected response size.

### FIFO Registers (`0x09`, `0x0A`)

`FIFODataReg` is the single data port into and out of a 64-byte FIFO. A write
increments its write pointer; a read advances its read pointer.

`FIFOLevelReg[6:0]` reports bytes currently stored. Writing bit 7
(`FlushBuffer`) resets both FIFO pointers and clears `BufferOvfl`; reading bit 7
always returns zero.

The host and command state machine share this FIFO. Do not access it while a
command owns it unless the command's documented streaming behavior requires
that access.

### `ControlReg` (`0x0C`)

- Writing bit 7 stops the timer immediately.
- Writing bit 6 starts the timer immediately.
- `RxLastBits[2:0]` reports valid bits in the final received FIFO byte. Zero
  means all eight bits are valid. This field is essential for four-bit ACK/NAK
  responses.

### `BitFramingReg` (`0x0D`)

- Writing `StartSend` at bit 7 starts transmission for an active `Transceive`
  command.
- `RxAlign[2:0]` chooses where the first received bit is stored during bitwise
  anticollision.
- `TxLastBits[2:0]` specifies how many bits of the final FIFO byte to transmit.
  Zero means all eight; REQA and WUPA require seven.

### `CollReg` (`0x0E`)

- `ValuesAfterColl` controls whether bits received after a collision remain
  available. It is used only by 106 kBd bitwise anticollision.
- `CollPosNotValid` indicates that no usable collision position exists.
- `CollPos[4:0]` gives the first collided data-bit position. A value of zero
  means bit 32, not bit zero.

### RF Configuration (`0x11` to `0x19`, `0x26` to `0x29`)

- `ModeReg.CRCPreset = 01b` selects preset `0x6363` for explicit ISO/IEC 14443 A
  `CalcCRC` operations. During RF communication, framing mode selects the
  appropriate preset automatically.
- `TxModeReg.TxCRCEn` and `RxModeReg.RxCRCEn` enable framing CRC generation and
  checking.
- `TxModeReg.TxSpeed` and `RxModeReg.RxSpeed` select 106, 212, 424, or 848 kBd.
- `TxControlReg.Tx1RFEn` and `Tx2RFEn` enable the two antenna drivers.
- `TxASKReg.Force100ASK` forces the 100% ASK modulation used for Type A polling.
- `RFCfgReg.RxGain[2:0]` controls receiver gain. Higher numeric field values do
  not form a simple monotonic dB scale; use the datasheet table.

Analog tuning values are board- and antenna-dependent. Do not blindly treat
values copied from another breakout or library as universal defaults.

### Timer (`0x2A` to `0x2F`)

The timer clock is derived from 13.56 MHz. The 12-bit prescaler combines
`TModeReg.TPrescaler_Hi[3:0]` and `TPrescalerReg`. The 16-bit reload combines
`TReloadRegH` and `TReloadRegL`.

With `DemodReg.TPrescalEven = 0`, the timer period is:

```text
period = (2 * TPrescaler + 1) * (TReload + 1) / 13.56 MHz
```

With `TPrescalEven = 1` on version 2.0 silicon, the first term becomes
`2 * TPrescaler + 2`. `TimerIRq` is set when the counter decrements from one to
zero. A timer expiry reports an event; it does not automatically cancel the RF
receiver, so the host must terminate the command.

### `VersionReg` (`0x37`)

Known genuine MFRC522 values are:

| Value | Device |
| --- | --- |
| `0x91` | MFRC522 version 1.0 |
| `0x92` | MFRC522 version 2.0 |

Other values can indicate broken SPI communication, a powered-down device, or
a compatible/clone part. The value is a diagnostic, not proof that the RF path
works.

## Generic Transceive Sequence

A future register-level transceive operation should follow this state flow:

1. Write `Idle` to stop any previous command.
2. Clear relevant `ComIrqReg` flags.
3. Flush the FIFO.
4. Configure `BitFramingReg`, CRC behavior, and timer for this frame.
5. Write the outbound bytes to `FIFODataReg`.
6. Write `Transceive` to `CommandReg`.
7. Set `BitFramingReg.StartSend`.
8. Poll IRQ flags or wait on the `IRQ` pin until receive, idle, error, or timer
   completion.
9. Inspect and retain `ErrorReg`, `FIFOLevelReg`, and
   `ControlReg.RxLastBits`; also retain `CollReg` when a collision is reported.
10. Write `Idle` when the operation must be terminated explicitly. Do this only
    after retaining the error state because starting a command clears error
    flags other than `TempErr`.
11. Read exactly the previously reported response bytes from `FIFODataReg`.

Completion criteria vary by command. For example, `MFAuthent` blocks normal Tx
and Rx IRQs and may not terminate on a missing card response; automatic timer
operation and `TimerIRq` are required.

## CRC Coprocessor Sequence

For an explicit CRC calculation:

1. Stop the current command with `Idle`.
2. Clear `DivIrqReg.CRCIRq`.
3. Flush the FIFO and write the bytes to check.
4. Configure `ModeReg.CRCPreset` as required.
5. Start `CalcCRC`.
6. Wait for `DivIrqReg.CRCIRq` or `Status1Reg.CRCReady`, with a host-side
   timeout.
7. Read `CRCResultRegL` and `CRCResultRegH` in the byte order required by the RF
   protocol.
8. Return the command state to `Idle`; `CalcCRC` does not stop merely because
   the FIFO becomes empty.

## Reset and Power Notes

- Pulling `NRSTPD` low enters hard power-down. A rising edge releases reset.
- The reset-low pulse must be at least 100 ns.
- After clock startup, the MFRC522 needs 1024 oscillator clocks before register
  access; crystal startup time is additional.
- `SoftReset` restores register reset values and terminates automatically.
- Soft power-down retains registers, FIFO contents, and configuration.
- Clearing `CommandReg.PowerDown` starts wake-up; poll until hardware clears
  the bit rather than assuming wake-up is immediate.
- The antenna field is off unless both `TxControlReg.Tx1RFEn` and `Tx2RFEn` are
  enabled for the intended antenna circuit.

## References

- NXP, *MFRC522 Standard performance MIFARE and NTAG frontend*, Rev. 3.9,
  sections 8 through 10.
