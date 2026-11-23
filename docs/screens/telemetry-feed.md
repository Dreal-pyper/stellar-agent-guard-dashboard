# Telemetry Feed Component

`TelemetryFeed` tails contract events from Soroban RPC:

- Queries `getEvents` with `event_auth_checked` topic filter.
- Shows the two kinds of value a row holds apart, because they are not equally
  trustworthy. The chain's own bytes — `ledger`, `ledgerClosedAt`,
  `transactionHash`, and the event XDR the feed keeps as `event.raw` — are
  `raw`. The labels `stellar-agent-guard-sdk` concluded from the topic symbols —
  `kind`, `decision`, `decision.reason`, `contractId` — are `decoded`. Each row
  says so in one line under the event name,
  `decoded by stellar-agent-guard-sdk 0.1.1 · 4 raw · 5 decoded · 0 unknown`, and
  its tooltip names every unknown field along with what is missing.
- A decoded label is the SDK's interpretation, not a value the event carried.
  "Authorization decision" is `interpret()`'s reading of the symbol
  `event_auth_checked`, and the symbol itself is rendered next to the label. A
  topic the decoder has no label for is shown as `event_… (raw topic)` rather
  than dressed up as a label.
- Unknowns are shown as unknowns. A refusal whose reason the decoder did not get
  reads `reason unknown` — never `blocked`, which is a decision, not a reason. A
  row whose original bytes the feed could not pair with it counts `raw` as
  unknown, because nothing on that row can be re-decoded independently. The
  timestamp split is the one most worth reading: the Time cell shows
  `ledgerClosedAt` (the chain's close) when the row has it and `observedAt`
  (this console's clock) when it does not, and each is listed under its own
  source.
- The version in that line is a constant in `lib/guard/decodeProvenance.ts`, not
  something the SDK publishes at runtime: `stellar-agent-guard-sdk` exposes no
  version, and its `exports` map hides its own `package.json` from a browser
  bundle. `tests/unit/decodeProvenance.test.ts` pins the constant against the
  installed package's manifest, so the row cannot show a version that was not
  the one that decoded it.
- Color-coded badges highlight blocked transactions and policy violations in real-time.

## Row severity tiers

Rows are tiered so a block is findable by looking, not by reading every row. The tier
rides on the `<tr>` as a class and a `data-severity` attribute; `data-stream` carries the
SDK's stream discriminator (`ledger` or `diagnostic`).

| Tier | Row | Marker | Wording already in the row |
| --- | --- | --- | --- |
| `blocked` | The guard refused the call | danger left rule + tinted surface | the reason badge, e.g. `per_tx_cap_exceeded`, or `reason unknown` when no reason was decoded |
| `allowed` | The guard authorized the call | none — deliberately neutral | the `allowed` badge |
| `diagnostic` | A refused-simulation row with no decision of its own | warn left rule | the `diagnostic` source chip |
| `lifecycle` | Heartbeat / admin event, no decision made | none — deliberately neutral | the event name, and `—` for the decision, which the row's provenance line counts as an unknown |

Severity is derived from the fields the decoder already produced — `decision.result` and
`event.source` (`stellar-agent-guard-sdk`, `dist/telemetry.d.ts`) — so no topic or reason
string is parsed per render, and every tier is stated in words as well as in colour. The
provenance line beside it comes from `lib/guard/decodeProvenance.ts`, which reads the same
decoded fields and reports which of them are present.

There is **no sound**. An operator console runs unattended and muted, and a noise that can
only be silenced in the tab that made it is not an alert.

## Per-event-class alert selection

The rows above are silent because the feed itself makes no noise. Sound, when there is any,
comes from the opt-in alerts panel ([`components/TelemetryAlerts.tsx`](../../components/TelemetryAlerts.tsx)),
and it is two separate choices (issue #492), not one switch. The **channel** — the Web Audio
chime, the browser notification, or both — is opt-in and both default to off. Which **event
classes** may use a channel that is on is a per-class list, ticked one at a time under
"Alert me for":

| Class | Event it covers |
| --- | --- |
| Blocked calls | the guard refused a call — an unauthorized transfer or an unexpected contract call |
| Account frozen | an admin froze the account |
| Policy revoked | a policy was revoked |

All three start ticked, so a channel an operator has switched on alerts for everything
security-critical without further configuration; narrowing it down is opt-out. The selection is
stored with the channels under `stellar-agent-guard-dashboard.alerts.v1`, and a record written
before the choice existed — one with no class list at all — keeps every class enabled rather than
disarming somebody who had alerts on. Anything that is not one of the three classes (heartbeats,
allowed decisions, the policy installs this console makes) never interrupts, whatever the list says.

The two choices are also reported separately. With every class unticked a channel is still on and
nothing can ever fire, so the panel says nothing will alert and the pill reads `off` rather than
`armed` — the rule that a control must not imply protection the tab cannot deliver. Quiet hours,
permission prompts and the browser's own refusals are each reported on their own line for the same
reason. Behaviour and its failure cases are covered by
[`tests/unit/alertClassPreferences.test.ts`](../../tests/unit/alertClassPreferences.test.ts) and
[`tests/unit/telemetryAlertsPanel.test.ts`](../../tests/unit/telemetryAlertsPanel.test.ts).

## Quiet hours

A console left open overnight is a console whose alerts get muted at the browser level, and a denied
permission is permanent in a way a switch is not. So the panel can hold its own noise back for part of
the day (issue #494). Four fields, all of them the operator's:

| Field               | What it is                                                        |
| ------------------- | ----------------------------------------------------------------- |
| On or off           | off until asked — an upgrade never silences anybody by itself     |
| `From` and `Until`  | local wall clock, `HH:MM`; the window wraps past midnight, and two equal clocks mean the whole day |
| Zone                | an IANA name, because `22:00` means a different hour in two offices |
| Interrupt anyway    | the classes that still alert inside the window (`frozen` and `policy revoked` by default) |

The state line under the controls is what makes the setting checkable: it repeats the window, the zone
it is being read in, the offset in force at this instant, and the local clock right now, and says whether
this tab is currently holding anything back. On a day with a daylight-saving transition it adds that the
local day is 23 or 25 hours long, so a wall-clock window that day runs an hour shorter or longer in real
time than it looks. Evaluation is on the wall clock in the named zone, which is what makes both
transitions fall out of the arithmetic: a `02:xx` window on a spring-forward day matches nothing, and the
repeated `01:xx` on a fall-back day is quiet on both passes.

What quiet hours do **not** do is the point of the disclosure. They act on this tab's chime and its
notifications, and on nothing else: the guard keeps enforcing, the ledger keeps recording, and the feed
keeps showing every row, at every hour. A held-back alert is not saved up for the morning either — it is
marked seen like any other, so switching quiet hours off never replays the night. An unusable zone name
is reported and the window is read in UTC instead, and a schedule that cannot be read at all leaves the
alert loud: the failure this panel will not produce is going quiet for a reason it never stated.
Semantics and both DST boundaries are in
[`lib/guard/quietHours.ts`](../../lib/guard/quietHours.ts) with
[`tests/unit/quietHours.test.ts`](../../tests/unit/quietHours.test.ts).

## Pausing and resuming

`Pause stream` freezes the table; it does not freeze the feed.

- What pause protects: the rows on screen, the filters applied to them, and the poll
  cursor. The poll loop keeps running behind the frozen table, so the panel still names
  the last successful poll while paused — "frozen" and "dead" would otherwise look
  identical.
- What arrives during a pause is queued, not shown and not discarded. The notice counts
  the queue, the button reads `Resume (n)`, and resume prepends the queue in arrival
  order. A queued event whose age takes it past the buffer limit is reported as dropped
  in the same notice rather than vanishing.
- Re-clearing with `Clear buffer` empties the list and likewise keeps the cursor, so a
  cleared feed does not re-read history.

Proof: `tests/unit/telemetryControls.test.ts` for the buffer transitions, and
`tests/e2e/telemetryPause.spec.ts` for the operator-visible contract — including that a
pause does not re-fetch the page it already consumed, which the de-duplicated table hides.

## A link to one event

Every row the console can address on chain carries `link to this row`. It focuses that
event in place — the address bar gains an `event` parameter and the feed, the wallet
session and any form draft survive the click — and the same URL is shareable.

- The link carries three parameters and nothing else: `network`, `guard`, and an `event`
  locator of `<ledger>:<transaction hash>:<topic>`. There is no event payload in the URL,
  so a shared link cannot carry a secret key, a signed envelope or transaction contents.
  Codec and validation: `lib/guard/eventLink.ts`.
- Opening a link re-reads the event from Soroban RPC in the browser, through a throwaway
  feed with a bounded number of pages. The live tail's cursor is not touched, and no
  server is involved: the console stays client-only (`npm run check:no-server`).
- An event that cannot be produced is reported by *why*, in four distinct wordings,
  rather than as a blank panel: never committed on chain (a diagnostic refusal has no
  ledger to re-read), pruned past the RPC's retention boundary, not found in the guard's
  stream, or the read failed.
- Rows the console cannot address render no link rather than one that would fail.

Proof: `tests/unit/eventLink.test.ts` for the codec, the parameter whitelist and each
unavailable reason; `tests/unit/eventDetailPanel.test.ts` for focus-without-navigation
and the panel's states.

