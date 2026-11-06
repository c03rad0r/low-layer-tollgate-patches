# Local FIPS patches — private side repo

Everything here is **unmerged, unsubmitted, and not for a public fork.** This
repository is private on purpose and must stay private.

Results, harness and spec drafts live in the public evidence repo:

- https://github.com/c03rad0r/low-layer-tollgate-exploration

Per `AGENTS.md`: no PRs or issues against `jmcorgan/fips`. These patches exist so
the experiments can measure what a hook would buy. If a patch proves a finding,
the operator raises it out of band via Signal, after the result is reproducible.

Each patch is a plain `git format-patch` file against the FIPS tree the
experiment pinned.

## Patch files

| File | What | Status |
|---|---|---|
| `0001-per-peer-forwarding.patch` | (a) runtime admission, (b) per-peer transit policy, (c) price in `link_cost` | **implemented locally**; used by A2, PB, PD, MP, PE, CTRL and PRICE |

The three hooks are separable in the diff but ship as one patch series, because
they were tested together on the same FIPS revision (`2ba790b`, 0.6.0-dev).

## (a) Runtime admission

`set_admission {npub, action: allow|deny|clear}` stores a `Node.acl_overrides`
entry that `authorize_peer` checks before the file ACL. Admission changes take
effect on the next handshake instead of the 1 s file tick. It is in-memory, so
the gate still writes the files for restart durability. Test:
`runtime_admission_override_beats_the_file_acl`.

## (b) Per-peer forwarding role

`0001-per-peer-forwarding.patch` adds `Node.forwarding_blocked`, the
`set_forwarding` control command, and a transit drop in
`handle_session_datagram`. A blocked peer keeps its link, Noise session and
local delivery; only transit is dropped. Tested on an A–B–C chain: A→C fails
while A→B and the A–B link survive; allow restores transit. A four-node star
shows the block is per peer (A→D fails while C→D and A→B keep working, n=3).

This is a local, unmerged patch. Do not submit it to `jmcorgan/fips`.
The out-of-band note is drafted in `SIGNAL-DRAFT.md` and has not been sent.

## (c) Price input

`Node.link_prices` applies `cost = link_cost * (1 + price)` at every
`peer_costs` construction. `set_price {npub, price|null}` / `fipsctl set-price`
sets or clears it, and a revision counter forces one reevaluation on the next
tick. The tree's own hysteresis (0.2) and hold-down (30 s) still apply. Diamond
lab: a 0.05 price never crossed hysteresis; 1.0 moved the parent in 351–513 ms
with routing intact, and clearing did not revert (sticky).

## Rules

- Never push these to a public fork.
- Never open a PR with them.
- Each patch file carries a header comment saying it is unmerged.
- A result that depends on a patch must say so in its result file.
- Do not add lab data here; results belong in the public evidence repo.
