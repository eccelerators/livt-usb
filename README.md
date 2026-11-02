# Livt.Usb

`Livt.Usb` provides reusable USB components for Livt FPGA applications. It combines
low/full-speed signaling, packet processing, host transfers and device discovery
with HID boot-keyboard and read-only mass-storage support. It is a separate base
library alongside `Livt.IO`.

See [CHANGELOG.md](CHANGELOG.md) for release notes.

Version 0.0.4 provides:

- `Livt.Usb.IO.Usb2Phy`: low-speed and full-speed signaling with context-derived timing.
- `Livt.Usb.Packets`: token, data and handshake processing with PID and CRC checks.
- `Livt.Usb.Host.HostController`: control, interrupt and full-speed bulk transfers
  for one directly attached device, with retained completion and bounded retries.
- `Livt.Usb.Host.DeviceEnumerator`: standard descriptor discovery and configuration.
- `Livt.Usb.Descriptors`: bounded device, interface and endpoint parsing.
- `Livt.Usb.Hid.BootKeyboardHost`: boot-keyboard initialization, polling and usage events.
- `Livt.Usb.MassStorage`: read-only SCSI Bulk-Only Transport and retained block reads.
- Configurable buffering with BRAM defaults and alternative storage providers.
- Configurable host and packet-transmit pipelines for timing and latency tradeoffs.
- `Livt.Usb.Testing.TestHostController`: a deterministic controller test double.

## 📦 Package

```toml
[dependencies]
"Livt.Usb" = "0.0.4"
```

`Livt.IO 1.7.0` supplies memory interfaces, storage implementations and the
transport-independent `IReadOnlyBlockDevice` contract. `Livt.Utils 1.1.0` supplies
generic CRC helpers. Applications using their types directly should also declare
those dependencies.

Both dependencies are pinned to published registry versions and resolved by
`livt sync`. No sibling library checkouts are required.

`Livt.Usb` has no UART or CLI dependency. Applications supply their own transport,
commands and presentation; platform integration supplies clocks, reset and I/O pads.

## 📚 Namespaces

| Namespace | Components | Purpose |
|---|---|---|
| `Livt.Usb` | `IHostController`, USB enums | Scheduled transfer lifecycle and capabilities |
| `Livt.Usb.IO` | `IUsb2Phy`, `Usb2Phy`, line/RX/TX components | Low/full-speed signaling and pad ownership |
| `Livt.Usb.Packets` | `PacketReceiver`, `PacketTransmitter`, `PacketPid`, `TokenCodec`, `PacketIntegrity`, `PacketBuffer` | Packet encoding, integrity and payload storage |
| `Livt.Usb.Host` | `HostController`, `RootPort`, `TransactionEngine`, `ControlTransfer`, `DeviceEnumerator`, `HostProfile`, `TransferStorage` | Host timing, transfers and discovery |
| `Livt.Usb.Descriptors` | `DeviceDescriptorParser`, `ConfigurationDescriptorParser` | Bounded descriptor parsing |
| `Livt.Usb.Hid` | `BootKeyboardHost`, `BootKeyboardReportParser`, `KeyboardEventQueue`, `IKeyboardSource`, `BootKeyboardBinding` | Boot-keyboard lifecycle and usage transitions |
| `Livt.Usb.MassStorage` | `MassStorageHost`, `BulkOnlyTransport`, `ReadOnlyScsiCommand` | Read-only SCSI commands, BOT recovery and retained block access |
| `Livt.Usb.Testing` | `TestHostController` | Manually driven controller for tests |

USB signaling and PHY adapters belong to `Livt.Usb.IO`; general-purpose I/O stays
in `Livt.IO`. See [architecture](docs/architecture.md) for component ownership and
extension boundaries.

## 🔌 API Overview

### Host transfers

`IHostController` provides setup/output staging, bounded submission, completion
queries, received-data access, cancellation and explicit release. Acceptance
reserves the request slot; the caller waits for completion and inspects its result
before using received data. Completion remains available until released.

`HostController` implements this contract with autonomous root-port and wire
service. One scheduled client serializes controller access. Control and interrupt
transfers support low/full speed; bulk transfers support full speed. Transfer
storage defaults to 512 bytes per input/output store.

