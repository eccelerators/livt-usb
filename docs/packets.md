# USB packet layer

`Livt.Usb.Packets` implements individual low/full-speed packets above `IUsb2Phy`.
It does not enumerate devices, retry transactions, choose data toggles, or decide
when a received packet should be acknowledged. Those policies belong to
`Livt.Usb.Host`; see [host operation](host.md).

## Components

- `PacketPid`: supported PID values and PID/complement checks.
- `TokenCodec`: seven-bit address/four-bit endpoint packing and eleven-bit fields
  followed by CRC5. SOF uses the same field width for its frame number.
- `PacketReceiver`: PID/length/CRC validation and one retained payload slot.
- `PacketTransmitter`: staged payload, token and handshake construction.
- `PacketIntegrity`: USB CRC conventions and final packet-shape checks.
- `PacketError`: physical, PID, unsupported, length, checksum, overflow and occupied
  errors. Packet acceptance is not host-transfer success.

The CRC algorithm lives in `Livt.Utils.Hashing.Crc`. USB selects polynomial 0x05
for CRC5 and 0x8005 for CRC16, reflected processing, all-ones initial state and
all-ones final XOR. Tokens protect exactly eleven bits, excluding PID. Data CRC
covers payload only; the receiver checks the raw 0xB001 residue after both CRC
bytes. Handshakes carry no CRC. CRC16 is serialized low byte first.

The supported PID set is IN, OUT, SETUP, SOF, DATA0, DATA1, ACK, NAK and STALL.
SOF is rejected at low speed. Other PIDs, including high-speed and hub extensions,
are explicitly unsupported. Low-speed payloads are limited to eight bytes. The
packet buffer defaults to 64 bytes; a capacity of 1..1023 can be selected, but
larger capacity does not implement isochronous or high-speed host services.

## Clock and storage contracts

Both components share the PHY clock/reset. Drive explicit input signal fields
from a combinational binding process to preserve cycle timing and avoid implicit
storage when borrowing whole components.
Assert `cancel` alongside any PHY reset/disconnect/cancellation. Change speed
only while idle or with cancellation asserted. Context reset cancels work.

Each component owns one RAM instance, selected by generics:

```livt
receiver: PacketReceiver
smallReceiver: PacketReceiver<8, MemoryStyle.Distributed>
transmitter: PacketTransmitter<64, MemoryStyle.Block>
```

The transmitter has the same parameters. `B` must implement
`ISynchronousRam<byte, uint, RamGeometry<CAPACITY>>` with a default constructor.
Reads respond after one edge and writes commit on the accepting edge. The default
requests BRAM inference. A register-array backend must preserve the same
synchronous request/response contract. The scheduled
`PacketBuffer` and `IRam` APIs cannot substitute directly into this wire datapath.

Payload cells are not cleared by the default backend on reset. Validity, length
and publication metadata prevent reads of unwritten bytes. Only one owner may
operate each component's input signals.

## Receive

Capture every `active && valid` edge. A two-byte tail retains the CRC bytes, so
only payload bytes enter RAM. Publication occurs after active falls, and only
when PID, length, CRC and PHY status all pass.

- `completed` pulses for one edge for every received packet, valid or invalid.
- `accepted` pulses with `completed` only for a successfully published packet.
- Completion/acceptance events and all published metadata share one register
  boundary, so the packet is coherent on the first available edge.
- `error` retains the most recent completion result until completion or cancel.
- `available`, `pid`, `length`, `token` retain the published packet until release.
- `readRequest && readReady` accepts an in-range payload read. `readValid` and
  `readData` respond after that edge; consecutive reads are supported.
- `release` invalidates the held payload. Do not overlap release with an unfinished
  read. Release on the start edge permits reception into the now-empty slot.
- A packet arriving while occupied produces an `Occupied` error without changing
  the held metadata or bytes. The caller must drain/release before the next wanted
  packet. There is no receive backpressure on the wire.
- `cancel` invalidates publication and suppresses completion; it is not a successful
  empty packet. Physical errors invalidate the entire packet.

Unpublished bytes are never exposed. A CRC-valid duplicate is still just a valid
packet: the transaction layer decides whether to acknowledge/discard its payload.

## Transmit

- `loadValid && loadReady` appends one byte to the initialized staging prefix.
  Hold load data while awaiting acceptance. Full staging refuses further bytes.
- `clear` discards staging while idle and prevents command/load acceptance on that
  edge. It has no effect on an active transmission; use shared cancellation.
- `request && ready` accepts a command. Unsupported PID/speed/length combinations
  pulse `rejected` one clock after the accepting edge, without transmitting or
  consuming staging.
- Token commands use `token`; DATA commands use the entire staged prefix, including
  zero bytes; handshakes carry PID only. Request takes precedence over loading.
- During a packet, staging and command inputs cannot mutate the active request.
- `busy` covers preparation, wire transmission and EOP. `completed` pulses only
  after PHY busy falls. Successfully sent data consumes staging; token/handshake
  commands preserve it. Transmission completion does not mean device ACK.
- RAM prefetch occurs before a DATA PID and ahead of each payload byte. The output
  stream remains valid through the final CRC byte. This adapter requires IUsb2Phy
  byte pacing (at least 64 clock cycles between accepted bytes); it is not a
  general-purpose one-byte-per-clock stream producer.

## Integration notes

Wire child inputs through a combinational process and assign packet lengths for
each transaction. Publish TX byte and valid together from one owner register
boundary.

Protocol reference: USB 2.0 specification sections 8.3 and 8.4, with turnaround
constraints from chapter 7: https://www.usb.org/document-library/usb-20-specification

Host transactions, retries and enumeration use this layer through
`TransactionEngine` and `HostController`.

## Optional transmit pipeline

The final `PacketTransmitter` generic `PIPELINE_OUTPUT: bool = false`, added in 0.0.4,
selects a registered phase-to-byte boundary. `false` retains direct byte publication;
`true` publishes the byte and valid together one edge after phase advancement.
Both require the PHY's existing minimum 64 clocks between byte acceptances and
retain the synchronous packet-memory contract. `TransactionEngine` selects this
option through its own `PIPELINE_LINK` generic, which defaults to `true` and also
registers PHY feedback. See [link pipeline](host.md#link-pipeline).
