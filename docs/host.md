# Direct USB host and boot keyboard

`Livt.Usb.Host.HostController` implements `IHostController` for one directly
attached low/full-speed device. It owns `Usb2Phy`, context-derived root-port timing,
transaction engine and retained transfer storage. The board owns electrical pads,
VBUS, clock generation and reset conditioning. No soft CPU or external USB PHY is
required by this composition.

## Layers and ownership

- `RootPort`: 100 ms attach debounce, 12 ms reset, 10 ms recovery, 1 ms frame time
  and disconnect detection outside host drive.
- `TransactionEngine`: packet/frame arbitration, SETUP/OUT/IN exchanges, immediate
  IN ACK, duplicate recognition, NAK/STALL and bounded response timeout. Neither
  scheduled calls nor UART latency pace its response path.
- `HostController`: one scheduled submission owner, controller-owned setup/output
  staging, packetization, control setup/data/status stages, retained completion,
  deadline-bounded retries and per-endpoint/direction data toggles.
- `ControlTransfer`: standard/class request construction without class policy.
- `DeviceEnumerator`: descriptor discovery, EP0 sizing, address one, first
  configuration selection and bounded endpoint/interface binding.
- `BootKeyboardHost`: alternate-zero HID boot interface selection, SET_PROTOCOL(0),
  SET_IDLE(0), interrupt polling and queued usage transitions.

All components share the same clock/reset context. An application must serialize
its scheduled controller calls. `BootKeyboardHost` owns that client while active;
other clients must not submit concurrent requests. Packet and frame logic has
independent hardware ownership. Completion data stays valid until `TryRelease`.

## Transfer behavior

Control and interrupt are supported at low/full speed; bulk is full-speed only.
Isochronous requests and invalid endpoint geometry are rejected before reserving
storage. Control data starts with DATA1, status uses DATA1, and SETUP uses DATA0.
Endpoint toggles advance when the packet is acknowledged on the wire, even if
retaining an IN packet subsequently overflows or is cancelled. Duplicate IN packets
are ACKed without publishing duplicate data. OUT retries reload the same immutable
payload and retain their toggle until ACK. Successful SET_CONFIGURATION clears
all endpoint toggles. After successful CLEAR_FEATURE(ENDPOINT_HALT), the owner can
call `ClearToggle` while idle; `ResetPort` requests complete port re-enumeration.

The transaction engine retains received data until the next request. It does not
wait for the scheduled client before ACK. A 200 us guard prevents a new exchange
from starting too near a frame boundary; SOF or low-speed keep-alive takes priority
at the next idle opportunity. The response window including a low-speed payload
is bounded to 128 us. NAK, missing/corrupt response and duplicate retries are
bounded by the caller's millisecond deadline. Interrupt attempts, including subsequent submissions, obey bInterval;
other retries wait for a new frame.

