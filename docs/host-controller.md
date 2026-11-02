# Host-controller contract

`Livt.Usb.IHostController` is a scheduled application boundary, independent of the
physical implementation. The initial contract has exactly one request/completion
slot. Calls can span multiple clock cycles. One owner serializes staging, submission,
polling, reading, cancellation and release in the inherited clock/reset context.
An adapter is required for other clock domains or multiple concurrent clients.

## Vocabulary and bounds

`UsbSpeed`, `TransferKind`, `TransferDirection`, `SubmissionResult`, `TransferState`
and `TransferResult` are enums in `Livt.Usb`. They describe USB concepts separately
from connector shape and device class. High speed and isochronous requests can be
expressed and rejected explicitly; they are not implemented features.

`HostProfile` specifies the initial low/full-speed profile:

| Limit | Initial value |
|---|---|
| Attached devices | 1 directly attached device; no hubs |
| Interface entries | 4 |
| Endpoint entries | 4, counting control endpoint zero once |
| Active or retained requests | 1 |
| Data-stage capacity | Default 512 bytes; configurable 8..65535 |
| Setup storage | Exactly 8 separate bytes |
| Deadline | 1..65535 milliseconds (`timeoutFrames`) |

Capacity getters describe resource bounds, not the numeric address range. Endpoint
identity is `(device address 0..127, endpoint number 0..15, direction)`. Address zero
and endpoint zero are used for enumeration/control. Non-control requests require a
nonzero address/endpoint. `HostProfile` validates request geometry;
`DeviceEnumerator` discovers descriptor metadata and the class driver selects its
interface and endpoint. See [host operation](host.md#enumeration-limits).

The profile accepts control and interrupt at low/full speed and bulk at full speed.
Low-speed control uses 8-byte packets and interrupt uses 1..8. Full-speed control
and bulk use 8, 16, 32 or 64; interrupt uses 1..64. Interrupt interval is 1..255
milliseconds, otherwise zero. These checks follow the USB 2.0 transfer profiles;
see the [USB-IF specification](https://www.usb.org/document-library/usb-20-specification),
sections 5.5–5.8 and endpoint descriptors in chapter 9. Larger request lengths are
multiple-packet transfers, not larger full-speed packets.

## Staging and acceptance

While Idle, `TryStageSetup` appends setup bytes and `TryStageOutput` appends outgoing
data. `TryClearStaging` abandons staging. The controller owns and copies these bytes;
there is no externally borrowed mutable payload. Input requests require empty
outgoing data. Output requests require exactly the requested length staged.

Control requests require eight setup bytes, endpoint zero and matching little-endian
`wLength`. For a nonempty data stage, its direction must match `bmRequestType` bit 7.
For no-data control requests, submit `HostToDevice`, length zero; the controller must
execute setup and the opposite-direction status transaction. Non-control requests
must have no setup bytes staged. `HostController` handles control stages,
packet splitting and endpoint data toggles; the caller submits a complete transfer.

`TrySubmit` returns:

| Result | Meaning |
|---|---|
| `Accepted` | Metadata and bytes are frozen; the slot is Active |
| `Busy` | Active or unconsumed completed request owns the slot |
| `Disconnected` | No device is attached |
| `Unsupported` | Speed/type combination is not supported |
| `Invalid` | Geometry, deadline, interval or staged data is invalid |

Rejection changes neither staging nor the last accepted snapshot. Callers may fix
the arguments or clear/rebuild staging and retry. `Accepted` is never an assertion
that the USB operation succeeded. Connection getters are snapshots, not reservations;
disconnect can occur immediately after acceptance.

## Completion and data visibility

Poll `GetState()` for Complete, then inspect `GetResult()` and `GetActualLength()`.
Idle/Active report Pending and zero actual length. Completion remains stable until
`TryRelease`, even if the device disconnects after completion. Release discards all
staging and received data. The owner must finish reading before releasing.

Actual length counts transferred data-stage bytes, excluding setup and status.
A successful IN may be shorter than requested, including zero bytes. Actual length
below requested length describes short completion; no separate short-packet error
is needed. Successful OUT must acknowledge its full requested length. Zero-length
OUT requests represent an explicit zero-length transfer. No trailing zero packet
is implicitly added after a nonzero bulk OUT; callers submit one explicitly when
their class protocol requires it.

`TryReadInput` only exposes the initialized input prefix of a retained completion.
It returns false and zero for out-of-range, unpublished or output data. Error
completions may retain an acknowledged input prefix, but the application must not
treat it as a successfully loaded file or report.

Results distinguish Success, Stall, Timeout, Disconnected, Cancelled, ProtocolError
and Overflow. NAK does not complete a request by itself: retry within the bounded
deadline. The deadline starts at acceptance and includes scheduling, retries and
all control stages. Interrupt polling cadence must not reset it. Timeout means a
millisecond deadline, not a count of application calls.

## Cancellation and reset

`Cancel` returns true after an Active request becomes a retained Cancelled completion
and its engine has stopped using the buffers. It returns false without changing
Idle/Complete state. Cancellation cannot undo OUT bytes already acknowledged by
the device. Disconnect terminates active work as Disconnected. Reconnection must
not overwrite an unconsumed completion or reuse endpoint state from the old device.

Context reset cancels scheduled calls, discards the slot, invalidates lengths and
requires all participating components to reset/restart coherently. No completion
is promised across reset. Reset is not a request to erase BRAM payload cells.

## Test backend and resource selection

`Livt.Usb.Testing.TestHostController` implements the interface for deterministic
tests. Default data storage is 512-byte block-style RAM; an explicit
`ArrayRam<byte, 32>` can be selected with
`TestHostController<32, MemoryStyle.Block, ArrayRam<byte, 32>>`. Setup bytes are
small fixed metadata held in registers. `TransferStorage` owns the common storage
logic; `HostProfile` owns request geometry checks.

The model is driven manually: Connect/Disconnect establish attachment, ReceiveByte
injects already validated input, Complete supplies a terminal outcome, AdvanceFrame
advances one modeled millisecond, and Reset restores a disconnected Idle state.
It does not run a bus clock, enumerate devices, validate CRC, maintain endpoint
toggles or emulate packet ACK/NAK. Tests supply endpoint descriptors' geometry.
For model OUT, acknowledged progress is supplied at Complete; cancellation/timeout
before Complete reports zero, because the model has not simulated any OUT ACK.

Bind the controller through the owning component's constructor and serialize
scheduled calls. Consumers wait for call completion rather than assuming a fixed
number of cycles.
The direct physical implementation is `Livt.Usb.Host.HostController`; see
[host operation](host.md) for its timing, ownership and supported profile.

## Endpoint-halt recovery

`IHostController.ClearToggle(endpoint, inputDirection)` resets the host data toggle.
Call it only while Idle, after a successful CLEAR_FEATURE(ENDPOINT_HALT) request.
This allows class transports to synchronize host data toggles without depending
on a concrete controller implementation. Custom IHostController implementations
must implement this method.
