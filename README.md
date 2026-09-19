# mfrc522-avr

## Overview

`mfrc522-avr` is a register-level C library for controlling an NXP MFRC522
contactless reader from an ATmega328P. The initial target uses a 16 MHz system
clock and the microcontroller's hardware SPI peripheral. The project does not
use Arduino, an MCU framework, or an existing MFRC522 library.

The library is being developed in phases. The current phase documents the
hardware interface, the MFRC522 register model, and the ISO/IEC 14443 A protocol
flow. No driver implementation is included yet.

## Technologies Used

- C11
- AVR 8-bit architecture and ATmega328P peripherals
- `avr-gcc`
- SPI, with the ATmega328P as controller and the MFRC522 as peripheral
- ISO/IEC 14443 A at 13.56 MHz

## How It Works

The ATmega328P exchanges command, configuration, status, and FIFO data with the
MFRC522 over SPI. The MFRC522 then handles the 13.56 MHz analog front end,
framing, parity, CRC support, and selected MIFARE operations. The host remains
responsible for sequencing register operations and the card activation,
anti-collision, selection, and application-level protocol.

The relevant design notes are:

- [SPI notes](docs/spi-notes.md)
- [MFRC522 register notes](docs/mfrc522-register.md)
- [Protocol notes](docs/protocol-notes.md)
- [ATmega328P notes](docs/atmega328p.md)

## Hardware Warning

The MFRC522 is a 3.3 V-class device and its inputs are not 5 V tolerant. A
16 MHz ATmega328P is outside its guaranteed operating area at 3.3 V, while a
common 16 MHz setup powers the MCU at 5 V. Such a design must provide
appropriate logic-level translation. Do not assume that an MFRC522 breakout
board includes level shifters; verify its schematic.

## AI Usage

OpenCode with GPT Sol has been used so far for datasheet cross-checking. The
technical decisions and resulting documentation remain subject to review
against the cited manufacturer documentation.
