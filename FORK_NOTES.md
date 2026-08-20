# zestful fork of `vte`

## Base

Upstream `alacritty/vte`, tag **`v0.15.0`** (`3b3da71`) — the released version
`zestful-terminal` actually depends on, not upstream `master`, so this diff
reads as our change and nothing else.

Our work is on branch **`zestful/apc-and-osc-passthrough`**. Consumers pin it
**by rev** in `[patch.crates-io]`, never by branch: a branch pin means a
force-push here silently changes what the workspace builds.

## Delta

Two files, both additive. `alacritty_terminal` compiles against this unchanged.

### `src/lib.rs` — APC payloads reach the performer

Upstream collapses PM and APC into one `State::SosPmApcString` (`:377`) and
routes it through `anywhere()` (`:438`), which reacts only to CAN/SUB/ESC and
drops everything else. **Every APC payload byte is discarded, with no callback
and no buffering**, and `Perform` has no APC method at all.

- `State::ApcString` and `State::ApcIgnore`, split off `0x5F` only. `0x5E` (PM)
  and `0x58` (SOS) keep upstream behaviour exactly.
- `advance_apc_string`, modelled on the existing `advance_osc_string`
  (`:407-435`), reusing the `osc_raw` buffer — only one string escape can be in
  flight at a time. This is the approach upstream's own maintainer suggested on
  PR #115: *"Ideally to ensure parser state doesn't get too big one would
  likely reuse the same byte buffers for both, since it's not possible to have
  two different escape sequences at the same time anyway."*
- `const MAX_APC_RAW = 256 KiB`. Over the cap the sequence is **discarded
  whole, never dispatched truncated**. See below for how the size was chosen —
  it is *not* copied from kitty as a protocol limit.
- `Perform::apc_dispatch(&mut self, _bytes: &[u8]) {}`, defaulted.

### `src/ansi.rs` — APC on `Handler`, and unhandled OSCs pass through

- `Handler::apc_dispatch`, defaulted, and `Performer::apc_dispatch` forwarding
  to it. This is the seam `Term` implements.
- `Handler::osc_unhandled(&[&[u8]])`, defaulted. Upstream's nested
  `unhandled()` inside `osc_dispatch` `debug!`s and drops; it now also forwards.
  vte dispatches OSC 0/2, 4, 8, 10/11/12, 22, 50, 52, 104, 110/111/112 and
  **everything else falls to that site** — so OSC 7, 9;4, 133 and 777 are
  unreachable without this. With it, each is a match arm in the embedder rather
  than a second parser in its PTY read path.

## Why

The kitty graphics protocol is carried entirely over APC, so a terminal
implementing it cannot receive a single byte of the protocol through stock vte.

Two workarounds were considered and rejected. A **byte tee** ahead of the parser
loses placement anchoring: the spec anchors an image at the cursor position when
the final chunk arrives, and a tee reading the same buffer sees that position
only after the parser has already consumed whatever moved the cursor later in
the same `read()`. An **APC-truncating `Read` filter** — the approach Zellij's
`KittyApcInterceptor`, `par-term-emu-core-rust`'s `apc_filter.rs` and `kou` all
ship — is correct on anchoring, but it puts terminal-state mutation inside a
`Read` impl and needs re-deriving the parser's own framing rules to find the
boundaries.

## Two behaviours this changes, deliberately

**1. BEL terminates an APC here.** This is a deviation from the vt500 state
diagram vte implements, which specifies `00-17,19,1C-1F,20-7F / ignore` for the
sos/pm/apc string state — `0x07` is inside `00-17`, so **upstream vte is
conformant and is not buggy here.**

The justification is robustness, not conformance and not compatibility. An APC
that never receives ST silently destroys all subsequent output to the next ESC;
a client emitting BEL (which current kitty accepts) hits exactly that. Accepting
BEL costs nothing — a conformant client sends ST and is unaffected.

It is **not** true that every terminal does this. The ecosystem disagrees four
ways and vte is not an outlier:

| terminal | BEL inside APC |
|---|---|
| kitty | terminates the APC (both parser generations; undocumented) |
| xterm | rings the bell and **continues** the string |
| wezterm | **appends it to the payload**; BEL-as-terminator scoped to OSC on purpose |
| foot | ignores it — byte-for-byte identical to upstream vte |

All of that is **source reading; nobody has run these terminals.**

**2. `MAX_APC_RAW` bounds a buffer upstream does not have.** Upstream buffers no
APC at all, so it has no cap to diverge from.

*The size* is chosen from what a legitimate APC needs: the graphics protocol
recommends 4096-byte chunks, so 256 KiB is ~64 chunks of headroom, and it is not
smaller because a single non-chunked transmission is permitted and can be large.
It happens to equal kitty's `MAX_ESCAPE_CODE_LENGTH` (`BUF_SZ / 4u`,
vt-parser.c:18-21). That is a sanity check, **not a precedent** — kitty's number
is the largest escape it will *buffer*, and it keeps a streaming hatch behind
it, special-casing OSC 52 on overflow to dispatch a partial payload and continue
(vt-parser.c:459-471). Nothing in the protocol specifies a maximum APC length.

*The policy* — discard rather than truncate-and-dispatch — is the load-bearing
choice. A truncated graphics command that looks well-formed corrupts the
*assembled* image under `m=1` chunking, which is worse than a clean refusal.
kitty moved the same way: v0.32.2 truncated and dispatched (parser.c:1228-1231);
current kitty reports an error and discards (vt-parser.c:472-473). We improve on
it in one respect — kitty returns to ground immediately, so an oversized APC's
tail is printed as text; we consume to the terminator first.

## Rebase costs

- **`exceed_max_buffer_size` needs no edit.** This was expected to conflict:
  upstream *deliberately tests* that `MAX_OSC_RAW` is inert under `std`
  (`lib.rs:1019-1050` asserts std keeps the oversized payload, `no_std`
  truncates). **Measured: it passes unmodified.** Our divergence is scoped to
  the APC state; the OSC path keeps upstream's unbounded-under-std behaviour
  untouched. All **54** upstream tests pass with zero modifications.
- **The OSC passthrough is the part we carry longest.** A blanket passthrough
  exposes everything upstream chose not to implement and is a larger ask than
  the APC methods. Expect to keep rebasing it even if the APC half lands
  upstream.
- **`unhandled()` gained a parameter** (`fn unhandled<H: Handler>(handler, params)`),
  touching 13 call sites inside `osc_dispatch`. Mechanical, but it is the part
  most likely to textually conflict when upstream adds an OSC.
- **PR #115 is not an exit path for this fork.** It does the APC half, applies
  cleanly to v0.15.0 and passes tests — but it was **never approved** (zero
  approving reviews, three `CHANGES_REQUESTED`, closed 2026-07-12 for
  inactivity, not on the merits), and it is the *parserless byte-at-a-time*
  design (`apc_put()` per byte), not our buffered `apc_dispatch(&[u8])`. It
  supplies no `MAX_APC_RAW`, because it buffers nothing. Reviving it upstream
  would not be a drop-in retirement of this fork.

## Tests

`cargo test --features std,ansi` → **70 passed**: upstream's 54 unmodified, plus
16 covering APC delivery, chunk reassembly across `advance` calls, BEL and ST
termination, the cap boundary and its discard policy, CAN/SUB abort without
dispatch, PM/SOS remaining untouched, `osc_raw` sharing, and the OSC
passthrough. `cargo build --no-default-features` (no_std) is clean.
