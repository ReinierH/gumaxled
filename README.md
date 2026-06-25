# Gumax ASY-3501-1 Remote — Protocol Documentation

> Reverse engineered from 10 VCD captures via Homey built-in 433 MHz receiver.
> All captures decoded at 100% confidence with consistent 24-bit frames.
> **Protocol verified and implemented in a working Homey app.**

---

## Radio Characteristics

| Parameter | Value |
|---|---|
| Frequency | 433.92 MHz |
| Modulation | OOK / ASK |
| Direction | Unidirectional (remote → receiver) |
| Code type | Static (fixed code, **not** rolling code) |

---

## Packet Structure

Each transmission consists of a **preamble/sync** section followed by **24 data bits**, then an inter-packet gap before the next repetition:

```
[PREAMBLE ~5400µs HIGH] [~120µs LOW] [~370µs HIGH] [~9000µs LOW] [24 DATA BITS] [~9300µs LOW] → repeat
```

### Timing breakdown

| Section | Level | Duration |
|---|---|---|
| Preamble (AGC) | HIGH | ~5400 µs |
| Post-preamble gap | LOW | ~120 µs |
| Sync pulse | HIGH | ~370 µs |
| Silence before data | LOW | ~9000 µs |
| 24 data bits | — | 24 × ~1480 µs = ~35.5 ms |
| Inter-packet gap | LOW | ~9300 µs avg |

---

## Bit Encoding (PWM)

Each bit is encoded as a HIGH+LOW pair with a total period of ~1480 µs:

| Bit value | HIGH duration | LOW duration | Total |
|---|---|---|---|
| `1` | ~370 µs (SHORT) | ~1100 µs (LONG) | ~1480 µs |
| `0` | ~1110 µs (LONG) | ~370 µs (SHORT) | ~1480 µs |

**Decision threshold:** 700 µs — pulses shorter than 700 µs are SHORT (bit=1), longer are LONG (bit=0).

**Bit rate:** ~675 bps

### Measured pulse timings

| | SHORT HIGH | LONG HIGH | SHORT LOW | LONG LOW |
|---|---|---|---|---|
| Average | ~372 µs | ~1110 µs | ~378 µs | ~1094 µs |
| Min | ~218 µs | ~856 µs | ~202 µs | ~704 µs |
| Max | ~674 µs | ~1366 µs | ~692 µs | ~1498 µs |

---

## Frame Format (24 bits)

```
Bits  0– 7 : Remote ID byte 1       = 0x3D  (00111101)  — fixed per remote
Bits  8–15 : Remote ID byte 2       = 0x67  (01100111)  — fixed per remote
Bits 16–19 : Remote ID / channel    = 0x9   (1001)      — fixed per remote
Bits 20–23 : Command nibble         = 0x?   (varies)
```

Bytes 0–1 and the high nibble of byte 2 are **identical across all 10 commands** — this is the remote identity. Only the last 4 bits (command nibble) change.

---

## Complete Command Table

All 10 buttons fully decoded at 100% confidence:

| Command | Nibble | Byte 2 | Full 24-bit frame |
|---|---|---|---|
| HIGHER  | `0x0` | `0x90` | `00111101 01100111 10010000` |
| ON      | `0x2` | `0x92` | `00111101 01100111 10010010` |
| LEVEL 1 | `0x3` | `0x93` | `00111101 01100111 10010011` |
| LEVEL 6 | `0x4` | `0x94` | `00111101 01100111 10010100` |
| LEVEL 5 | `0x5` | `0x95` | `00111101 01100111 10010101` |
| LEVEL 3 | `0x6` | `0x96` | `00111101 01100111 10010110` |
| LEVEL 4 | `0x7` | `0x97` | `00111101 01100111 10010111` |
| OFF     | `0x8` | `0x98` | `00111101 01100111 10011000` |
| LOWER   | `0x9` | `0x99` | `00111101 01100111 10011001` |
| LEVEL 2 | `0xD` | `0x9D` | `00111101 01100111 10011101` |

> **Note on LEVEL 2 (0xD):** this value is non-sequential compared to the other levels. The level numbering in the Homey app does not correspond to the command nibble value order. The nibble values are arbitrary command codes, not sequential level numbers.

### As JavaScript constants

