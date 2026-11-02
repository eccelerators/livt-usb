# Read-only mass storage

`Livt.Usb.MassStorage.MassStorageHost` owns enumeration and command traffic through
one `IHostController`. `BulkOnlyTransport` owns GET_MAX_LUN and the shared control
helper used by reset recovery. It implements command/data/status framing,
validated signatures/tags/residue/status, a single CSW stall retry, and reset recovery
in the order class reset, clear IN halt, clear OUT halt. Host data toggles follow
successful halt clears. A failed recovery disables media readiness until reconnect.
Transfer deadlines are bounded; unplug invalidates data.

`ReadOnlyScsiCommand` exposes only INQUIRY, TEST UNIT READY, REQUEST SENSE,
READ CAPACITY(10) and one-block READ(10). No write opcode is exposed. The transport
validates the exact data length for each permitted command before submitting a CBW. Zero-data commands clear the direction
flag; input-data commands set it.
Command bytes are generated on demand without retaining a full CDB register.
SCSI encodings stay in this namespace until another transport needs a shared SCSI API.

The transport-independent `Livt.IO.IReadOnlyBlockDevice` contract retains one accepted
read through Active, Complete/Failed and explicit release. Only successful complete
blocks expose bytes; consumers can pause without data loss. The storage provider
is a generic `IRam<byte>` with 512-byte `SequentialRam`/BRAM default; `ArrayRam<byte,512>`
is an alternative. A second `ENDPOINT_STORAGE` generic selects the enumerator's
32-byte endpoint table, also defaulting to `SequentialRam`/BRAM. Both providers
follow scheduled completion rather than a fixed call latency; see
[storage configuration](storage.md#selecting-nested-storage). The controller and
class buffer are separate because CSW receipt reuses the controller buffer after
data receipt.

## Reading a block

Construct the host with an `IHostController` in the same clock/reset context. The
host owns every scheduled controller call; application code accesses only its
`IReadOnlyBlockDevice` interface while the driver runs.

1. Wait for `IsReady()`, then inspect `GetBlockCount()` and `GetBlockSize()`.
2. Call `TryReadBlock(lba)` with an in-range LBA. Acceptance reserves the single slot.
3. Poll `GetReadState()` until `Complete` or `Failed`.
4. On `Complete`, use `TryReadByte(index, value)` for indices 0..511. Check its return
   value because disconnect can invalidate the block between calls.
5. Call `ReleaseRead()` after consuming the block or observing failure.

`CancelRead()` requests cancellation; wait for the read to leave `Active` before
releasing it. Context reset discards ownership and restarts discovery without
clearing RAM payload cells. No successful completion is promised across reset.

## Supported profile

The initial profile selects one full-speed SCSI-transparent (08/06/50) interface,
paired bulk endpoints, LUN zero, 512-byte sectors and capacities representable by
READ CAPACITY(10). Descriptor limits are inherited from `DeviceEnumerator`.
Multi-LUN media use only LUN zero. Readiness is attempted eight times with at least
100 ms between failed attempts.
The delay is derived from the clock context; scheduled control adds overhead.
FAT32 parsing, filenames, partition policy and UART formatting remain outside USB.

Protocol reference: [USB-IF Bulk-Only Transport specification](https://www.usb.org/sites/default/files/usbmassbulk_10.pdf).

`MassStorageHost.status` reports the lifecycle; `phase` retains enumeration or class
initialization progress on failure. Phases 7/8/9/10 identify LUN discovery, inquiry,
readiness and capacity; readiness failures 11..15 distinguish transport error,
failed recovery, unplug, cancellation and exhausted readiness retries. Ready media
reports phase 6.
