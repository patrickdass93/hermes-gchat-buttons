# Hermes Google Chat Buttons

Working card buttons for the Google Chat adapter in [Hermes Agent](https://github.com/NousResearch/hermes-agent):
clarify prompts that resolve when tapped, delivered as a patch you apply to your own checkout.

## What / Why

Stock Hermes can already post cards to Google Chat, but on a Pub/Sub-connected Chat app a button tap does
nothing useful. Two things are missing:

1. **The buttons point at a topic that does not exist.** Chat publishes each click to the topic named on the
   button (`onClick.action.function`). Stock `send_clarify` puts a bare function name (`hermes_clarify`) there,
   which is not a valid Pub/Sub resource name, so Google rejects the publish. The user sees a red "<app> is
   unable to process your request" card; nothing reaches your subscription or your log.
2. **Nothing handles a click that does arrive.** The adapter's card-click branch is an ACK-only stub (logged as
   `v2 feature, deferred`): the event is consumed and dropped. A click whose `ce-type` string does not match
   falls through to the message path and is acked just as silently.

So clarify questions can only be answered as the base-class numbered text prompt. The patch here makes the
click path work end to end: buttons name the real topic, clicks are recognised by payload shape, and the
pending clarify prompt is resolved with the card patched in place to `✅ <user>: <choice>`. Scope: clarify
buttons and click routing only, on Pub/Sub-connected apps — no approval-queue UI, no decision spooling, and
HTTP-callback deployments are not covered.

## The patch

1. **Topic read at connect.** The adapter reads the subscription's real `topic` from `subscriptions().get()` and
   puts that full path on every button, so Chat's publish leg resolves to a valid resource. The old
   `/subscriptions/` → `/topics/` name transform is kept only as a fallback for when the field is absent.
2. **Payload-shape click gate.** Clicks are recognised by envelope shape — `chat.cardClickPayload`,
   `type: CARD_CLICKED`, or the add-ons-surface shape (`chat.buttonClickedPayload`, parameters at the envelope
   level) — not by the `ce-type` attribute string, which varies across Chat surfaces and versions. An event
   matching no shape is logged at WARNING (ce-type plus a redacted envelope) before it is acked.
3. **Click handler.** A tap resolves the pending clarify prompt, records the choice, and patches the card to `✅ <user>: <choice>`.
   "Other" flips the prompt into typed-answer mode; an expired or unknown prompt gets an honest "this prompt expired" note.
   Redelivery is safe: a second click re-patches identical content, and an already-resolved prompt returns success.
