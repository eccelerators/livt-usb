# Clock configuration

Library components inherit the owning clock/reset context. They do not create a
PLL or select a board oscillator. Place `HostController` in a suitable context;
its PHY, root port and transaction engine share that clock. Crossing clocks
requires an explicit adapter.

## Sampled PHY

`Usb2Phy`, `Usb2LineReceiver` and `Usb2PacketTransmitter` require an exact integer
multiple of 12 MHz, with at least eight samples per full-speed bit. For example:

| Context clock | Full-speed samples/bit | Low-speed samples/bit |
|---|---:|---:|
| 96 MHz | 8 | 64 |
| 108 MHz | 9 | 72 |
| 120 MHz | 10 | 80 |
| 144 MHz | 12 | 96 |

The minimum accommodates the sampler/decoder pipeline and packet RAM prefetch.
100 MHz is not an exact divisor of USB bit timing; 48 MHz supplies too few samples
for this implementation. Select a suitable PLL output in the integration.
Construction assertions reject unsupported rates. Release builds also suppress
capabilities, packet acceptance and pad drive at unsupported rates.

Bit periods, EOP, idle/SE0 qualification and receive sampling phase follow the
context. Counter widths grow with the required counts. Pipeline stages remain
fixed clock-cycle latencies. A faster context still needs implementation timing
closure, clock accuracy and electrical verification for the chosen target.

## Pipeline timing

`HostController` and `TransactionEngine` enable `PIPELINE_LINK` by default. The
added stages use the inherited clock; they do not select a fixed clock frequency
or change USB bit periods. Disabling the option removes the optional stages,
while preserving the same supported clock rates and packet-storage contract.

A standalone `PacketTransmitter` defaults to `PIPELINE_OUTPUT = false`; the host
engine selects it through `PIPELINE_LINK`. Both settings require at least 64
clocks between PHY byte acceptances. See [link pipeline](host.md#link-pipeline)
and [transmit pipeline](packets.md#optional-transmit-pipeline) for the boundaries.

## Host timing

| Timing | Default / unit |
|---|---|
| Attach debounce | 100,000 microseconds |
| Bus reset | 12,000 microseconds |
| Reset recovery | 10,000 microseconds |
| Frame interval | 1 millisecond |
| Disconnect SE0 | 2.5 microseconds, rounded up to a whole tick |
| New-transaction frame guard | 200 microseconds |
| Response-start deadline | 18 bit times at the selected speed |
| IN response window | 128 microseconds |
| Other response window | 32 low-speed bit times |
| Transaction gap | 2 microseconds |
| SET_ADDRESS recovery | At least 2 milliseconds |

`RootPort<DEBOUNCE_US, RESET_US, RECOVERY_US>` and the corresponding
`HostController` parameters express durations, not clock counts. All three must
be positive and their sum must not exceed one second. Shortened durations are
useful for simulation; production integrations should retain the defaults.
A standalone `RootPort` requires a whole-MHz context. The integrated host has the
stricter PHY constraint above. Host deadlines and endpoint polling intervals use
millisecond frames, so their meaning does not change with the clock.

## Duration conversion workaround

The current compiler rounds `context.periodNs` before evaluating `TicksFor`.
For example, `TicksFor(1ms)` incorrectly produces 100,000 ticks at 96 MHz.
To avoid this rounding error, physical timing uses exact `TicksPer` ratios:
`TicksPer(1MHz)` for microseconds, `TicksPer(1000Hz)` for milliseconds, and
USB bit-rate ratios for serialization. The supported clock constraints make
these ratios exact. Standalone `DeviceEnumerator` rounds its two-millisecond
wait up using the context frequency in Hz, so it also supports other controller
clocks. Do not replace these calculations with rounded nanosecond periods.

The current compiler can select the wrong resize helper for unsigned arithmetic
cast to a context-sized logic vector. Bounded timing arithmetic therefore passes
through `int` before narrowing. These counts fit in signed 32 bits; this source
workaround preserves their values without modifying generated HDL.
