# Working Google Chat buttons on Hermes today

**Status:** patch kit + guide. It is for a stock `hermes-agent` checkout whose
`plugins/platforms/google_chat/adapter.py` still carries the ack-only card-click stub
(true of `main` at the time of writing, commit `4566587908`). It goes stale the moment
upstream ships its own click implementation. Read
[the drift warning](#5-the-drift-warning-read-this-before-you-patch) before touching anything.

**Audience:** you run a Pub/Sub-connected Google Chat app on Hermes, plain text DMs work, and the
clarify buttons either show Google's red "<app> is unable to process your request" card or do
nothing at all. You are willing to edit one Python file and restart the gateway.

**Scope:** Pub/Sub-connected Chat apps only (the wiring the
[Google Chat docs page](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/google_chat)
describes). On HTTP callbacks a click arrives as an HTTP event, which stock `main` does not act on
either; that path is not covered here.

---

## 1. What you get

When the agent asks a multiple-choice question it calls `send_clarify`. On stock `main` that
already builds a Card v2 with one button per choice plus an "Other / type answer" button, and the
docs page says clicking a button answers the question with no configuration. That sentence is
ahead of the code. Two things are missing on stock `main`:

1. **The buttons point at the wrong topic.** For a Pub/Sub-connected app, Chat publishes the
   `CARD_CLICKED` event to the topic named in the button's `onClick.action.function`. Stock
   `send_clarify` puts a bare function name there (`hermes_clarify`), which is not a valid topic
   resource name, so the click dies inside Google with an `INVALID_ARGUMENT` and never reaches your
   subscription. The user sees the red error card; your gateway log shows nothing.
2. **Nothing handles a click that does arrive.** `_handle_pubsub_message` routes on the `ce-type`
   attribute string and its card/widget branch is `logger.info("... ack'd (v2 feature, deferred)")`
   followed by `message.ack()`. A click whose `ce-type` does not match that string falls through
   to the message path and is acked just as silently.

After the method in section 3 the buttons both **render and resolve**:

- **Clarify prompts.** Tapping a choice resolves the waiting session with that choice, the agent
  continues, and the card is patched in place to `✅ <user>: <choice>`. "Other" flips the prompt
  into typed-answer mode; an expired or unknown prompt gets an honest "this prompt expired" note
  instead of a silent drop.
- **Decision acks.** The in-place card patch is the user-visible acknowledgement, and a click is
  no longer lost to a transient failure: the handler runs to completion before the message is
  acked, a crash or timeout nacks so Pub/Sub redelivers, and a redelivered click on an
  already-resolved prompt is a no-op (no double answer). The same pipeline carries your own
  approve/skip buttons with a small handler branch, see
  [Your own decision buttons](#your-own-decision-buttons).
- **A text fallback that stays usable.** The card lists the options as text too, so a typed reply
  (the number, the option text, or a free answer) still resolves the prompt even when the click
  leg fails. If the card cannot be sent, the adapter falls back to the standard numbered text
  prompt and now logs why.

What you do **not** get: any change on the Google side. The click's publish leg (Chat app to
Pub/Sub topic) is Google's, and it can fail without a trace in your logs. Section 4 shows how to
tell that case apart without admin log access.

---

## 2. Prerequisites

1. **A working Pub/Sub-connected Chat app, wired per the
   [gateway wiring docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/google_chat).**
   In summary: a GCP project with the Google Chat API and Cloud Pub/Sub API enabled, a topic, a
   **pull** subscription on it, the Chat app configured with Connection = Cloud Pub/Sub pointing at
   that topic, and App status = Live. Text DMs must already work end to end; that proves the
   app → topic → subscription → gateway → reply leg. Once the buttons name the real topic, clicks
   ride exactly that leg.
2. **A service account with a JSON key**, referenced by `GOOGLE_CHAT_SERVICE_ACCOUNT_JSON` in
   `~/.hermes/.env` alongside `GOOGLE_CHAT_PROJECT_ID` and `GOOGLE_CHAT_SUBSCRIPTION_NAME` (the full
   `projects/<id>/subscriptions/<name>` resource path, never a bare name).
3. **IAM, on the right resources:**
   - `roles/pubsub.publisher` on the **topic** for the principal Chat publishes as (the wiring docs
     name it; it is not your service account). Without it nothing, text or click, reaches Pub/Sub
     while the gateway still shows `connected`.
   - `roles/pubsub.subscriber` **and** `roles/pubsub.viewer` on the **subscription**, for your
     service account. Subscriber alone lets a pull succeed while the metadata read the gateway does
     at startup (`pubsub.subscriptions.get`, which lives in viewer) fails. A subscription created
     seconds ago has zero bindings; nothing carries over from the subscription it replaces.
   - The service account cannot read or grant its own IAM; an admin applies these.
4. **An ack deadline that fits your agent.** Set the subscription's ack deadline to about 600 s
   (`ackDeadlineSeconds` in the API; `gcloud pubsub subscriptions update <sub> --ack-deadline=600`).
   The 10 s default expires while the agent is still generating and Pub/Sub redelivers, which
   shows up as duplicate replies. The click handler's bound (section 3) must stay under this.
5. **A checkout you can edit and restart.** The gateway imports `adapter.py` at startup, so you
   need write access to `plugins/platforms/google_chat/adapter.py` in the running install and the
   ability to restart the gateway. On the published Docker image the same edit works but is wiped
   by every image pull; bake it into a derived image or mount the patched file.
6. **Useful, not required:** admin access to the GCP project. One of the three probes in section 4
   needs it (creating a debug-tap subscription), and so does the one log that names a failed
   click publish (`logName="projects/<project>/logs/gsuiteaddons.googleapis.com%2Ferrors"`).

Confirm you are on a checkout that needs this (run from the checkout root):

```bash
grep -c '_handle_card_click' plugins/platforms/google_chat/adapter.py      # 0 on stock main
grep -n  'v2 feature, deferred' plugins/platforms/google_chat/adapter.py   # the ack-only stub
```

Zero and a hit, respectively, means this guide applies. A non-zero first count means the click
pipeline is already present; go straight to [verification](#4-verification).

---

## 3. The method

Two routes to the same result. **Route A** ports the click methods from the reference
implementation with the anchored-insert technique; **Route B** is the concept-level description of
what the adapter must do, for when you would rather write it yourself or the anchors have moved.
Both end with the same static checks and live test.

Back up first, somewhere an update cannot reach (see section 5), so rollback is one copy:

```bash
mkdir -p ~/.hermes/patch-guard
cd <checkout>
cp plugins/platforms/google_chat/adapter.py ~/.hermes/patch-guard/adapter.py.orig
```

### Route A: port from the reference implementation

The reference implementation is the patch series in this repository's `patches/` directory,
generated against upstream `main` at commit `4566587908`. Do not
apply a saved `.patch` file to your checkout: patch files are pinned to the exact upstream text
they were generated against and fail, or half-apply, on any other vintage. Instead use the
**method-block porter** technique, which is vintage-agnostic because it keys on small pieces of
text that are stable across releases:

1. **Extract, don't retype.** Slice the two method blocks (`_looks_like_card_click` and
   `_handle_card_click`) out of the reference adapter: `str.find` the line the block starts on
   (its `@staticmethod` or `async def` line) and slice to the next method definition. Write each
   block to its own plain text file. Prefer `str.find` and slicing over regular expressions; if a
   regex fails to match text that `find` locates, switch to slicing rather than debugging the
   pattern.
2. **Write the porter as a literal script that reads those files.** Pasting large blocks into
   triple-quoted template strings collides with nested quotes and the template self-destructs.
3. **Insert at asserted-unique anchors.** Every edit is "find this exact text, expect it exactly
   once, replace it with itself plus the insertion". The porter must abort **before writing** if
   any anchor is missing or duplicated. A silent no-op insert reads as success and ships dead code.
4. **Make it idempotent.** Gate on a marker (`_handle_card_click` already present → skip), so
   re-running is safe and an "already ported" exit proves the guard.
5. **Dry-run on a pristine copy first.** Copy the file, port, compile, grep markers, and only then
   touch the live file.

The edits the port consists of, with the anchors as they stand on `main` at `4566587908`
(each occurs exactly once in that file; expect them to drift):

| # | Edit | Anchor (exact text, once) |
|---|------|---------------------------|
| 1 | Initialise `self._click_topic_path = None` in `__init__` | `        self._project_id = self._subscription_path = self._bot_user_id = None  # bot id is users/{id}` |
| 2 | Capture the subscription's real topic in `_check_subscription` (Route B, step 1) | `            await asyncio.to_thread(lambda: self._subscriber.get_subscription(request={"subscription": subscription_path}))` and the `return True` after the `subscription.get failed` line |
| 3 | Name the topic on the buttons in `send_clarify` (Route B, step 2) | `        def _button(text: str, choice: str) -> dict[str, Any]:` + the `"action": "hermes_clarify"` line |
| 4 | Route and await clicks in `_handle_pubsub_message` (Route B, steps 3 and 5) | `            elif "widget" in ce_type or "card" in ce_type.lower():` + the `v2 feature, deferred` line |
| 5 | Insert both method blocks before `_check_subscription` | `    async def _check_subscription(self, subscription_path: str, credentials: Any) -> bool:` |
| 6 | Let the loop-submit helper return the future and take a `track` flag (Route B, step 5) | `    def _submit_on_loop(self, coro: Any) -> None:` |

The shape of such a porter, deliberately short so you can read all of it:

```python
#!/usr/bin/env python3
"""Anchor-and-insert porter. Every edit keys on text expected exactly once.
Aborts BEFORE writing anything if an anchor is missing or duplicated."""
import py_compile, sys

SUB_DEF = '    async def _check_subscription(self, subscription_path: str, credentials: Any) -> bool:\n'
ATTR = '        self._project_id = self._subscription_path = self._bot_user_id = None  # bot id is users/{id}\n'

# (label, old, new): `new` is `old` with an insertion before or after it.
EDITS = [
    ("attr-init", ATTR, ATTR + "        self._click_topic_path = None  # real topic the subscription binds to\n"),
    ("method-insert", SUB_DEF,
     open("block_lookup.txt").read().rstrip() + "\n\n"
     + open("block_handler.txt").read().rstrip() + "\n\n" + SUB_DEF),
    # ...sub-get, sub-tail, buttons, widget-branch, submit-helper: same pattern
]

def port(path):
    src = open(path).read()
    if "_handle_card_click" in src:
        return f"{path}: already ported, skip"
    out = src
    for label, old, new in EDITS:
        n = out.count(old)
        if n != 1:
            sys.exit(f"ABORT: anchor '{label}' found {n}x (need exactly 1); nothing written. Re-port by hand.")
        out = out.replace(old, new, 1)
    with open(path, "w") as fh:
        fh.write(out)
    py_compile.compile(path, doraise=True)
    return f"{path}: ported, compiles"

for p in sys.argv[1:]:
    print(port(p))
```

Never force an apply (`git apply --reject`, or pasting through conflicts) and never leave a
`.rej` or a conflict marker in `adapter.py`: the gateway imports that file at startup and a
marker-bearing file takes the whole unit down.

### Route B: what the adapter must do

Everything below lives in `plugins/platforms/google_chat/adapter.py`, class `GoogleChatAdapter`.
Every helper it calls already exists on stock `main`: the card builder `card_spec_to_cards_v2`,
the `_patch_message` sender, and in `tools/clarify_gateway.py` the two resolve functions
`resolve_gateway_clarify` and `mark_awaiting_text`.

1. **Read the real topic at connect.** In `_check_subscription`, keep the response of
   `get_subscription` and store its `topic` field on the adapter (`self._click_topic_path`) when it
   starts with `projects/`. Keep the old `/subscriptions/` → `/topics/` name transform only as a
   fallback, with a WARNING that says the topic was derived from the name. The transform is a
   guess that is wrong whenever the subscription is not named `<topic>-sub` (the docs' own
   example pairs `hermes-chat-events-sub` with topic `hermes-chat-events`), and a wrong guess sends
   every click to a topic that does not exist.
2. **Put the full topic path on every button.** In `send_clarify`, build `_button()` with
   `"action": self._click_topic_path` instead of the bare `"hermes_clarify"`, falling back to the
   name transform and then to the bare name. The parameters stay `{"clarify_id": ..., "choice": ...}`.
   HTTP-callback mode (no subscription path) keeps the bare action name.
3. **Gate clicks on payload shape, not the `ce-type` string.** Add a static
   `_looks_like_card_click(envelope)` and OR it with the existing `ce_type` test, because the
   `ce-type` attribute is not stable across Chat surfaces (it is empty on the add-ons surface).
   Three shapes are known:

   ```python
   @staticmethod
   def _looks_like_card_click(envelope) -> bool:
       if not isinstance(envelope, dict):
           return False
       if envelope.get("type") == "CARD_CLICKED":                 # native
           return True
       chat = envelope.get("chat")
       return isinstance(chat, dict) and ("cardClickPayload" in chat      # Workspace add-on
                                          or "buttonClickedPayload" in chat)  # Chat add-ons
   ```

   Make the non-match branch loud: before the catch-all `message.ack()`, log the unmatched
   `ce-type`, the envelope's top-level keys and a redacted dump of the envelope at WARNING
   (`Unrouted Pub/Sub event dropped: ...`). A silent ack of an unknown shape turns a one-line fix
   into a long hunt, because nothing in the log ever shows the event arrived.
4. **Resolve the click, idempotently.** Add `async def _handle_card_click(self, envelope)`:
   - Find the payload: `chat.cardClickPayload`, else `chat.buttonClickedPayload`, else the envelope
     itself (native shape).
   - Collect parameters from every place Chat has been seen to put them: `payload.action.parameters`
     (a list of `{key, value}`), `payload.actionResponse`, `payload.clickedWidget`,
     `payload.commonEventObject`, and the **envelope-level** `commonEventObject.parameters` (a flat
     `{key: value}` dict). On the add-ons surface they ride that envelope-level object and are
     absent from `buttonClickedPayload`; a handler that only looks under the payload misses them.
   - Take `clarify_id` and `choice`. Without a `clarify_id`, log at INFO and return.
   - `choice == "__other__"`: `mark_awaiting_text(clarify_id)`, then patch the card to say a typed
     answer is awaited. Otherwise `resolve_gateway_clarify(clarify_id, choice)`: on `True`, patch
     the card (a fresh `cardsV2` via `card_spec_to_cards_v2`, sent with `_patch_message` on
     `payload.message.name`) to `✅ <displayName>: <choice>`; on `False` (already resolved, expired
     or unknown), patch it to an honest "this prompt expired, please send a new request" note and
     return normally.
   - Wrap the body in `try/except Exception: logger.exception(...)`. The handler must never raise.
   - Idempotence comes from the above: a redelivered click finds the prompt resolved, takes the
     expired branch and returns; a re-patch writes identical content.
5. **Await the handler before the ack.** Replace fire-and-forget dispatch for clicks with a
   bounded run: submit the handler on the loop, wait on its future with
   `concurrent.futures.wait(..., timeout=<bound>)`, then `message.ack()` only if it completed
   without error. On a crash, a timeout, or a loop that is not accepting callbacks, `message.nack()`
   so Pub/Sub redelivers. On the timeout path **cancel the future before nacking**: a nack alone
   redelivers while the original handler still runs, and both copies execute. Keep `<bound>` well
   under the subscription's ack deadline (120 s against a 600 s deadline is a sane pairing), and
   add a short pause before the nack so a deterministic bug does not hot-loop the subscription.
   For this, change `_submit_on_loop` to return the future and accept `track=False` so awaited
   callers do not get the helper's fire-and-forget failure logging on top of their own.
6. **Make the card fallback visible.** `send_clarify` already falls back to the base-class text
   prompt when `send_card` fails, but that fallback returns the text prompt's success, so the card
   failure leaves no trace anywhere. Log the card error at WARNING before falling back, and raise
   `send_card`'s generic-exception branch from DEBUG to WARNING. If your reference carries a
   `clarify_buttons` flag, check its default: set `platforms.google_chat.clarify_buttons: true`
   when it defaults to off (read once at adapter start, so restart after changing it). The
   published patch ships it defaulting to on, as a kill-switch.

**Tests are part of the method, not an afterthought.** The google libraries are optional
dependencies, so install stub modules into `sys.modules` before importing the adapter;
`tests/gateway/test_google_chat.py` on `main` already does this (`_ensure_google_mocks`), and a
new test module can simply import it first. Cover, at minimum: the three payload shapes (and
that plain messages are not clicks); topic derivation for a subscription whose real topic differs
from the name-transform guess, plus the no-`topic`-field fallback; and the four ack paths on a
running loop: success → ack, handler crash → nack, timeout → cancel then nack, no accepting loop →
nack. `py_compile` and marker greps prove syntax, not behaviour.

### Static checks (both routes)

```bash
cd <checkout>
python -m py_compile plugins/platforms/google_chat/adapter.py
grep -c '_handle_card_click'     plugins/platforms/google_chat/adapter.py   # >= 2 (definition + call)
grep -c '_looks_like_card_click' plugins/platforms/google_chat/adapter.py   # >= 1
grep -c '_click_topic_path'      plugins/platforms/google_chat/adapter.py   # >= 3
grep -cE '^(<<<<<<<|=======$|>>>>>>>)' plugins/platforms/google_chat/adapter.py  # 0 conflict markers
python -m pytest tests/gateway/test_google_chat.py -q
```

Then restart the gateway and confirm the adapter-level connect line in its log,
`[GoogleChat] Connected; project=...`. The gateway's own `connected` status is not proof that the
pull attached.

### Your own decision buttons

Clarify is the only flow that works without further code. For approve/skip buttons on your own
cards the same pipeline applies, plus four rules learned the hard way:

- Branch `_handle_card_click` on **dedicated parameter keys** (for example `item_id` + `decision`)
  before the clarify branch; a click whose keys match neither is dropped with no error card and no
  log line.
- Buttons carry the full topic path (step 2) and the **real record id** of the item they refer to,
  never text re-derived from the display label.
- The handler only **records** the decision and re-renders the card; it never performs the effect.
  A double-click or a redelivery then lands on an already-marked entry. Chat has no per-section
  update, so the ack replaces the **whole** card: re-render it from the recorded decisions every
  time, or a bare "✅ recorded" line kills the buttons of every other item on it.
- Escape each interpolated value, not the composed card text, or the card's own formatting tags
  arrive as literal entities.

This part is application-specific and is not part of the core click patch.

---

## 4. Verification

Three symptoms, three causes, and they are told apart by what the user sees plus the subscription
backlog, before any log access:

| On tap | Backlog | Cause |
|---|---|---|
| Red "<app> is unable to process your request" card | empty | The click never left Google: the button names a topic that does not exist or is not publishable (stock `main`), or the publish grant is missing |
| Loading bar resolves, no reply | growing | The gateway is down or stale; restart it |
| Loading bar resolves, no reply | empty | The gateway consumed the click and dropped it (the ack-only stub, or a `ce-type` the string gate did not match) |

An empty backlog does **not** mean nothing arrived: the gateway drains the subscription even when
it then discards the event.

### Zero-log discriminators (no admin log access needed)

**1. The real topic behind your subscription.** Run with the gateway's own service account:

```python
from google.oauth2 import service_account
from googleapiclient.discovery import build

sub = "projects/my-project/subscriptions/hermes-chat-events-sub"
creds = service_account.Credentials.from_service_account_file(
    "/path/to/sa.json", scopes=["https://www.googleapis.com/auth/pubsub"])
meta = build("pubsub", "v1", credentials=creds).projects().subscriptions().get(subscription=sub).execute()
real, guess = meta["topic"], sub.replace("/subscriptions/", "/topics/")
print("real topic :", real)
print("name guess :", guess)
print("ackDeadlineSeconds:", meta.get("ackDeadlineSeconds"))
print("MATCH" if real == guess else "MISMATCH: a name-transform adapter publishes every click to a topic that does not exist")
```

`PERMISSION_DENIED` here while a pull succeeds means `roles/pubsub.viewer` is missing on the
subscription. A `MISMATCH` is the whole bug for any adapter that derives the topic from the
subscription name; after the patch the buttons carry `real`. This is also the check to repeat
whenever a subscription is recreated or repointed.

**2. A probe card aimed at a topic that does not exist.** Post a card whose button names a
nonexistent topic to your DM space (`spaces.messages.create` with the service account, the API
Explorer, or a short script):

```json
{"cardsV2": [{"cardId": "click-probe", "card": {
  "header": {"title": "Click probe"},
  "sections": [{"widgets": [{"buttons": [{"text": "Tap me", "onClick": {"action": {
    "function": "projects/my-project/topics/does-not-exist", "parameters": []}}}]}]}]}}]}
```

The card renders; tapping it reproduces the red error card exactly. So: probe renders and chips on
tap, alongside a real clarify card that fails the same way, proves the card payload is fine and
the failure is the click's publish leg (routing), not rendering. If the probe card itself does not
render, the problem is the send (space, credentials, card JSON), and clicks are not yet the issue.
One round separates a render bug from a routing bug.

**3. A debug-tap subscription.** A second subscription on the same topic receives its own copy of
every event and never competes with the gateway (only a second *consumer on the same
subscription* would). As admin:

```bash
gcloud pubsub subscriptions create click-tap --topic=hermes-chat-events --project=my-project
# ...tap a button on a real clarify card now...
gcloud pubsub subscriptions pull click-tap --max-messages=5 --project=my-project
gcloud pubsub subscriptions delete click-tap --project=my-project
```

- The tap has the `CARD_CLICKED` event while the gateway stayed silent: the delivery leg. The
  gateway is not consuming (not connected, wrong subscription) or it dropped the click, which is
  what stock `main` does by design.
- The tap is empty after a tap that showed the red card: Google never published. The button names
  the wrong topic or the publish leg is broken; only the project's Chat API error log names the
  topic and the reason (`INVALID_ARGUMENT` = bad topic name on the button, `PERMISSION_DENIED` =
  grant), and it is admin-gated.
- Either way, decode the `data` field (base64) to see which of the three envelope shapes your
  Chat surface actually sends, and test your gate against that captured envelope.

Do not use the service account to publish a synthetic click into the tap: it is a consumer and
normally cannot publish to its own topic, so a denial there is expected, not a finding.

### The acceptance test

Static checks, connect lines and probes are strong evidence; one live tap is proof. DM the bot
something decision-shaped ("prefer A or B?"), tap a choice on the card, and require all three:

1. the card patches to `✅ <you>: <choice>`;
2. the agent continues with that choice;
3. the gateway log shows `Card click resolved clarify` and no `Unrouted Pub/Sub event dropped`
   line for that tap. If it does show `Unrouted ... ce-type=''`, the click arrived in a shape your
   gate does not know: the redacted payload in that line is the shape to add.

Then tap a card whose prompt has already expired and expect the "this prompt expired" note, not a
crash or a silent nothing.

If the clarify arrives as plain text instead: either the caller passed no `choices` (an open-ended
question renders as text by design), a `clarify_buttons` flag is off in your vintage, or the card
send failed. After the patch the third case logs `clarify card failed (...)` at WARNING.

---

## 5. The drift warning (read this before you patch)

**Patches rot.** Everything in section 3 edits a file that the next `hermes update` (or the next
Docker image pull) rewrites. Each upstream release can move, reword or delete the anchors the port
keys on. A patch file saved today will fail to apply on a later `main`, and an update that
autostashes local changes can hit a conflict while restoring them and leave the adapter silently
unpatched (the update log names the stash to re-apply from). Nothing warns you: text DMs keep
working, and the first symptom is a button that stops answering in the middle of a conversation,
weeks later.

So, for as long as you carry this patch:

- **Keep the kit outside the checkout** (reference blocks, porter, reapply script, and the
  `.orig` backup), in a directory the update does not touch. Anything stored inside the checkout,
  including its `patches/` directory, is wiped by the very update it exists to survive.
- **Re-port per vintage, never force-apply.** After every update, grep the markers first. If they
  are gone, re-extract from a reference built against the *new* `main` and re-run the porter; its
  fail-fast anchor check is the signal that the vintage changed. If your checkout's update left a
  stash because of a conflict, the stash holds the latest patched file; recover from it rather
  than from an old backup, then regenerate the patch.
- **Keep a patch-guard.** A small idempotent script, run after updates (a cron entry is the usual
  way): markers present and `py_compile` clean → exit 0 and print nothing; markers missing →
  re-port, compile, then restart the gateway. Make the restart step self-verifying (it should
  confirm the gateway's start time actually changed) and exercise the failure path once on purpose;
  a guard that has never failed has never been tested.
- **Re-run the acceptance test after every update**, not just the marker grep. A present marker
  proves the code is there, not that Google still publishes clicks the same way.

**Re-porting per vintage is the maintenance path.** No upstream PR is currently planned, so there
is no merge to wait for: `hermes update` will not bring the click pipeline, its tests or the
`clarify_buttons` kill-switch as ordinary upstream code, and nothing here retires itself. Carry
the kit, the patch-guard and the acceptance test for as long as you want working buttons; the
patch is the destination, not a stopgap.

Rollback at any point is one copy: restore `adapter.py.orig` (or
`git checkout -- plugins/platforms/google_chat/adapter.py` on a clean tree) and restart.
