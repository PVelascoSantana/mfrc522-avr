# Protocol Notes

## Scope and Layer Boundaries

The MFRC522 supports the RF signaling and framing used by ISO/IEC 14443 A,
MIFARE, and NTAG products. It does not autonomously run the complete card
selection or application protocol. Host firmware must issue frames in the
correct order and validate every response.

Three distinct command layers must remain separate:

| Layer | Example | Transport |
| --- | --- | --- |
| MFRC522 SPI register operation | Read `VersionReg` | SPI only |
| MFRC522 state-machine command | `Transceive = 0x0C` | Written to `CommandReg` |
| PICC RF command | `REQA = 0x26`, `READ = 0x30` | FIFO, then transmitted over RF |

An RF command byte must never be written to `CommandReg`, and an MFRC522 command
code must never be sent to a card as FIFO payload.

These notes focus on 106 kBd Type A activation and the protocol concepts needed
for the initial driver. They are not a replacement for ISO/IEC 14443 or the
datasheet of the specific card product.

## Terminology

- **PCD**: proximity coupling device, here the MFRC522 reader and its antenna.
- **PICC**: proximity integrated circuit card, tag, or transponder.
- **UID**: PICC identifier, 4, 7, or 10 bytes in the Type A selection procedure.
- **ATQA**: Answer To Request, returned after REQA or WUPA.
- **SAK**: Select Acknowledge, returned after SELECT.
- **ATS**: Answer To Select, returned after RATS by ISO-DEP-capable cards.
- **CRC_A**: the Type A 16-bit frame CRC.
- **BCC**: block check character; XOR of the four bytes in an anticollision
  cascade-level block.
- **CT**: cascade tag byte `0x88`, used inside a cascade level when the UID
  continues.
- **NVB**: number of valid bits field in an anticollision or SELECT frame.

## RF Physical and Framing Facts

- Carrier frequency: 13.56 MHz.
- At 106 kBd, PCD-to-PICC communication uses 100% ASK and modified Miller
  coding.
- At 106 kBd, PICC-to-PCD communication uses 13.56 MHz / 16 subcarrier load
  modulation and Manchester coding.
- Each complete data byte at 106 kBd is followed by an odd parity bit. The
  MFRC522 normally generates and checks parity in hardware.
- Bytes are transmitted least-significant bit first over the RF interface. This
  is unrelated to SPI, which transfers each host-interface byte MSB first.
- Type A supports higher rates, but initial activation begins at 106 kBd.

## CRC_A

CRC_A uses the polynomial `x^16 + x^12 + x^5 + 1`. The MFRC522 CRC coprocessor
supports the required Type A preset `0x6363`; for RF communication the selected
framing mode can choose the preset automatically.

The two CRC bytes are appended least-significant byte first on the RF frame.
Whether CRC is present depends on the command:

| Frame | CRC_A |
| --- | --- |
| REQA or WUPA | No |
| ATQA | No |
| ANTICOLLISION command/response | No |
| Full SELECT command | Yes |
| SAK | Yes |
| HLTA | Yes |
| RATS and ATS | Yes |
| MIFARE/NTAG memory commands | Product command dependent, normally yes |

Do not append CRC twice. Either enable MFRC522 transmit CRC generation for the
frame or calculate and append it explicitly, but not both. Receive-side CRC
checking similarly must match the expected response type.

## Type A Card States

A useful host-side model is:

```text
POWER-OFF -> IDLE -> READY -> ACTIVE -> HALT
                 anti-collision/select
```

- A PICC enters `IDLE` when energized and ready to answer a request.
- `REQA` requests cards in `IDLE`; a card in `HALT` does not answer it.
- `WUPA` requests cards in either `IDLE` or `HALT`.
- A valid request moves responding cards toward `READY` and anticollision.
- Completing all required cascade levels selects one PICC into `ACTIVE`.
- `HLTA` asks the selected PICC to enter `HALT`.

The RF field should remain stable through activation and subsequent card
commands. Removing the field powers down passive cards and resets their state.

## Request and Wake-Up