4. **Ack semantics.** The handler is awaited (bounded at 120 s, well under the subscription's ack deadline)
   before `message.ack()`. A crash or timeout cancels the running handler first, then nacks, so Pub/Sub
   redelivers and no click is lost to a transient failure.
5. **Clarify card send + kill-switch.** `send_clarify` posts a button card and falls back to the base-class
   text prompt on any card-send failure, logging why. Controlled by `clarify_buttons` (default on; see [Config](#config)).

## Install

The `patches/` directory carries a numbered 4-commit `git format-patch` series
(`0001`–`0004`) generated against upstream `main` at commit `4566587908`, plus
`patches/gchat-card-clicks.consolidated.patch` (the same change as one diff, for
`git apply`). Apply whichever fits your checkout; see the Drift section if yours
is not at that vintage.

You need a hermes-agent checkout you can edit and restart, and a Pub/Sub-connected Chat app on which plain text
DMs already work (service account, IAM, ack deadline: the guide's [Prerequisites](docs/INTERIM-GUIDE.md#2-prerequisites)).
The change lives in [`patches/`](patches/): a numbered `git format-patch` series (`git am`) and one consolidated
`.patch` (`git apply`), both covering `plugins/platforms/google_chat/adapter.py` and `tests/gateway/test_google_chat_card_click.py`.

```bash
cd <your hermes-agent checkout>
grep -c '_handle_card_click' plugins/platforms/google_chat/adapter.py      # 0 on stock Hermes
mkdir -p ~/.hermes/patch-guard && cp plugins/platforms/google_chat/adapter.py ~/.hermes/patch-guard/adapter.py.orig
git apply --check /path/to/hermes-gchat-buttons/patches/<consolidated>.patch   # dry run
git apply         /path/to/hermes-gchat-buttons/patches/<consolidated>.patch   # never force it: no --reject
python -m py_compile plugins/platforms/google_chat/adapter.py
grep -c '_handle_card_click' plugins/platforms/google_chat/adapter.py      # >= 2 (definition + call)
python -m pytest tests/gateway/test_google_chat_card_click.py -q
```

Then restart the gateway and confirm its log shows `[GoogleChat] Connected; project=...`.

**If `git apply --check` fails, the file has drifted** from the vintage the patches were cut against — do not
force it. Port the change by hand: the guide's [method](docs/INTERIM-GUIDE.md#3-the-method) has the
anchored-insert technique (Route A), a concept-level spec (Route B) and the [static checks](docs/INTERIM-GUIDE.md#static-checks-both-routes).

## Config

`platforms.google_chat.clarify_buttons` — boolean, default `true`. Kill-switch only: set it to `false` under
`platforms.google_chat` in `~/.hermes/config.yaml` to send clarify prompts as the base-class numbered text
prompt instead of a button card. It switches the send path only; inbound clicks are resolved either way. The
value is read once at adapter init, so a change needs a gateway restart.

## Verification

An empty backlog proves nothing: the gateway drains the subscription even when it then discards the event.

| On tap | Backlog | Cause |
|---|---|---|
| Red "<app> is unable to process your request" card | empty | The click never left Google: bad topic name on the button (stock Hermes) or a missing publish grant |
| Loading bar resolves, no reply | growing | The gateway is down or stale; restart it |
| Loading bar resolves, no reply | empty | The gateway consumed the click and dropped it (the ACK-only stub) |

Three probes need no admin log access; the scripts and commands are in the guide's
[zero-log discriminators](docs/INTERIM-GUIDE.md#zero-log-discriminators-no-admin-log-access-needed):

1. **The real topic behind your subscription.** Call `subscriptions().get` with the gateway's own service account
   and compare the returned `topic` to the subscription name with `/subscriptions/` swapped for `/topics/`. A
   mismatch is the whole bug for any adapter that guesses the topic from the name; after the patch the buttons carry the real one.
2. **A probe card aimed at a topic that does not exist.** It renders, and tapping it reproduces the red error
   card exactly. A probe failing the same way as a real clarify card proves the card payload is fine and the
   failure is the click's publish leg (routing), not rendering.
3. **A debug-tap subscription.** A second subscription on the same topic receives its own copy of every event.
   It holds the click while the gateway stayed silent: the gateway dropped it. Empty after a red card: Google
   never published.

Then the [acceptance test](docs/INTERIM-GUIDE.md#the-acceptance-test): DM the bot something decision-shaped ("prefer A or B?"),
tap a choice, and require all three — the card patches to `✅ <you>: <choice>`, the agent continues with that choice,
and the log shows `Card click resolved clarify` with no `Unrouted Pub/Sub event dropped` line for that tap.

## Drift warning

**Patches rot.** Every patch here edits a file that the next `hermes update` (or Docker image pull) rewrites,
with no warning: text DMs keep working, and the first symptom is a button that stops answering weeks later.
This change is not in upstream `hermes-agent` and no upstream PR is pending — there is no merge to wait for, so
re-porting per vintage is the maintenance path, not a stopgap.

- **Keep this repo outside the checkout.** Anything stored inside it, including its own `patches/` directory, is
  wiped by the very update it exists to survive.
- **Re-port per vintage, never force-apply.** After every update, grep the markers above. Gone → re-apply; apply
  rejected → the anchors moved, re-port by hand per the guide.
- **Keep a patch-guard.** A small idempotent script run after updates: markers present and `py_compile` clean →
  exit 0; markers missing → re-port, compile, restart the gateway.
- **Re-run the acceptance test after every update.** A present marker proves the code is there, not that Google
  still publishes clicks the same way.

The full procedure, patch-guard included, is [the guide's drift warning](docs/INTERIM-GUIDE.md#5-the-drift-warning-read-this-before-you-patch).
Rollback at any point is one copy: restore `adapter.py.orig` and restart.
