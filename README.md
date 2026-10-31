# CAPL Script Pack — CAN / CANoe Test Automation

A collection of production-oriented CAPL scripts for automotive ECU and ADAS bench testing:
message simulation, signal monitoring, table-driven test sequences, raw ISO-TP / UDS
diagnostics, and panel-driven test control.

All code is written from scratch. No customer data, DBC files or proprietary project
material is included.

---

## Contents

| File | Purpose | External prerequisites |
|---|---|---|
| `can_message_simulator.can` | Periodic + event-driven CAN frame simulation, reacts to inbound requests | None (classic CAN, raw byte layout) |
| `signal_monitor.can` | Decodes a status frame, tracks min/max/mean, debounced threshold alarms, alive-counter loss detection | None |
| `automated_test_sequence.can` | Timer-driven, 6-step test sequence with pass/fail reporting | None |
| `uds_iso_tp_client.can` | Raw ISO-TP (single + multi-frame RX with flow control) UDS client — no ODX/CDD required | None |
| `panel_controller.can` | Panel / system-variable driven test orchestration | System variables listed in the file header |

Everything except `panel_controller.can` runs against an empty CANoe configuration with
a single CAN channel — no DBC, no diagnostic database, no hardware required if a virtual
channel is used.

---

## Why the scripts look like this

**Raw byte layout instead of DBC signals.**
So the code compiles and runs in any configuration. Signal-based access (`msg.EngineSpeed`)
is shown in comments where it would normally be used — it requires the DBC to be loaded,
which makes a portfolio project non-runnable for anyone who clones it.

**No `testWaitFor...` calls.**
The CAPL Test Feature Set requires an additional license. The sequence runner is built on
`msTimer` + an explicit state machine, so it works on a plain CANoe installation.

**All timing through timers, never `sleep`.**
CAPL has no blocking sleep and should not emulate one. Every wait is a timer with a
defined timeout, which is also what makes failures diagnosable.

**Debounced alarms.**
A threshold breach raises an alarm only after N consecutive samples. Single-sample spikes
are noise; treating them as failures is how test suites lose credibility.

**Defensive parsing.**
Every inbound frame is length-checked before decoding. A malformed or truncated frame must
produce a diagnostic message, not a crash or a silently wrong value.

**ISO-TP implemented explicitly.**
`uds_iso_tp_client.can` speaks ISO-TP directly (SF / FF / CF / FC) instead of relying on the
CANoe diagnostic API. It needs no ODX/CDD, and it demonstrates what actually happens on the
bus — which is what people mean when they say they "know UDS".

---

## How to run

1. Open CANoe and create a configuration with **one CAN channel** (a virtual channel is fine).
2. Add a **Simulation Setup** node.
3. Attach the `.can` file to the node (right-click → *Configuration* → *.can file*).
4. Start the measurement.
5. Use the keyboard to drive the scripts:

| Key | Script | Action |
|---|---|---|
| `a` / `s` | simulator | enable / disable frame output |
| `r` | simulator | reset state |
| `t` | sequence | start the 6-step test sequence |
| `x` | sequence | abort the sequence |
| `u` | UDS client | send TesterPresent (0x3E) |
| `v` | UDS client | read DID 0xF190 (0x22) |
| `w` | UDS client | switch to extended session (0x10 0x03) |
| `i` | UDS client | inject an invalid length to test error handling |

Output goes to the **Write** window. For persistent logging, attach a Logging block and
record the bus to BLF/ASC — that is the correct place for logs, not inside a CAPL script.

> Press keys while the Write window has focus.

**No CANoe licence?** Vector publishes a free **CANoe Demo** with reduced functionality,
which is sufficient for running these scripts. Students and university programmes can often
obtain an extended educational licence. Verify current terms on the Vector website.

---

## Frame layout used by the simulator / monitor / sequence

```
0x100  Status    DLC 8
       byte 0-1   Engine speed   factor 0.25 rpm/bit   (unsigned, little endian)
       byte 2-3   Vehicle speed  factor 0.01 km/h/bit  (unsigned, little endian)
       byte 4     Gear
       byte 5     Output enable (1 = active)
       byte 6     Rolling cycle counter
       byte 7     Cycle counter low byte (diagnostic)

0x200  Command   DLC 8
       byte 0-1   Target speed   factor 0.01 km/h/bit

0x300  Alive     DLC 8
       byte 0     Rolling alive counter
       byte 1     Cycle counter low byte
```

---

## ISO-TP / UDS behaviour in `uds_iso_tp_client.can`

- **TX**: single frame only (payload ≤ 7 bytes). Covers 0x10, 0x11, 0x14, 0x3E, 0x22 with
  2-byte DIDs, and short 0x2E writes. Multi-frame transmission is documented as an extension
  point rather than half-implemented.
- **RX**: single frame and multi-frame (FF + consecutive frames), with flow control sent back
  to the tester. Sequence numbers are validated; out-of-order frames abort the transfer.
- **P2 / P2\***: P2 = 50 ms, P2\* = 5000 ms. Negative response `0x78` (response pending)
  restarts the P2\* timer instead of failing.
- **ISO-TP padding**: `0xCC` (any non-PCI value is acceptable; the choice is documented).

---

## Extension points

- Add CAN FD support (`.FDF`, `.BRS`, DLC > 8) — the simulator is structured to allow it
- Replace raw bytes with DBC signal access once a database is available
- Add a CRC-8 (SAE J1850) calculation over the status frame
- Feed results into `testWaitForSignalInRange()` if a Test Feature Set licence is available
- Emit results as system variables so a Panel can display live pass/fail state

---

## Author

Automotive test engineer — CAPL / CANoe / HIL / ADAS validation.
Available for contract work on test automation, bench setups and diagnostic tooling.

- Email: you@example.com
- GitHub: [@yourname](https://github.com/yourname)
