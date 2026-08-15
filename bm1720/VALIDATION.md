# BM1720 driver validation against shipped firmware

The driver in this directory was validated against the two firmware images
that actually drive the BM1720:

| Image | Binary | Arch | Transport | Relevance |
| ----- | ------ | ---- | --------- | --------- |
| `AntRouterR3LTCSIADASH20180815.bin` | `usr/bin/cgminer` (squashfs) | MIPS32 BE | UART | `--sia` option selects chiptype **1720** |
| `AntminerA32018111311360M.tar.gz` | `usr/bin/cgminer` (initramfs) | ARM LE | FPGA bridge | A3 hash boards are BM1720 |

All addresses below are virtual addresses in the respective binary. The R3
binary loads at 0x400000 (second segment: file offset 0x109350 → vaddr
0x519350); the A3 binary loads at 0x8000.

## What checked out

- **PLL table: byte-exact.** All 99 entries match the R3 firmware table.
  The binary holds two identical copies (file offsets 0x10b5ac and 0x10bbdc);
  `get_plldata()` (dispatch at 0x44cadc..0x44cb04, comparing the literals
  1387/1485/1760/1720) selects one copy each for chiptypes 1720 and 1760.
- **Divider relations.** Every entry satisfies
  `freq = 25 * fbdiv / (refdiv * postdiv1 * postdiv2)` with the label the
  truncated MHz value, and the `fildiv1`/`fildiv2`/`vilpll` packings hold for
  all 99 entries. Across the whole table `refdiv == 2`, `postdiv1 ∈ {2,3,4}`,
  `postdiv2 == 1`.
- **Chiptype 1720.** `--sia` maps to the literal 0x6b8 = 1720
  (R3 0x455810/0x455814), and the option table ties `--sia` to the flag
  byte the sia code paths test (opt entry at file offset 0x109dcc → flag
  0x51c904).
- **Frame lengths.** 5-byte short frames and 9-byte register frames, CRC
  byte included, confirmed in both firmwares' builders.
- **Registers.** PLL parameter = 0x0c and ticket mask = 0x14 confirmed in
  both firmwares.
- **Baud claim.** "must be 115200 or 57600" string present in the R3 option
  parser.
- **CRC16.** Standard CCITT-FALSE (poly 0x1021, init 0xffff); driver
  implementation matches the check vector.

## Mistakes found in the submitted driver (all fixed here)

### 1. CRC5 result bit-reversed — every TX frame had a bad CRC

Both firmwares implement the same 5-bit LFSR (poly x^5+x^2+1, init all
ones) and pack the result as `state[4]<<4 | state[3]<<3 | state[2]<<2 |
state[1]<<1 | state[0]` (R3 0x44ef84, epilogue 0x44f028; A3 0x592f8,
epilogue 0x59464). The submitted driver used the same recurrence but packed
`state[0]` into bit 4 — a bit-reversed CRC. Measured against the firmware
algorithm, 7496 of 10000 random 8-byte frames mismatched (the rest are
palindromic values). Example: chain-inactive `53 05 00 00` needs CRC 0x03;
the submitted code produced 0x18. The chip would silently drop essentially
every command.

### 2. Wrong command opcodes — the BM1720 does not use the BM1387/BM1485 map

