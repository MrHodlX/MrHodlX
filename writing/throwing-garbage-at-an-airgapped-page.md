# Throwing 4 MB of garbage at an air-gapped page, every night

*EntropyLab's adversarial harness: hostile input delivered for real, judged
against three invariants — and the traps that almost made the whole thing
test nothing.*

EntropyLab is a single HTML file meant to run on an air-gapped machine, where
it will be fed seeds, PSBTs, and journal files typed or pasted by humans who
make mistakes. The security promises are specific: never touch the network,
never crash on hostile input, always *tell the user* when input is rejected.
Unit tests cover the parsers. But the promises are about the *whole page*, so
we test the whole page — by attacking it.

Every night, [`test/adversarial/harness.mjs`](https://github.com/MrHodlX/entropylab/blob/main/test/adversarial/harness.mjs)
launches headless Chrome, drives the built page over the Chrome DevTools
Protocol (zero npm dependencies — Node's native WebSocket), and runs eight
hostile-input scenarios against it.

## The invariants

Before any page script runs, the harness installs a tap that wraps every
network primitive — `fetch`, `XMLHttpRequest`, `sendBeacon`, `WebSocket`,
`EventSource` — and records every uncaught exception and unhandled rejection.
After each scenario, three invariants are checked:

1. **Zero network attempts.** An air-gapped tool that "phones home" only
   under weird input is worse than one that always does — you'd never notice.
   One recorded attempt fails the run.
2. **No uncaught exceptions or unhandled rejections** while digesting the
   input.
3. **The page is still alive and responsive** afterwards.

Plus scenario-specific assertions: the app must actually *report* bad input
through its own error UI. Silently swallowing garbage is a fail, not a pass.

## The scenarios

A flavor of what runs (`scenarios.mjs`):

- **`psbt-garbage-paste`** — junk, NUL bytes, and a `<script>` tag into the
  PSBT inspector. Undecodable input must disable the Inspect button; base64
  that decodes but isn't a PSBT must re-enable it and surface the decoder's
  error. (The script tag is bait — it must end up as text, never markup.)
- **`psbt-oversized-paste`** — ~4 MB of base64-looking junk. The page must
  stay responsive while it chews.
- **`seed-encoding-tricks`** — RTL overrides, zero-width characters, and
  homoglyphs into a seed field. Unicode is where input handling goes to die.
- **`journal-hostile-import`** — malicious JSON (script tags, wrong types,
  huge) dispatched through the file-import input as a real `File` object.
- **`workspace-tab-hammer`** — 40 rounds of clicking every workspace tab,
  looking for exception accumulation and state leaks between views.
- **`dice-oversized`** — 120k characters plus NULs into the dice field.

## Trap one: the run that tests nothing

The most dangerous outcome of an adversarial harness isn't a crash — it's a
*green run that never delivered its payload*. A selector goes stale after a UI
rename, the paste lands nowhere, the scenario logs "garbage pasted, no
errors," and CI stays green while testing a page that was never touched.

So every payload delivery is recorded: the helper pushes
`{ label, sent, landed }` for each paste and upload, and the harness turns a
complete non-delivery into an invariant failure:

> `payload never landed in #dice (sent 120000 chars, field holds 0) — scenario did not exercise its target`

A broken *scenario* now fails the run just like a broken app. Partial
filtering is fine — the dice field legitimately keeps only digits — but zero
landing means the scenario proved nothing.

Related subtlety for file uploads: read the attachment *before* dispatching
the `change` event, because the app's own handlers consume and clear the
input. Checking afterwards reports zero for a delivery that actually worked —
a false failure that would send you hunting a bug that isn't there.

## Trap two: timing your own sleep

The first version of the oversized-paste scenario measured responsiveness
like this:

```
t0 = now; await sleep(4000); report(now - t0)
```

That reports ~4000 ms *no matter what the app does* — it times its own sleep.
The fixed version measures the two things that actually describe
responsiveness: the **worst main-thread block** (the longest gap between
`setTimeout(0)` ticks) and the **time until the app answered through its own
error UI**. A frozen page and a snappy one now produce wildly different
numbers, which is the point.

## Trap three: judging severity by hand

Eight scenarios produce a lot of log to eyeball nightly. So each scenario's
compact log goes to an LLM judge (Jev, via a pinned CLI) that returns
**ok / suspect / broken** plus severity and network-egress scores. The
discipline is in what each verdict *does*:

- **broken** fails the run (exit 1) — the workflow status means something.
- **suspect** does *not* fail — it's a flag for a human to read in the table.
- **No verdict at all** (API down, no key) also doesn't fail — the invariants
  above are the ground truth; the judge is triage, not oracle.

An LLM that can turn CI red on a hallucination is a liability. An LLM that
reads logs and ranks what a human should look at first is a genuinely useful
tool. The harness keeps it in the second bucket.

## What this buys

The parsers were already unit-tested. What the harness adds is confidence
about the *composition*: the built page, real Chrome, real DOM events, real
`File` objects, on a `file://` URL with no debug flags — the same artifact
and the same environment the air-gapped user gets.

And the meta-lesson generalizes to any security-sensitive UI: **assert that
your attack actually happened.** An adversarial suite that can't tell "app
handled the payload" from "payload never arrived" is security theater with a
green checkmark.

---

*Source: [`test/adversarial/`](https://github.com/MrHodlX/entropylab/tree/main/test/adversarial)
in the EntropyLab repo — `harness.mjs`, `scenarios.mjs`, and the README
documenting the setup.*
