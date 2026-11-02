# Storage configuration

Buffering components expose compile-time capacity and backend choices.
`PacketBuffer` provides scheduled payload staging with this configuration:

```livt
component PacketBuffer<PAYLOAD_CAPACITY: int = 64, STYLE: MemoryStyle = MemoryStyle.Block,
	B: IRam<byte> & new() = Ram<byte, PAYLOAD_CAPACITY, STYLE>>
```

| Configuration | Storage intent |
|---|---|
| `PacketBuffer` | 64 payload bytes in block-style RAM |
| `PacketBuffer<512>` | 512-byte staging region in block-style RAM; not a 512-byte full-speed USB packet claim |
| `PacketBuffer<64, MemoryStyle.Distributed>` | Default RAM with distributed-memory intent |
| `PacketBuffer<64, MemoryStyle.Auto>` | Default RAM with automatic placement |
| `PacketBuffer<64, MemoryStyle.Block, ArrayRam<byte, 64>>` | Explicit resettable logic-array backend; STYLE is unused by this backend |

Applications choosing these types import `Livt.IO` and `Livt.Usb.Packets` and
instantiate the exact same type in the owning constructor. There is no runtime
selector and no requirement to instantiate unused alternatives.

## Backend requirements

`B` must implement `IRam<byte>`, have a parameterless constructor, and expose stable
valid addresses from zero through `PAYLOAD_CAPACITY-1` in the inherited context. Reads and
writes must satisfy the scheduled completion contract of `IRam`. The buffer owns
the backend exclusively. It checks each append address before committing metadata;
supplying a smaller backend is a configuration error, not automatic resizing.

Metadata is separate from payload. A prefix becomes readable only after every
byte has been written and the owner publishes it. Do not reset a RAM payload array
just to make unused cells appear zero; initialization visibility is handled by the
buffer contract. A caller can explicitly choose `ArrayRam` when resettable array
storage is useful without changing the published data contract.

## Choosing defaults

Use block-style RAM by default for payload, descriptor and other substantial
buffers. Use registers for small counters,
state flags and other control metadata. A tiny buffer may waste a BRAM block;
consumers can select an array or distributed backend after considering the target.
Capacity should follow the supported profile rather than a maximum for every USB
generation. Keep defaults small and document their limits.

`MemoryStyle.Block` is an implementation hint, not proof of BRAM allocation.
Resettable logic arrays often map to registers but synthesis can optimize them.
Measure the actual LUT/register/BRAM usage and routed timing on each board.
Storage changes must preserve acceptance, completion, ordering and reset semantics;
neither a style hint nor substitution may silently add a different public protocol.

Scheduled storage can vary in latency. Wire-rate paths require a suitable
signal-level backend/adapter and their own timing tests. Do not assume the
scheduled `IRam` API has one-edge read latency.

## Host and class-driver storage

| Component | Default storage |
|---|---|
| `HostController` | Separate 512-byte input/output stores using `SequentialRam`; eight setup bytes in registers |
| Host polling timestamps | 32 entries of `logic[16]` in a separately selectable `IRam<logic[16]>` backend |
| `ConfigurationDescriptorParser` | 32-byte metadata table using `SequentialRam` |
| `BootKeyboardReportParser` | Two eight-byte report snapshots in a 16-byte `SequentialRam` |
| `KeyboardEventQueue` | 32 entries of `logic[18]` using `SequentialRam` |
| `BulkOnlyTransport` / `MassStorageHost` | One retained 512-byte block using `SequentialRam`, separate from controller storage |
| `PacketReceiver` / `PacketTransmitter` | 64-byte synchronous packet stores |

The scheduled owners serialize backend operations. The event queue arbitrates its
producer and consumer before accessing the underlying sequential RAM. Replacing
these backends with compatible arrays changes storage implementation, not the
public lifecycle or validity rules. See [host operation](host.md#transfer-behavior)
for the host's separate payload and polling-storage parameters.

Packet engines require `ISynchronousRam` with the documented accepting edge and
read-response latency. Scheduled `IRam` implementations cannot be substituted
at that signal-level boundary. See [packet storage contracts](packets.md#clock-and-storage-contracts).

## Selecting nested storage

The host owners forward their endpoint-table provider instead of fixing it inside
the enumerator:

| Owner | Generic provider | Required capacity |
|---|---|---|
| `DeviceEnumerator<LIMIT, ENDPOINT_STORAGE>` | `IRam<byte>` | 32 bytes |
| `BootKeyboardHost<ENDPOINT_STORAGE>` | `IRam<byte>` | 32 bytes |
| `MassStorageHost<B, ENDPOINT_STORAGE>` | `IRam<byte>` for each provider | 512 bytes for `B`; 32 for endpoints |

Each defaults to `SequentialRam` with `MemoryStyle.Block`. For example,
`BootKeyboardHost<ArrayRam<byte, 32>>` selects a register-array endpoint table.
`MassStorageHost<ArrayRam<byte, 512>, ArrayRam<byte, 32>>` selects arrays for both
retained block data and endpoint metadata. The providers have parameterless
constructors and are owned exclusively by their components. Scheduled operations
wait for completion, so compatible providers may trade resources for extra cycles.

`HostController` additionally forwards its final `PACKET_STORAGE` generic to
`TransactionEngine`. Its default is `SynchronousRam<byte, uint, 64, STYLE>`;
`STYLE` therefore reaches both scheduled transfer storage and wire-packet memory.
A custom packet provider must implement
`ISynchronousRam<byte, uint, RamGeometry<64>>` and preserve its accepting edge and
read-response timing. This signal-level contract does not allow arbitrary extra
read cycles. Each packet owner constructs its own provider instance.

Small descriptor headers and control fields remain register arrays. Moving tiny
buffers to scheduled RAM can cost more control logic than it saves in data storage;
choose implementations from measured synthesis and timing results.
