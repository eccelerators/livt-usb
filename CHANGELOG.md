# Changelog

All notable changes to this package are documented here.

## 0.0.4

### Added

- `PIPELINE_LINK: bool = true` on `TransactionEngine` and `HostController` for
  configurable receive sampling, ACK decisions, frame admission and transmit
  feedback. The host forwards this choice to its packet transmitter.
- `PIPELINE_OUTPUT: bool = false` on `PacketTransmitter` for byte/valid publication
  one clock edge after phase advancement. Direct packet users retain the default
  unpipelined behavior.
- Transaction tests against the physical transmit serializer for both settings,
  covering IN/OUT bytes, CRCs, retained payloads, duplicate acknowledgement,
  cancellation, frame admission and bounded ACK handoff.

### Changed

- The default host separates PHY observation, receive processing and transmit
  preparation with pipeline stages. Set `PIPELINE_LINK` to `false` for the direct
  path with lower latency.
- Receive completion uses the byte count and checksum before EOP, avoiding the
  active-byte update path without changing packet publication timing.

### Compatibility and dependencies

- New generics are appended, preserving existing positional arguments. Default
  host wire timing gains pipeline cycles; scheduled transfer ownership and
  retained-completion semantics remain unchanged.
- Packet storage providers retain their synchronous read-response contract, and
  PHY byte acceptance must remain at least 64 clocks apart. Pipeline selection
  does not permit arbitrary memory latency.
- Continue to use published `Livt.IO 1.7.0` and `Livt.Utils 1.1.0`. No filesystem,
  UART, CLI or board dependency is introduced.

## 0.0.3

### Added

- `MassStorageHost`, `BulkOnlyTransport` and read-only SCSI command encoding for
  direct full-speed, LUN-zero, 512-byte block devices through
  `Livt.IO.IReadOnlyBlockDevice`.
- Validated command status, bounded readiness retries, retained sense information,
  stall handling and BOT reset recovery.
- `IHostController.ClearToggle` exposes endpoint toggle reset after successful halt
  clearing. Custom controller implementations must implement this member.

### Changed

- Exposed endpoint-table providers through `DeviceEnumerator`, `BootKeyboardHost`
  and `MassStorageHost`, retaining BRAM defaults and scheduled storage contracts.
- Forwarded `HostController` packet storage and `STYLE` to the transaction engine;
  custom packet memory retains the synchronous port timing contract.
- Reduced retained BOT validation temporaries and grouped independent metadata
  updates into shared states without changing retained-block ownership.

### Compatibility and dependencies

- `IHostController` implementations must provide `ClearToggle` for endpoint-halt
  recovery. Existing consumers retain default storage configurations.
- Depend on published `Livt.IO 1.7.0` for memory providers and the
  `IReadOnlyBlockDevice` contract; replace the development path dependency with
  this registry version.
- Continue to use `Livt.Utils 1.1.0`. UART/CLI integration, filesystems and board
  support remain outside this package.

## 0.0.2

### Changed

- Replaced sibling path dependencies with published registry versions:
  `Livt.IO 1.6.0` and `Livt.Utils 1.1.0`.
- Updated package usage documentation for installation without sibling library checkouts.

## 0.0.1

### Added

- Low/full-speed `Usb2Phy` and `IUsb2Phy` signal contract, with context-derived sampling,
  NRZI, bit stuffing, SYNC/EOP, reset signaling and low-speed keep-alive.
- Context-derived bit periods, host delays, frame timing and counter widths;
  configurable root-port durations use microseconds. The sampled PHY accepts
  exact multiples of 12 MHz with at least eight samples per full-speed bit.
- Token, data and handshake packet processing, PID/length/CRC checks, full-speed
  SOF, bounded payload retention and explicit cancellation.
- `IHostController` and a direct host implementation with root-port timing,
  control/interrupt transfers, full-speed bulk transfers, endpoint data toggles,
  deadline-bounded retries and retained completion.
- Standard device/configuration discovery and bounded interface/endpoint parsing.
- HID boot-keyboard initialization, polling, report validation, usage transitions
  and a bounded event queue. Repeated reports and duplicate usages do not repeat
  key presses; disconnect and queue overflow are observable.
- Scoped keyboard-interface selection for Rapoo `24AE:2010` and `24AE:2013`, plus
  cached device identity and binding observations.
- Configurable capacities and storage providers with BRAM defaults, including
  scheduled packet buffers, transfer storage, descriptor metadata and event queues.
- `TestHostController` for deterministic scheduled-interface tests, plus library
  regressions including root-port timing across clock contexts.
- Architecture, API, timing, ownership and storage documentation.

### Dependencies

- `Livt.IO 1.6.0` supplies memory contracts and storage implementations.
- `Livt.Utils 1.1.0` supplies protocol-independent CRC helpers; USB CRC profiles
  and wire serialization remain in `Livt.Usb.Packets`.
- The initial manifest resolved both dependencies through sibling checkout paths.
  UART transport, CLI commands and platform integration remain application choices.

### Scope

- One directly attached low/full-speed device and one active or retained request.
- Boot-keyboard HID; layout, text conversion, LEDs and repeat policy are application-owned.
- No hubs, high-speed or isochronous transfers, arbitrary HID report parsing,
  mass-storage class driver, USB device role, Type-C management or Power Delivery.
