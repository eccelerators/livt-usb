# Changelog

All notable changes to this package are documented here.

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