| Command | Value | Length | Expected response |
| --- | --- | --- | --- |
| `REQA` | `0x26` | 7 bits | 2-byte ATQA |
| `WUPA` | `0x52` | 7 bits | 2-byte ATQA |

For either request:

- Set `BitFramingReg.TxLastBits` to 7.
- Do not transmit CRC_A.
- Expect exactly two complete response bytes.
- Restore byte-aligned framing before ordinary commands.

ATQA contains protocol capability information, but it is not a globally unique
card-type identifier. Different products can share an ATQA, and clone products
may report misleading values. Use the complete activation result and the
specific product protocol instead of identifying a card from ATQA alone.

## Anticollision and Selection

### Cascade Levels

| Cascade level | SEL byte | UID block content before BCC |
| --- | --- | --- |
| CL1 | `0x93` | First four UID-related bytes |
| CL2 | `0x95` | Next four UID-related bytes |
| CL3 | `0x97` | Final four UID bytes |

Each anticollision block contains four UID-related bytes followed by one BCC.
`BCC = byte0 XOR byte1 XOR byte2 XOR byte3`.

UID layout by size:

| UID size | CL1 | CL2 | CL3 |
| --- | --- | --- | --- |
| 4 bytes | `UID0 UID1 UID2 UID3` | Not used | Not used |
| 7 bytes | `CT UID0 UID1 UID2` | `UID3 UID4 UID5 UID6` | Not used |
| 10 bytes | `CT UID0 UID1 UID2` | `CT UID3 UID4 UID5` | `UID6 UID7 UID8 UID9` |

`CT` is `0x88` and is not part of the UID returned to an application.

### No-Collision Path for One Cascade Level

1. Send `SEL`, `NVB = 0x20` to request all 32 UID-related bits and the BCC.
2. Receive five bytes and verify the BCC.
3. Send `SEL`, `NVB = 0x70`, the same five bytes, then CRC_A.
4. Receive one SAK byte and CRC_A; validate the CRC.
5. If SAK bit 2, the cascade bit (`0x04`), is set, continue with the next
   cascade level.
6. If the cascade bit is clear, selection is complete and the accumulated UID
   has 4, 7, or 10 bytes according to the levels traversed.

`NVB` encodes the number of complete bytes in its high nibble and additional
valid bits in its low nibble. `0x20` therefore means two complete command bytes
and no known UID bits; `0x70` means the seven complete bytes preceding CRC in a
full SELECT frame.

### Collision Path

When several PICCs respond, the MFRC522 reports a collision through
`ErrorReg.CollErr` and `CollReg`:

- Check `CollPosNotValid` before using `CollPos`.
- `CollPos = 0` denotes collision at bit 32.
- Preserve the unambiguous received UID prefix.
- Choose a value for the collided bit, update NVB and framing alignment, and
  repeat anticollision with the now-known prefix.
- To enumerate all cards, retain the alternative branch and revisit it after
  selecting or halting the first card.

Anticollision requires bit-granular transmit and receive handling. Use
`BitFramingReg.TxLastBits`, `BitFramingReg.RxAlign`, `ControlReg.RxLastBits`, and
`CollReg`; a byte-only abstraction cannot implement the complete tree.

## SAK and Protocol Branching

SAK must be CRC-validated before use. The cascade bit only states whether UID
selection continues. Other SAK bits describe protocol capabilities, but fixed
SAK values are not reliable product identities.

After final selection:

- A PICC advertising ISO-DEP support can receive `RATS` and return ATS.
- MIFARE Classic uses its proprietary authentication and encrypted command
  flow.
- MIFARE Ultralight and many NTAG products use Type A-compatible memory
  commands without Crypto1, subject to each product's command set.

The driver should expose protocol evidence rather than silently guessing a
product solely from ATQA and SAK.

## Halt

The HLTA frame is:

```text
0x50 0x00 CRC_A_L CRC_A_H
```

A correctly selected PICC does not send a normal response. Therefore, timeout
after HLTA is expected behavior, not by itself an error. A response indicates a
protocol exception or collision scenario that must be evaluated separately.