`PIPELINE_LINK` defaults to `true`, registering receive signals, ACK decisions,
frame admission and transmit feedback, and enabling the packet output pipeline.
Set it to `false` for the direct path when lower latency is preferred. Both
settings preserve transfer ownership, completion and packet-memory contracts.
See [link pipeline](docs/host.md#link-pipeline) for configuration and timing.

`DeviceEnumerator` discovers device and configuration descriptors, assigns address
one and selects the first configuration. `ControlTransfer` constructs control
requests; the descriptor parsers retain bounded interface and endpoint metadata.
See [host contracts](docs/host-controller.md) and [host operation](docs/host.md).

### Boot keyboards

`BootKeyboardHost` borrows the controller's sole scheduled client and owns device
discovery, boot-protocol initialization and interrupt polling. `IKeyboardSource`
exposes queued HID usage transitions and status. Repeated reports do not repeat
presses; disconnect releases held state. Queue overflow is observable.

Applications own keyboard layout, key-to-text conversion, Caps Lock policy, LEDs
and repeat behavior. The library reports usages and modifiers without choosing a
terminal or character encoding. See [keyboard reports and events](docs/host.md#keyboard-reports-and-events).

### Read-only mass storage

`MassStorageHost` owns enumeration, interface selection and SCSI initialization
through one host controller. It implements `Livt.IO.IReadOnlyBlockDevice` for
one retained 512-byte block. Submit an LBA, wait for completion, read the bytes,
then release the block. Consumers can pause between reads without losing data;
failed commands and disconnects invalidate the payload.

The supported profile is a directly attached full-speed SCSI-transparent
Bulk-Only Transport device, LUN zero and READ CAPACITY(10). `BulkOnlyTransport`
validates command status and handles endpoint stalls and reset recovery.
Filesystems, partitions, filenames and application data processing remain outside
USB. See [mass storage](docs/mass-storage.md) for ownership, recovery and limits.

### PHY and packets

`IUsb2Phy` uses signal-level handshakes. `Usb2Phy` provides sampling, NRZI,
bit stuffing, SYNC/EOP, reset signaling and low-speed keep-alive.
Timing and counter widths follow the inherited clock context. The current sampled
PHY requires an exact multiple of 12 MHz with at least eight samples per
full-speed bit (96 MHz or higher). See [clock configuration](docs/clocking.md).
`PacketReceiver` and `PacketTransmitter` add PID/CRC processing, bounded payload
retention and packet construction. Their default packet stores are 64-byte
synchronous block-style RAM.

The integration owns electrical pads, VBUS, clock generation and reset conditioning.
See [PHY contracts](docs/phy.md) and [packet contracts](docs/packets.md) for timing,
cancellation, storage and ownership requirements.

### Scheduled packet storage

`PacketBuffer` maintains an initialized payload prefix. Append bytes, publish the
payload, read it, then clear it after all readers finish. Empty publication represents
a zero-length payload. `TryAppend`, `TryPublish` and `TryRead` report acceptance
explicitly; failed operations do not expose unwritten storage.

Calls may take multiple cycles and require serialized ownership. This staging API
is separate from the signal-level packet datapath. See [packet storage](docs/packet-buffer.md).

### Controller test double

`TestHostController` implements the scheduled interface without physical pins.
A fixture supplies attachment, received bytes, completion and frame advancement.
It supports deterministic lifecycle and application tests; it does not simulate
wire packets or enumerate devices. See [test-backend contracts](docs/host-controller.md#test-backend-and-resource-selection).

## ⚙️ Storage configuration

Resource-bearing components expose compile-time capacities and storage providers.
Defaults request block RAM; small control fields use registers. For example:

```livt
using Livt.IO
using Livt.Usb.Packets

// Fields in an owning component.
payload: PacketBuffer
logicPayload: PacketBuffer<64, MemoryStyle.Block, ArrayRam<byte, 64>>
distributedPayload: PacketBuffer<64, MemoryStyle.Distributed>
```

Construct the matching types in the owner's constructor:

```livt
payload = new PacketBuffer()
logicPayload = new PacketBuffer<64, MemoryStyle.Block, ArrayRam<byte, 64>>()
distributedPayload = new PacketBuffer<64, MemoryStyle.Distributed>()
```

`PacketBuffer` defaults to `Ram<byte, 64, MemoryStyle.Block>`. Its capacity counts
payload bytes, excluding PID and CRC. An explicit backend replaces the default;
`STYLE` does not change the behavior of an independently supplied `ArrayRam`.

Host transfer storage defaults to `SequentialRam`; packet engines use synchronous
signal-level memory. Match the required interface and timing when substituting
storage. Style is a synthesis hint; inspect target resource reports for physical
mapping. Endpoint-table providers are selectable through `DeviceEnumerator`,
`BootKeyboardHost` and `MassStorageHost`; `HostController` also forwards its packet
memory provider. Scheduled providers may take different numbers of cycles, while
packet providers must preserve their synchronous response timing. See
[storage configuration](docs/storage.md) for capacities and reset rules.

## 🧪 Build and Test

```sh
livt sync
livt validate
livt test --release -W all,error
livt build --release -W all,error
```

The package's [tests](tests/) cover transfer contracts, descriptor parsing,
keyboard transitions, queue behavior, mass-storage framing and recovery, retained
block reads, storage alternatives and root-port timing across clock contexts.
Transaction tests exercise both pipeline settings against the physical transmit
serializer, including packet bytes, CRCs, ACK handoff and cancellation.

## 🛠️ Development Notes

- Keep physical timing in signal-level engines and scheduled ownership explicit.
- Preserve acceptance, completion and reset semantics across storage providers.
- Keep layout, terminal commands, filesystems and platform wiring in their owning layers.
- Introduce namespaces when implementing their components; avoid placeholder APIs.

## 🚧 Scope and Outlook

Version 0.0.4 supports one directly attached low/full-speed device, one active or
retained host request, boot-keyboard HID and full-speed read-only mass storage.
Standard discovery defaults to a 512-byte configuration bound, four interfaces
and three nonzero endpoints plus EP0.
Only alternate setting zero and the first configuration are selected.

Hubs, high speed, isochronous transfers, USB device role, arbitrary HID report
parsing, storage writes, UAS and READ CAPACITY(16) are not implemented. FAT32 and
other filesystem handling belong above the block-device boundary. Type-C connection
management and Power Delivery are separate future extensions.

## 📄 License

This project is licensed under the MIT License. See [LICENSE](LICENSE).
