# Signal draft — FIPS link-layer selling (DO NOT SEND YET)

Status: draft only. This file is not a message and has not been sent. Per
`AGENTS.md`, protocol PRs/issues go to `OpenTollGate/tollgate`; FIPS itself is
contacted out of band only, after reproducible results.

## Context

We are testing whether a FIPS link can be sold at the link layer: meter FMP wire
bytes per direction, price in bytes per millisat, admit on payment, and pause
(rather than disconnect) when a prepaid allowance runs out. All work is a local,
unmerged patch against FIPS `2ba790b` (0.6.0-dev); the fleet runs 0.5.x.

## What we changed, and why

Three small, separable hooks. Each is a local patch; none is submitted.

1. **Runtime admission** (`set_admission {npub, allow|deny|clear}`).
   The ACL file reloads on a 1 s tick. Measured 0.76–0.98 s on the fleet, and
   re-admission through the file cost 0.75–0.83 s of a 2.7–2.9 s total.
   A runtime override checked before the file ACL saved **0.86–1.40 s** per
   re-admission (n=3). The override is in memory; the gate still writes the
   files for restart durability. `clear` returns authority to the files (3/3).

2. **Per-peer transit policy** (`set_forwarding {npub, allow}`).
   Today a peer either has full transit rights or is disconnected. Blocking
   transit while keeping the link, Noise session and local delivery gives the
   "pause" state the design needs. On an A–B–C chain, blocking A at B made A→C
   fail while A→B and the A–B link stayed; `allow` restored A→C. On a
   dual-homed Ethernet chain the same held 3/3.
   Per-peer drop attribution (`policy_blocked_by_peer`) was needed because the
   link counters still count frames the policy drops — a seller must subtract
   them to avoid billing refused transit.

3. **Price in `link_cost`** (`set_price {npub, price}`).
   `cost = link_cost * (1 + price)` at the existing cost-map sites. A price
   change forces one re-evaluation on the next tick. In a four-node diamond a
   1.0 penalty moved a leaf's parent in 351–513 ms with routing intact; a 0.05
   penalty never crossed the existing 0.2 hysteresis (3/3). Clearing the price
   did not revert the parent: choice is sticky, as it was after ACL-deny
   re-parenting.

## End-to-end evidence

- Unpaid → paid → pause → renew on A–B–C with real offline Cashu ecash:
  unpaid transit blocked, a real payment restores it, exhaustion pauses with
  the link id preserved, a second payment resumes. No disconnect. 3/3 UDP and
  3/3 TCP.
- A seller controller drives the same loop from session state (unpaid blocked,
  payment resumes, grace pauses, payment resumes), n=3.
- Metering is FMP frame bytes at the transport API, on UDP, TCP and Ethernet.
  Overshoot at exhaustion is one measurement step: 0.05–1.3 % with one-packet
  steps, versus 11.6 % with 40-packet batches.

## Questions for FIPS

1. Is a per-peer forwarding role something you would consider upstream? If so,
   is `Node.forwarding_blocked` + a drop in `handle_session_datagram` the right
   seam, or would you put it in the routing view?
2. Is runtime admission state better kept in memory (our patch) or persisted and
   audited? We chose memory + file write, but a restart currently widens
   admission to the file.
3. For price-aware parent selection, the 30 s `hold_down_secs` suppresses a
   price-driven switch entirely. Should a local price change bypass hold-down,
   or is that a protocol-level decision (offer/beacon) rather than a local knob?
4. Any objection to per-peer policy counters living in `ForwardingMetrics`
   (a `Mutex<HashMap<NodeAddr, _>>` on the cold path)?

## What we are not asking for

- No PR or issue against `jmcorgan/fips`.
- No change to your release process.
- We can share the patch and the lab harness (Docker compose, no hardware) on
  request.

## Reproduction

- Patch: `patches/0001-per-peer-forwarding.patch` (local, reverse-check passes).
- Results: `results/A2-2026-11-03-runtime-admission.md`,
  `results/PB-2026-11-03-per-peer-forwarding.md`,
  `results/PD-2026-11-03-pause-payment.md`,
  `results/PRICE-2026-11-03-local-cost.md`,
  `results/CTRL-2026-11-03-seller-controller.md`,
  `results/PE-2026-11-03-pause-ethernet.md`.
- Labs: `harness/pause`, `harness/pause-tcp`, `harness/exp-eth`,
  `harness/exp-eth-chain`, `harness/exp-price`.
- Test mint: Nutshell 0.21.0 `FakeWallet`; test ecash only, no mainnet funds.