## ISO-DEP Entry

`RATS` has command byte `0xE0`, followed by a parameter containing FSDI in the
high nibble and CID in the low nibble, then CRC_A. A successful response is ATS.
ATS parsing, frame-size negotiation, timing, optional CID/NAD handling, and
ISO-DEP block transport form a separate protocol layer and should not be
treated as part of basic UID selection.

## MIFARE Classic Notes

The MFRC522 provides a hardware `MFAuthent` command for MIFARE Mini, 1K, and 4K.
Before starting it, write exactly 12 FIFO bytes:

```text
auth command, block address, key[0..5], UID[0..3]
```

- `0x60` authenticates with Key A.
- `0x61` authenticates with Key B.
- On success, `Status2Reg.MFCrypto1On` becomes set.
- Authentication uses the four UID bytes required by the card protocol. For a
  seven-byte UID, this is the final four UID bytes, not the first four.
- A missing response does not necessarily terminate `MFAuthent`; configure the
  timer and treat `TimerIRq` as a termination condition.
- Clear `MFCrypto1On` before selecting or communicating with another card.

Common post-authentication commands include `READ (0x30)` and `WRITE (0xA0)`,
but exact exchange rules, access bits, value operations, and trailer handling
belong to the MIFARE Classic product specification.

Crypto1 and default/reused keys do not provide modern security. UID reading is
identification, not authentication, and UIDs can be copied or emulated. Do not
base an access-control security boundary only on UID or MIFARE Classic Crypto1.

## Ultralight and NTAG Notes

Many Ultralight and NTAG products support `READ (0x30)` with a page address and
return 16 bytes spanning four pages. A four-byte `WRITE (0xA2)` is common, but
supported commands, protected areas, counters, password features, and memory
limits vary by exact product.

Never derive writable bounds from the MFRC522. Detect or configure the product
type and enforce the card datasheet's memory map. Manufacturer pages, lock
bits, one-time-programmable fields, configuration pages, and credentials can be
irreversible or security-sensitive.

## ACK and NAK Frames

Several MIFARE commands return a four-bit ACK or NAK rather than a complete
byte. For the common positive ACK, the nibble is `0xA`. Validation must include:

- FIFO byte count.
- `ControlReg.RxLastBits == 4`.
- Low nibble value.
- Applicable error flags.

Do not accept a byte whose low nibble happens to be `0xA` if the received bit
length is wrong.

## Timeouts and Error Classification

Every RF operation needs a bounded timeout. Distinguish at least:

| Result | Meaning |
| --- | --- |
| Success | Expected length/framing and all integrity checks pass |
| No card / timeout | Timer expired before an expected response |
| Collision | `CollErr` with usable or unusable collision position |
| CRC error | Hardware or software CRC validation failed |
| Parity/protocol error | `ParityErr` or `ProtocolErr` set |
| FIFO error | Overflow, impossible length, or invalid host access |
| Unexpected response | Valid transport but wrong length, bits, ACK, or state |

A timeout can be valid for HLTA but an error for SELECT. Error interpretation
must therefore include the command context.

## Initial Protocol Milestone

The smallest useful, standards-aware milestone is:

1. Enable the RF field at 106 kBd.
2. Send REQA and validate ATQA.
3. Select one PICC through all necessary cascade levels.
4. Validate BCC and SAK CRC at each level.
5. Return the complete 4-, 7-, or 10-byte UID and final SAK.
6. Send HLTA and classify its expected timeout correctly.

MIFARE authentication, memory access, multi-card enumeration, ISO-DEP, and
higher bit rates should remain later, explicit layers.

## References

- NXP, *MFRC522 Standard performance MIFARE and NTAG frontend*, Rev. 3.9,
  sections 8 through 10.
- NXP, *MIFARE type identification procedure*, AN10833.
- NXP, *MIFARE product and handling of UIDs*, AN10927.
- ISO/IEC 14443-3, *Identification cards - Contactless integrated circuit cards
  - Proximity cards - Part 3: Initialization and anticollision*.
- The data sheet for the exact MIFARE or NTAG product being accessed.
