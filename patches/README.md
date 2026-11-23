# Google Chat card-click pipeline — patch series for upstream

Base: `NousResearch/hermes-agent` `origin/main` @ `4566587908`. Four commits, five files,
+722/−12. Author on every commit: patrickdass93.

## Apply

Series (keeps the four commits and authorship):

    git am 0001-*.patch 0002-*.patch 0003-*.patch 0004-*.patch

Single squashed diff:

    git apply gchat-card-clicks.consolidated.patch      # or: patch -p1 < gchat-card-clicks.consolidated.patch

Both forms were verified against a pristine export of the base commit: the resulting five
files are byte-identical to the branch tip.

## The four commits

1. **fix(gchat): name the subscription's real topic on clarify buttons**
   Each button's `function` was the bare string `hermes_clarify` — not a valid Pub/Sub
   resource name — so the click's publish leg died Google-side (`Invalid resource name` in
   the gsuiteaddons error log) and the user saw "`<app>` is unable to process your request".
   `_check_subscription` now records the subscription's topic in `_click_topic_path`; the
   `<sub>-sub` name-transform is a fallback only. HTTP-callback mode keeps the bare action
   method name.

2. **feat(gchat): resolve clarify prompts from card clicks, ack after the handler**
   `CARD_CLICKED` was an ack-only stub ("v2 feature, deferred"): fire-and-forget plus an
   immediate ack, i.e. at-most-once. Clicks now resolve the pending clarify through
   `tools/clarify_gateway`, patch the card in place to show who answered what, and are acked
   only after the handler has run. Crash or timeout → nack for redelivery, bounded by
   `_CLICK_HANDLER_TIMEOUT` (120 s, under the ack deadline); the handler task is cancelled
   *before* the nack so redelivery never runs a second handler concurrently. Three new
   `platform.google_chat.clarify.*` strings in `locales/en.yaml`.

3. **fix(gchat): route card clicks on payload shape, not the ce-type string**
   On the add-ons surface the `ce-type` attribute arrives *empty*, and a click that missed
   the "widget"/"card" substring fell into the message branch and was acked silently. A
   click is now also recognised by shape (`chat.cardClickPayload` — Workspace add-on;
   `chat.buttonClickedPayload` — Chat add-ons envelope, whose parameters ride the
   envelope-level `commonEventObject`). Every other non-message event is logged at WARNING
   with its ce-type and a redacted envelope before it is acked.

4. **feat(gchat): clarify buttons on by default behind a kill-switch; log the text fallback**
   `platforms.google_chat.clarify_buttons` (default `true`); `false` restores the
   base-class numbered-text prompt. The card carries a typed-path hint, and the existing
   card→text fallback now logs the card error at WARNING instead of failing silently.

## Config

```yaml
platforms:
  google_chat:
    clarify_buttons: false   # kill-switch only; default true. Read once at adapter start.
```

## Tests

`tests/gateway/test_google_chat_card_click.py` — 35 tests: topic-path derivation
(including the `-sub` name-transform fallback), click routing across the three payload
shapes, ack-after-handler and bounded-timeout semantics, unrouted-drop redaction wiring,
kill-switch, fallback logging and the typed-path hint. One assertion in
`tests/gateway/test_google_chat.py` was updated to the topic-path form.