The submitted header borrowed the BM1387/BM1485 VIL opcode assignments
(SET_ADDRESS=1, SET_PLL=2, GET_STATUS=4, CHAIN_INACTIVE=5, SET_CONFIG=8).
Those opcodes do exist in the R3 binary — but only on its BM1485/scrypt
code paths ("vil Set addr" 0x41, "vil Set ChainInactive" 0x55, "vil Set
pll" 0x58). The BM1720 (`--sia`) paths, corroborated independently by the
A3 binary, use a different map:

| Command | BM1720 frame | Evidence |
| ------- | ------------ | -------- |
| SET_ADDRESS (cmd 0) | `40 05 <addr> 00 <crc>` | R3 0x456750, A3 0x610d8 |
| WRITE_REGISTER (cmd 1) | `51 09 00 <reg> <val32> <crc>` broadcast, `41 ...` unicast | R3 0x454924 (PLL, reg 0x0c), A3 0x60854 |
| READ_REGISTER (cmd 2) | `42/52 05 <chip> <reg> <crc>` | A3 0x6078c |
| CHAIN_INACTIVE (cmd 3) | `53 05 00 00 <crc>` | R3 0x456560, A3 0x6104c |

Consequences of the submitted mapping on a real BM1720: `set_address`
sent 0x41 = unicast **register write**; `set_frequency` sent 0x52 =
broadcast **register read**; `chain_inactive` (0x55), `set_ticket_mask`
(0x58) and `read_register` (0x44) sent opcodes with no confirmed meaning
on this chip. PLL writes now go through WRITE_REGISTER reg 0x0c (which is
also exactly what the A3's `set_frequency` at 0x61708 does), ticket mask
through WRITE_REGISTER reg 0x14 (A3 0x6189c).

### 3. Response header reversed, response CRC ignored

The R3 nonce parser (0x44f53c) requires responses to start `0x55 0xAA`
(byte 0 checked at 0x44f548, byte 1 at 0x44f620); the submitted driver
required `0xAA 0x55` and would have rejected every genuine response. In
sia/dash mode responses are 9 bytes (UART read length set at 0x44df74):
the 2-byte preamble plus a 7-byte body whose last byte carries a CRC5 over
the body's first 51 bits in its low 5 bits (validator at 0x44f078:
`CRC5(buf+2, 0x33)` compared to `buf[8] & 0x1f`) and a flag in bit 7 that
makes the firmware route the response without a CRC check. The submitted
driver validated no CRC and treated byte 8 (the flag/CRC byte) as a
"work id". `bm1720_parse_nonce()` now checks header and CRC and exposes
the flag bits.

### 4. Missing 0x55 0xAA TX preamble on UART

Every BM1720 command the R3 firmware sends is prefixed with `0x55 0xAA` on
the UART (set_pll: 11 bytes sent at 0x454a14; chain-inactive: 7 bytes at
0x456634; set-addr: 7 bytes). The A3's FPGA bridge takes bare frames. The
driver now has a `uart_preamble` flag on the chain; R3-style UART users
must set it.

### 5. Provenance/comment errors

- The table header claimed the table lives at "vaddr 0x50b5ac". That value
  is file offset 0x10b5ac plus the 0x400000 base, but the offset is beyond
  the first LOAD segment; the real vaddr is 0x51b5ac. Comment corrected.
- `compute_pll()` claimed the shipped table uses "postdiv1 2..7, postdiv2 1
  or 2". It uses postdiv1 2..4 and postdiv2 == 1 only. The search space is
  now restricted to the silicon-proven envelope, ties keep the lowest VCO,
  and requests more than ~2% away from any reachable frequency are refused
  instead of silently programming something far off (the submitted code
  answered a 10 MHz request with a 28.57 MHz configuration).
- The claim that the BM1720 "shares" the BM1387/BM1485 VIL layout is
  disproven by both binaries and has been removed.

### 6. Smaller hardening fixes

- `bm1720_build_vil()` forces the chip-address byte to 0 on broadcast
  frames, matching both firmwares' builders.
- `bm1720_enumerate()` defaults `addr_interval` to `256 / chip_count` —
  the exact computation the R3 firmware performs (`0x100 / chain_length`
  at 0x4563ec) — instead of a hardcoded 4, and rejects configurations
  whose addresses would wrap past 255.

## Test suite

`bm1720_selftest.c` pins the driver to the firmware-derived byte sequences
(exact frames incl. CRCs, preamble behaviour, response parsing, the full
99-entry table envelope, CRC16 vector):

```
cc -Wall -Wextra -o bm1720_selftest bm1720_selftest.c bm1720.c
./bm1720_selftest
all tests passed
```

Additionally, the fixed `bm1720_crc5()` was fuzz-compared against a
reimplementation of the firmware's CRC5 on 10000 random 8-byte frames:
0 mismatches.

## Still unverified against silicon

- Register offsets 0x00 (chip addr), 0x1c (misc control), 0x20 (i2c) are
  inferred from the BM1387/BM1485 map. The A3 also writes registers 0x28
  and 0x2c broadcast during bring-up (0x61378/0x613bc); purpose unknown.
- The meaning of response bytes 6 and 7 inside *nonce* returns (they are
  chip/register address in register-read replies).
- The 25 MHz reference clock is inferred from the divider arithmetic.
- Work submission framing and the nonce→chip attribution are not part of
  this driver yet.