Transfer capacity defaults to 512 bytes, with separate block-RAM input/output
stores and eight setup registers. Counts use capacity-derived widths. The default
`SequentialRam` access path relies on the host's exclusive buffer ownership;
the event queue likewise serializes its RAM operations in its own arbiter. The `STYLE` and `IRam<byte>` backend parameters
allow alternative storage, including `ArrayRam`. Packet slots use synchronous
64-byte RAM; `TransactionEngine` exposes its synchronous backend parameter.
`HostController` forwards that choice through its `PACKET_STORAGE` generic,
and its default follows `STYLE`. See [storage selection](storage.md#selecting-nested-storage).
The 32 endpoint/direction polling timestamps use a separate 16-bit BRAM table,
selectable through `P: IRam<logic[16]>`. A validity mask prevents reading unwritten
RAM after reset. Event queues also expose capacity, style and backend parameters.
Configuration parsing retains endpoint metadata in a 32-byte BRAM table; its
`B: IRam<byte>` parameter permits an `ArrayRam<byte, 32>` or another compatible
backend. Small duplicate-detection lists and current descriptor bytes use registers.

The engine command interface uses toggle mailboxes: change `clearRequest`,
`loadRequest` or `request` once, hold associated fields stable, and wait until the
matching `clearAccepted`, `loadAccepted` or `requestAccepted` equals that toggle.
Do not interpret the `ready` snapshots as scheduled-call handshakes. Cancellation
synchronizes pending mailboxes and discards uncommitted packet state. A completed
wire ACK remains observable so cancellation cannot roll back endpoint toggles.
The returned IN length counts complete packets retained in transfer storage;
partially copied or overflowing packets are not exposed to the client.

Root-port timing parameters are durations in microseconds, with the production
defaults above. `HostController` exposes `PORT_DEBOUNCE_US`, `PORT_RESET_US` and
`PORT_RECOVERY_US`; `RootPort` exposes `DEBOUNCE_US`, `RESET_US` and `RECOVERY_US`.
Retain the defaults in production. See [clock configuration](clocking.md) for
clock constraints, timing units and counter sizing.

## Link pipeline

Since 0.0.4, the `PIPELINE_LINK: bool = true` parameter is appended to both
`HostController` and `TransactionEngine`. Existing generic arguments keep their
positions. `HostController` forwards the choice to its engine.

| Component | Generic | Default |
|---|---|---|
| `HostController` | `PIPELINE_LINK` | `true` |
| `TransactionEngine` | `PIPELINE_LINK` | `true` |
| Standalone `PacketTransmitter` | `PIPELINE_OUTPUT` | `false` |

For a directly composed transaction engine, keep the same packet RAM and choose
only the pipeline behavior:

```livt
using Livt.IO
using Livt.Usb.Host

// Fields in the owning component; construct the matching types in its constructor.
pipelined: TransactionEngine
bypass: TransactionEngine<MemoryStyle.Block,
	SynchronousRam<byte, uint, 64, MemoryStyle.Block>, false>
```

Both instances own their packet stores. A design normally selects one of these
configurations; no runtime selector is created. When using `HostController`, its
final generic selects the same behavior after the storage and root-port parameters.

With the default enabled, the engine samples speed, cancellation, RX activity,
byte validity, data and physical errors together in an ingress register stage.
It also registers release of the retained RX slot and uses a separate state for
requesting an ACK after validating the received packet. This adds one receive
sampling edge and one ACK-decision edge, breaking combinational paths between
PHY observation, packet validation and transmit preparation. Frame admission is
also registered with one extra clock of reserve: `frameRemaining` must count down
by one per edge and wrap at `frameTick`, as `RootPort` does. This avoids combining
the frame-time subtraction with packet launch in one combinational path.

Transmit feedback registers an actual byte acceptance (`valid && ready`) and PHY
busy state. Registering ready alone would duplicate an idle-ready level. The
engine also enables the packet transmitter's output pipeline, which publishes
bytes one edge after its phase advances. The PHY's minimum 64 clocks between
accepted bytes leaves time for both stages without interrupting a packet.

Set `PIPELINE_LINK` to `false` to bypass these stages when lower latency is more
useful than the timing separation. The unused ingress registers can then be
removed by synthesis. Both settings retain the same packet, toggle, cancellation
and retained-completion contracts. Neither setting changes the memory provider
or permits an arbitrary-latency packet RAM. Qualify timing in the target design.

## Enumeration limits

The standard parser bounds configurations to 512 bytes, four interfaces and three
nonzero endpoints in addition to EP0. It skips unknown descriptor types by their
validated length, retains alternate setting zero, and rejects malformed/truncated
streams, duplicate endpoint addresses, inconsistent endpoint/interface counts,
unsupported packet geometry and hubs. Only the first configuration is selected.
Other class interfaces may coexist within these bounds; the keyboard driver binds
the first interrupt-IN endpoint belonging to class 3/subclass 1/protocol 1. Endpoint
address, packet size, interval, configuration value and interface number come from
descriptors, rather than fixed keyboard assumptions.

A discovery or class initialization failure waits for disconnect/reset. High speed,
hubs, isochronous endpoints, alternate-setting selection and arbitrary report
protocol parsing are not supported. Full-speed bulk transport also supports the
[read-only mass-storage driver](mass-storage.md).

## Keyboard reports and events

The boot report is exactly eight bytes: modifiers, reserved byte, six usages.
`BootKeyboardReportParser` validates before replacing held state. Error/rollover
usages 1–3 preserve the previous held keys. Repeated reports and duplicate usages
do not emit repeated presses. Release transitions precede presses; modifier changes
use usages E0–E7. The application owns keyboard layout, Caps Lock, LEDs and repeat.
The two eight-byte report snapshots default to BRAM through
`BootKeyboardReportParser<B>`; `ArrayRam<byte, 16>` is a supported substitute.
Validity metadata prevents first use or reset from exposing unwritten/retained
report bytes. Modifier bytes are cached in registers for bit access.

Event bits 7..0 are usage, 15..8 current modifiers, and 16 is press/release. Bit 17
is the host's disconnect marker. The default 32-event queue uses block RAM and
supports independent scheduled producer/consumer calls. Full queues reject new
events and increment the host's dropped counter; they do not block USB polling.
Disconnect generates held-key releases and resets parser state even if the queue
is full. Consumers must treat an overflow counter change as loss of event history.

## Protocol references

Protocol references: [USB 2.0 specification package](https://www.usb.org/document-library/usb-20-specification)
and [HID 1.11](https://www.usb.org/sites/default/files/documents/hid1_11.pdf), including
its boot-keyboard report and class-request definitions.

## Keyboard binding observations

`BootKeyboardHost.identity` exposes the validated vendor/product IDs (VID in the
upper 16 bits). `binding` packs configuration value, endpoint-zero packet size,
interface number and interrupt endpoint number from low to high bytes. These
observations are valid after successful initialization (`GetStatus() == 4`).
The observations forward retained, validated discovery data through combinational
ports. They avoid extra scheduled return buffers and do not create a second
scheduled client for the host or descriptor storage.
`DeviceEnumerator.GetConfiguration()` returns zero until configuration succeeds.

`BootKeyboardBinding` isolates device-specific selection exceptions. Rapoo
`24AE:2010` and `24AE:2013` expose auxiliary and normal keyboard interfaces.
Interface 1 carries auxiliary
controls, while interface 2 carries normal keys. Its auxiliary interface is skipped,
then the normal descriptor checks select the keyboard endpoint. Other device IDs
retain the first eligible interface policy. This does not add report-protocol HID
or merge multiple keyboards. See the [maintainer's device report](https://github.com/tmk/tmk_keyboard/issues/764)
and [descriptor capture](https://gist.github.com/tmk/20e2a574dc45ce7789b93e15e9ca9eb6)
for `2013`, and the [receiver logs](https://forums.raspberrypi.com/viewtopic.php?t=240314)
for `2010`. The exception is limited to these product IDs; standard descriptor
checks still determine endpoint number, packet size and polling interval.