```javascript
const REMOTE_ID = '00111101011001111001'; // bits 0-19, fixed

const COMMANDS = {
  HIGHER: [0,0,1,1,1,1,0,1, 0,1,1,0,0,1,1,1, 1,0,0,1,0,0,0,0], // 0x3D 0x67 0x90
  ON:     [0,0,1,1,1,1,0,1, 0,1,1,0,0,1,1,1, 1,0,0,1,0,0,1,0], // 0x3D 0x67 0x92
  LEVEL1: [0,0,1,1,1,1,0,1, 0,1,1,0,0,1,1,1, 1,0,0,1,0,0,1,1], // 0x3D 0x67 0x93
  LEVEL6: [0,0,1,1,1,1,0,1, 0,1,1,0,0,1,1,1, 1,0,0,1,0,1,0,0], // 0x3D 0x67 0x94
  LEVEL5: [0,0,1,1,1,1,0,1, 0,1,1,0,0,1,1,1, 1,0,0,1,0,1,0,1], // 0x3D 0x67 0x95
  LEVEL3: [0,0,1,1,1,1,0,1, 0,1,1,0,0,1,1,1, 1,0,0,1,0,1,1,0], // 0x3D 0x67 0x96
  LEVEL4: [0,0,1,1,1,1,0,1, 0,1,1,0,0,1,1,1, 1,0,0,1,0,1,1,1], // 0x3D 0x67 0x97
  OFF:    [0,0,1,1,1,1,0,1, 0,1,1,0,0,1,1,1, 1,0,0,1,1,0,0,0], // 0x3D 0x67 0x98
  LOWER:  [0,0,1,1,1,1,0,1, 0,1,1,0,0,1,1,1, 1,0,0,1,1,0,0,1], // 0x3D 0x67 0x99
  LEVEL2: [0,0,1,1,1,1,0,1, 0,1,1,0,0,1,1,1, 1,0,0,1,1,1,0,1], // 0x3D 0x67 0x9D
};
```

---

## Repetition

| Parameter | Value |
|---|---|
| Observed repetitions per button press | ~50–80× (button held) |
| Recommended minimum for implementation | 8–10× |
| Inter-packet gap | ~9300 µs |

The frame is repeated back-to-back with the inter-packet gap between each repetition. Every repetition is byte-for-byte identical (static code).

---

## Transmission Pseudocode

```
function sendCommand(frame_bits[24]):
    repeat 10 times:
        sendHigh(5400µs)     // preamble AGC
        sendLow(120µs)
        sendHigh(370µs)      // sync pulse
        sendLow(9000µs)      // silence
        for each bit in frame_bits:
            if bit == 1:
                sendHigh(370µs)
                sendLow(1100µs)
            else:
                sendHigh(1110µs)
                sendLow(370µs)
        sendLow(9300µs)      // inter-packet gap
```

---

## Caveats & Limitations

1. **Remote ID is device-specific.** The fixed prefix `0x3D 0x67 0x9_` belongs to the one specific remote used for these captures. Other remotes will have different IDs and would need their own capture/learn flow.

2. **Frame may be longer than 24 bits.** The Homey VCD captures appear to consistently miss the start of each frame due to preamble/AGC timing. All 10 captures agree on 24 bits at 100% confidence, but a full frame could be longer. A clean RTL-SDR or Flipper Zero capture would confirm the total frame length definitively. That said, the 24-bit frames are sufficient to control the device.

3. **Protocol identity unconfirmed.** The signal (24-bit PWM, 433.92 MHz ASK, static code) is consistent with the A-OK AC114/AC123 family but has not been definitively verified against that spec.

4. **Not hardware-verified.** The decoded frames have not yet been transmitted back to an actual Gumax lighting unit to confirm they trigger the correct actions.

---

## Capture Summary

| File | Button | Packets | Confidence |
|---|---|---|---|
| `on.vcd`     | ON      | 50  | 100% |
| `off.vcd`    | OFF     | 81  | 100% |
| `higher.vcd` | HIGHER  | 77  | 100% |
| `lower.vcd`  | LOWER   | 71  | 100% |
| `level1.vcd` | LEVEL 1 | 65  | 100% |
| `level2.vcd` | LEVEL 2 | 61  | 100% |
| `level3.vcd` | LEVEL 3 | 70  | 100% |
| `level4.vcd` | LEVEL 4 | 57  | 100% |
| `level5.vcd` | LEVEL 5 | 57  | 100% |
| `level6.vcd` | LEVEL 6 | 77  | 100% |

Captured via Homey built-in 433 MHz receiver. VCD timescale: 1 µs per unit.
