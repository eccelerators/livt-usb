# Packet buffer contract

`Livt.Usb.Packets.PacketBuffer<PAYLOAD_CAPACITY, STYLE, B>` owns a byte backend and metadata
for a single payload. Defaults are 64 bytes and block-style `Livt.IO.Ram`.
`BYTE_CAPACITY` is the compile-time payload limit. Any positive capacity is allowed
for staging; the caller still enforces the USB endpoint's actual packet limit.

## Lifecycle

1. Construction/reset starts unpublished with an initialized length of zero.
2. `TryAppend` writes the next byte and advances the prefix only after completion.
3. `TryPublish` freezes that prefix, including length zero.
4. `TryRead` exposes only published bytes within the prefix.
5. The owner calls `Clear` after all readers finish, allowing a new payload.

`Clear` can also abandon incomplete, unpublished data. It clears metadata and
never erases the backend. It is an owner operation, not an asynchronous reader
cancellation mechanism. There is no CRC, PID or report-format validation here;
the owner decides whether staged data may be published. `PacketReceiver` separately
validates wire packets before exposing its retained receive payload.

## Failure behavior

`TryAppend` returns false when full, published or the backend rejects the next
address. It leaves the prefix unchanged. A custom backend must support the full
configured capacity; the per-append check prevents false success if it does not.

`TryPublish` returns false when already published. `TryRead` returns false and
sets its output byte to zero when unpublished or outside the initialized prefix,
including negative indices. No failure exposes stale backend bytes. A successful
empty publication is visible through `IsReady()` even though no read can succeed.

## Ownership, clocks and reset

All operations run in the inherited clock/reset context and can take multiple
cycles. One caller serializes access; simultaneous calls from multiple processes
are outside the contract. Readers borrow bytes, not storage ownership. They must
finish before clear/reuse, including completing any scheduled read.

Context reset cancels pending execution and invalidates publication and length.
All participants must reset/restart coherently. Payload reset is backend-specific:
inferred RAM retains unspecified/old contents, while `ArrayRam` resets its cells.
Neither difference is visible through a valid published prefix. `Clear` is useful
for runtime reuse and tests but is not itself a physical reset pulse test.

## Example sequence

The owning component declares and constructs `payload: PacketBuffer` as shown in
the [README](../README.md). In one serialized process or scheduled function:

```livt
payload.Clear()
assert payload.TryAppend(0x41)
assert payload.TryAppend(0x42)
assert payload.TryPublish()
var value: byte
assert payload.TryRead(0, value)
assert value == 0x41
payload.Clear()
```

These assertions illustrate simulation checks. Production callers must handle
failure results explicitly rather than relying on assertions retained in release.
