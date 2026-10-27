# Pluralis fork of iroh

Branch `pluralis/1.3.0` is upstream iroh v1.3.0 (tag `v1.3.0`, commit 0072d7d84b) plus the changes
below. The crate version stays 1.3.0. Pluralis's peer-to-peer relay pins a commit of this branch
through `[patch.crates-io]`.

Every changed site in the code carries a comment that starts with `Fork change:`. When rebasing onto
a new iroh release, `git grep -n "Fork change"` lists them all.

## Changes against v1.3.0

1. **Holepunching can be turned off** (0864b6dd07, `iroh/src/endpoint/quic.rs`).
   `QuicTransportConfigBuilder::max_remote_nat_traversal_addresses(0)` now disables holepunching;
   upstream ignores every value below 8. Nonzero values below 8 are still ignored.
2. **A full IP socket is waited for instead of dropping the datagram** (3ab105aa6e,
   `iroh/src/socket/transports.rs`, `iroh/src/socket/transports/ip.rs`). `Sender::poll_send`
   returns `Pending` for IP paths, and `SendWaiters` wakes every sender waiting on the socket.
   Relay and custom transports still drop, as upstream.

Drop each change once upstream offers the same behaviour.

## 1. Holepunching off

**How.** With 0, noq stores `None` for the limit and does not offer the NAT traversal extension, so
it is not negotiated. iroh then exchanges no address candidates and opens no new paths; its periodic
holepunching check stops at the missing extension and schedules no retry. The first path of a
connection still becomes the direct IP path after the handshake, and multipath is still negotiated.

**Why.** iroh holepunches again every 60 s while no path has an RTT of 10 ms or less, which is always
the case across a WAN. It finds the peer's other addresses (other interfaces, Docker bridges, port
mappings) and moves the connection to any new path whose RTT reads at least 5 ms lower. The old
path's RTT is usually inflated by queueing, so the new path wins. noq keeps congestion state per
path, so each move restarts slow start from a 12,000 B window, about 5 to 6 round trips.

Measurements (Pluralis, October 2026):
- Local reproduction, one relay behind a link shaped to 50 Mbit/s and 100 ms each way, 0.5 MB each way
  every 15 to 20 s for 300 s: 39 path switches, 16 holepunch attempts, an estimated 28.7 s of slow
  start, and 24 of 64 transfers over 1 s. With holepunching off: 0 switches, 0 attempts. The costly
  switches came about once per connection per minute; the old path had a median window of 499 KB
  and a median RTT of 257 ms on a 200 ms base.
- CI, data-parallel run with volunteers shaped as above: about 30% of averaging rounds (97 of 324)
  were slow, with volunteers getting their result about 1.3 s after the last reducer instead of
  about 0.4 s. With holepunching off that lag was 0.25 s at p50 and 0.28 s at p90.

Sources (Pluralis-internal): PluralisResearch/dev PRs #2044 and #2045 (descriptions, and the CI
results comment on #2045), and PR #2015 (CI results comment, for the 30%).

**Risks and limits.**
- Nothing moves a connection to a better path any more. A connection that starts on a relay server
  stays there. (Pluralis's relay runs with relay servers off, so its connections start direct.)
- A peer whose address changes cannot be followed; the connection times out and must be redialled.
- Network change on the accepting side: on a network change noq clears each path's local address. A
  multi-homed host bound to `0.0.0.0` may then reply from another source address, the peer ignores
  those packets, and the connection dies about 17 s later. With holepunching on, iroh would find the
  new address and move the connection. Single-NIC hosts, hosts behind NAT and processes in their own
  network namespace are not affected. Possible fixes: keep a path's local address across a network
  change while it still exists, or bind to the announced address.
- iroh's path selection still picks the best path across all connections to a peer and opens it on
  the others, so two connections dialled in opposite directions through NAT can still move one.
- Upstream does not test with holepunching off.

## 2. Waiting on a full IP socket

**How.** Upstream's `Sender::poll_send` reported a `Pending` transport as sent and dropped the
datagram (marked "TODO: Revisit"), so noq counted it as lost and cut its congestion window. For IP
paths it now returns `Pending`, and noq retries the buffered transmit when woken. The socket keeps
only the last waker it was polled with, so `IpSender` polls it with a shared waker (`SendWaiters`)
that wakes every registered sender, registers the caller, then polls once more so a drain in
between is not missed. Every completed send also wakes the waiters, because netwatch can drop the
registered waker without waking it while it rebinds the socket. `Pending` can also come from
Tokio's per-task budget rather than a full buffer; on loopback the kernel counted no send-buffer
errors, so that was probably the cause there (inferred). Waiting handles both.

**Why.** On loopback, a busy two-way connection dropped 0.5 to 1% of its packets this way and ran at
an unsteady 1.4 to 2.7 Gbit/s each way. Waiting gave a steady 2.1 to 2.2 Gbit/s each way, with no
loss and about the same relay CPU per GiB (with one runtime thread: 2.3 to 2.5 s against 2.2 for one
busy connection, 2.7 to 2.8 s against 2.7 for four). Sources (Pluralis-internal): the 0.5 to 1% is
from a loopback benchmark of the relay on 2026-10-15, quoted in commit 3ab105aa6e; the rest is from
PluralisResearch/dev PR #2045 (description and decision log).

**Trade-offs and risks.**
- While a send waits, the same connection's sends on its other paths wait too (for example, an IPv4
  wait delays that connection's IPv6 or relay packets). Other connections are not held back. Relay
  and custom transports still drop because they can stay blocked for seconds.
- The dropped packets were also a loss signal when the bottleneck is the host's own outgoing queue.
  Without them, on a slow, busy uplink, how much data queues there is limited by the QUIC
  flow-control windows. Measured on a saturated link shaped to 50 Mbit/s and 100 ms each way:
  with 16 MiB stream windows, the averaging roster closed at p50 / p90 4.4 / 11 s and a small control
  message took about 600 ms one way; with 4 MiB, 1.74 / 1.9 s and about 190 ms. Switching back to
  dropping (16 MiB, one run) did not shorten it (4.1 / 11.6 s), so the window is what matters.
  Size the windows for the slowest busy uplink you expect, or use a congestion controller that
  does not need loss to slow down, such as BBR. Source: Linear PRO-1491 (Pluralis-internal); the
  dropping run is in that benchmark's raw results, not in the ticket.
- `SendWaiters` takes a mutex on every completed send and on every send that finds the socket full.
  Each completed send wakes every waiting task once; those that still find the socket full register
  again, so with many connections on one full socket one send can cause that many retries. The
  waiter list keeps a task's waker until the next wake, even after its connection has ended. The
  cost was not profiled on its own; the CPU figures above include it. Many connections or several
  runtime threads on one socket were not measured.
- A waiter whose registration netwatch dropped during a failed rebind waits for the next completed
  send or its connection's own timers.
