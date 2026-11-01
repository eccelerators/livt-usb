# Package architecture

`Livt.Usb 0.0.1` owns portable USB signaling, packets, host services and supported
class handlers. Applications own terminal commands, keyboard layouts and data-use
policy. Platform integration owns clock generation, reset conditioning, VBUS,
electrical pads and timing constraints.

## Ownership and dependencies

The application constructs protocol components and supplies physical bindings.
Library components do not create a UART, CLI, platform clock or filesystem.
`Livt.Usb` uses `Livt.IO 1.6.0` for storage and `Livt.Utils 1.1.0` for CRC helpers.
Neither dependency needs to depend on `Livt.Usb`.

`HostController` owns the PHY, root-port timing, transaction engine and retained
transfer storage. One scheduled client owns staging, submission and result access.
`BootKeyboardHost` borrows that client and owns enumeration, class setup, report
parsing and its event queue. Consumers must not submit competing requests while
that driver is active.

## Namespaces and layers

Keep namespaces shallow and introduce them with implemented contents.

| Namespace | Responsibility |
|---|---|
| `Livt.Usb` | USB enums and the scheduled `IHostController` contract |
| `Livt.Usb.IO` | `IUsb2Phy` boundary and low/full-speed signaling |
| `Livt.Usb.Packets` | Packet encoding, decoding, integrity and payload storage |
| `Livt.Usb.Host` | Root-port service, transactions, transfers and enumeration |
| `Livt.Usb.Descriptors` | Bounded device/configuration/interface/endpoint parsing |
| `Livt.Usb.Hid` | Boot-keyboard lifecycle, reports and usage transitions |
| `Livt.Usb.Testing` | Manually driven controller substitution |

USB signaling and PHY adapters belong to `Livt.Usb.IO`; general-purpose I/O stays
in `Livt.IO`. `Usb2Phy` implements low/full speed only. Its name does not imply
high-speed support.

The generic CRC algorithm belongs to `Livt.Utils.Hashing`. USB polynomials,
initialization, field coverage, finalization and byte order belong to the packet
layer. Descriptor parsing retains standard metadata; class drivers decide which
interfaces to bind and which class requests to issue.

## Timing and storage boundaries

Signal-level physical and packet engines handle line-rate acceptance and response
timing. Scheduled host calls submit complete transfers and observe retained results;
their latency does not pace USB ACKs or frame service. `PacketBuffer` provides
scheduled staging with publication, rather than a direct PHY byte source.

Storage selection is compile-time composition. Buffers default to block-style
memory and expose suitable backend parameters. Metadata guards unwritten payload;
reset invalidates ownership and lengths without requiring RAM contents to clear.
Alternative backends may have different scheduled latency, but must preserve
acceptance, completion, ordering and reset semantics. Signal-level backends must
also preserve their specified edge timing.

See [host contracts](host-controller.md), [host operation](host.md),
[PHY signaling](phy.md), [packet processing](packets.md) and
[storage configuration](storage.md) for the individual boundaries.

## Extension boundaries

Mass-storage class transport is not implemented. Future USB class handlers can
reuse host control/bulk services and standard discovery. General block-device,
SCSI and filesystem facilities should be shared with appropriate storage libraries
where their contracts are independent of USB.

Type-C connection management, Power Delivery, USB device role and other bus
generations are separate concerns. A connector shape does not select a data role
or speed. Introduce those APIs when implemented, preserving separation between
physical signaling, packet transport, class behavior and application policy.
