# Web Haptics API

Authors: [Nesh Gandhe](https://github.com/neshgandhe), [Kevin Babbitt](https://github.com/kbabbitt), [Limin Zhu](https://github.com/liminzhu), [Alexander Kyereboah](https://github.com/kyerebo)

## Status of this Document

This document is a starting point for engaging the community and standards bodies in developing collaborative solutions fit for standardization. The API is in the early ideation and interest-gauging stage, and the solution/design will likely evolve over time.

* This document status: **Active**
* Current version: **This document**

## Table of Contents
- [Introduction](#introduction)
- [User-Facing Problem](#user-facing-problem)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [Proposed Approach](#proposed-approach)
- [Real-World Scenarios](#real-world-scenarios)
- [Future Extensions](#future-extensions)
- [Alternatives Considered](#alternatives-considered)
- [Accessibility, Privacy, and Security Considerations](#accessibility-privacy-and-security-considerations)
- [Open Questions](#open-questions)
- [Reference for Relevant Haptics APIs](#reference-for-relevant-haptics-apis)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & Acknowledgements](#references--acknowledgements)

## Introduction

Modern operating systems have embraced haptics as a core part of user experience — providing subtle, low-latency tactile cues that reinforce visual and auditory feedback. These signals improve confidence, precision, and delight in everyday interactions. The Web Haptics API proposes a semantic, cross-platform interface that connects web applications to native haptic capabilities. By focusing on intent-driven effects, the API enables web apps to deliver tactile feedback consistent with OS design principles, while preserving user privacy and security.

This proposal offers two complementary mechanisms:

1. **Declarative API (CSS)** — an event-trigger model that maps user-interaction triggers to haptic effects through the `haptic` property.
2. **Imperative API (JS)** — `navigator.playHaptics(effect, intensity)` for interactions that require runtime logic or lack a supported declarative trigger.

## User-Facing Problem

The [navigator.vibrate()](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/vibrate) API exists today for basic haptics. However, it is mobile-centric, lacks broad engine and device support, and requires developers to manually program duration/pattern sequences — a low-level interface that doesn't map to the way designers think about haptic intent.

Beyond the limitations of the existing API, there is no declarative way for developers to add haptic feedback to common UI interactions — scroll-snap carousels, panel transitions, form validation — without JavaScript in the critical path.

## Goals

- Bring standardized, semantic haptic feedback to web apps across desktop and mobile platforms.
- Allow developers to signal intent/effect rather than programming raw patterns.
- Focus on reactive haptics feedback (i.e. haptics immediately after user input).
- Enable low-latency haptic feedback for common interactions without requiring JavaScript.
- Extensible interface for future haptics advancement on the web.
- Respect platform haptics user settings if available.
- Minimize privacy/fingerprinting concerns.

## Non-goals

- Guarantee identical tactile output across platforms — different platforms and user agents may choose varied output that best matches the intent.
- Cover haptics notification scenarios (e.g. vibrate to alert users when long-running task is completed).
- Cover/replace API for highly specialized hardware, namely gamepad.

## Proposed Approach

The Web Haptics API uses a predefined list of effects with an optional intensity parameter, without exposing raw waveform authoring or low-level parameters like duration, sharpness, or ramp. Developers request a named effect, and the user agent maps it to the closest native capability (which may be a generic pattern if OS or hardware support is lacking). To minimize fingerprinting risks, the API does not currently allow developers to query haptics-capable hardware or available waveforms.

For both imperative and declarative APIs, target selection follows one shared model: dispatch to the most recent input device. If that device is not haptics-capable, no haptic is played. User agents do not reroute to another connected haptics-capable device.

Both APIs are governed by a `"haptics"` [permissions policy](https://w3c.github.io/webappsec-permissions-policy/) with a default allowlist of `"self"`. Cross-origin iframes must receive explicit delegation (e.g. `allow="haptics"`) to play haptics. See [Security](#security) for full details.

Both the imperative and declarative paths share the same effect vocabulary.

### Effect Vocabulary

| Value   | Description | When to reach for it |
|---------|-------------|----------------------|
| `hint`  | A light, subtle cue that signals something is interactive or an action may follow. | Focusing an input field, entering a drop zone during drag. |
| `edge`  | A heavy boundary signal that indicates reaching the end of a range or hitting a limit. | Pull-to-refresh threshold, scroll hitting a boundary, reaching the end of a control range. |
| `tick`  | A firm pulse that marks discrete changes, like moving through a list or toggling a switch. | Scroll-snap landing, stepping through picker values, toggling a switch. |
| `align` | A crisp pulse when an object locks into place or aligns with guides or edges. | Drag-to-snap, window snapping to screen edges, zoom snapping to 100%. |
| `success` | A confirmation that communicates the successful completion of an action. | Form submitted, file saved, transaction completed, operation finished successfully. |
| `error` | A confirmation that communicates that an action failed or could not be completed. | Form submission failed, save failed, operation rejected, action could not be completed. |

hint, edge, tick, and align provide feedback during an interaction, while success and error communicate the outcome of a completed action. The mappings below are illustrative and may vary by platform. User agents may choose different mappings, including synthesizing custom effects from lower-level primitives and parameters.

| Web Haptics | Windows | macOS | iOS | Android |
|:-----------:|:-------:|:-----:|:---:|:-------:|
| hint | hover | generic | light impact | gesture_threshold_deactivate |
| edge | collide | generic | soft impact | long_press |
| tick | step | generic | selection | segment_frequent_tick |
| align | align | alignment | rigid impact | segment_tick |
| success | success | generic | success | confirm |
| error | error | generic | error | reject |

**Intensity** is always a normalized value between 0.0 and 1.0. If the platform exposes a system-level intensity setting, the effective intensity is `system intensity × developer-specified intensity`. Intensity defaults to 1.0 if left unspecified.

### Imperative API (JS)

The imperative API is not gated behind a permission but requires [sticky user activation](https://developer.mozilla.org/en-US/docs/Web/Security/User_activation).

```js
navigator.playHaptics(effect, intensity);
```

**Parameters:**
- `effect` — one of the predefined effect names: `"hint"`, `"edge"`, `"tick"`, `"align"`, `"success"`, `"error"`.
- `intensity` *(optional)* — a normalized value between 0.0 and 1.0. Defaults to 1.0.

The API always returns `undefined`. No haptic is played if the last input device is not haptics-capable, and the user agent does not reroute to another connected haptics-capable device. If sticky user activation has expired, the call is silently ignored.

### Declarative API (CSS)

The declarative API maps semantic browser events to haptic effects through an event-trigger model — no JavaScript required.

Haptics fire when associated triggers activate, based on the defined `Event`. [CSS Animation Triggers](https://drafts.csswg.org/animation-triggers-1/) defines event-trigger firing, naming, scoping, targeting, and propagation; Web Haptics defines how a haptic effect consumes a fired trigger. This makes the intended event source explicit and allows haptics to build on trigger infrastructure shared with animations.

#### Event triggers and the `haptic` property

`event-trigger` defines named triggers from events. The proposed `haptic` property consumes those triggers to play effects:

```css
button {
  event-trigger: --activate click;
  haptic: --activate align 0.8;
}
```

For common one-off cases, a proposed inline/anonymous form uses an event function directly instead of naming the trigger:

```css
button {
  haptic: click align 0.8;
}
```

**Value grammar.** For named triggers, `haptic` accepts a comma-separated list of mappings:

```text
haptic: [ <trigger-name> <effect-name> <intensity>? ]#
```

Here, `#` means one or more comma-separated mappings — for example, `haptic: --press hint 0.4, --activate align 0.8`.

- `<trigger-name>` — a named event trigger, such as `--activate`.
- `<effect-name>` — one of `hint`, `edge`, `tick`, `align`, `success`, `error`.
- `<intensity>` *(optional)* — a `<number>` between 0.0 and 1.0. Defaults to 1.0.

**Triggering behavior:**

- **Trigger activation.** Each time a referenced named or anonymous trigger fires, its mapping is eligible to produce a haptic request. Repeated activations can request the same effect without changing the computed `haptic` value.
- **Same-element resolution.** The CSS cascade determines the computed `haptic` value; mappings from separate rules do not automatically accumulate. If multiple mappings in that value activate in the same processing update, the last matching item produces the request.
- **No initial haptic.** Initial rendering or a selector-matching change alone does not play a haptic. A trigger must fire.
- **Request coalescing.** A user agent fires at most one haptic per target device per rendering frame. After accepting one request, it suppresses later requests for that device in the same frame. User agents may apply additional throttling.
- **Target selection.** The most recent input device is targeted. If it is not haptics-capable, no haptic fires and the request is not rerouted.

**Eligible triggers.** Haptics are limited to trigger activations causally associated with user input. Animation Triggers determines whether script-generated events can fire a trigger; Web Haptics defines any additional haptics-specific activation restrictions. The exact eligibility rules — including how scroll-snap interactions and script-generated events qualify — remain to be defined.

**Example — reusable trigger with multiple event sources:**

```css
.submit-button {
  event-trigger: --submit click keypress("Enter");
  haptic: --submit align 0.9;
}
```

### Filtering by input device type

Developers can use the Pointer Events API's [`pointerType`](https://www.w3.org/TR/pointerevents/#dom-pointerevent-pointertype) property to filter imperative haptic requests to desired input categories, such as mouse or pen:

```js
button.addEventListener('pointerdown', (event) => {
  if (event.pointerType === 'mouse' || event.pointerType === 'pen') {
    navigator.playHaptics?.('align', 0.8);
  }
});
```

For a pen-only experience, check only for `"pen"`; `"touch"` identifies touchscreen input. An existing limitation of Pointer Events is that mice and touchpads are both reported as `"mouse"`, so `pointerType` cannot distinguish between them. This filtering does not detect haptics support or select a different output device: the most-recent-input-device rule still applies. It is an imperative authoring technique, not a proposed CSS device filter.

## Real-World Scenarios

The following examples demonstrate how declarative haptics and the imperative API together cover common use cases.

### E-Commerce "Add to Cart" Button

A tactile activation confirmation, with visual press styling:

```css
.add-to-cart {
  haptic: click align 0.8;
}

.add-to-cart:active {
  scale: 0.95;
}
```

### Photo Carousel

A horizontal photo carousel could provide a tactile tick when its pending snap target changes:

```css
.carousel {
  haptic: scrollsnapchanging tick 0.5;
}
```

`scrollsnapchanging` is not currently listed as a supported event source in [CSS Animation Triggers](https://drafts.csswg.org/animation-triggers-1/#event-trigger-source). This illustrative example requires adding it or allowing such DOM events generically, as well as defining which user-input-driven snap changes are eligible for haptics.

### Drag-to-Snap Divider

An app may want to use haptics when [dragging a sidebar divider to a snap point](https://github.com/w3c/webappswg/issues/138#issuecomment-3514522699). CSS can't express distance-threshold logic — this requires the imperative API:

```js
// Sticky activation was granted by the pointerdown that started the drag.
divider.addEventListener('pointermove', (e) => {
  const width = e.clientX;
  if (!snapped && Math.abs(width - SNAP_POINT) < 4) {
    snapped = true;
    navigator.playHaptics?.('align', 0.8);
  } else if (snapped && Math.abs(width - SNAP_POINT) >= 4) {
    snapped = false;
  }
});
```

## Future Extensions

The following extensions are out of scope for this initial proposal but represent natural next steps that could broaden the declarative surface.

### Additional predefined effects

The current set of predefined effects is intentionally small. If the effect vocabulary grows, the API should accommodate additional predefined effects without syntax changes.

### User preference media feature (`prefers-haptics`)

A coarse user-preference media feature could be considered in a future phase (for example, `prefers-haptics: reduce | no-preference`) to help authors adapt non-essential feedback. This is deferred from v1 to avoid expanding API surface before concrete implementation and privacy review feedback.

## Alternatives Considered

### CSS alternatives

Before selecting the [event-trigger model](#declarative-api-css), we considered the following five alternatives. Unlike event triggers, these models derive haptic requests from selector matching, computed-value changes, or transitions. The event-trigger model was selected for its explicit event causality, potential for lower-latency actuation, and reuse of CSS Animation Triggers infrastructure.

- **A. Nested `@haptic`** — the former primary proposal: an at-rule nested inside a style rule that fires when the parent selector starts matching. It co-locates haptics with visual styles, but selector matching can change for reasons unrelated to the intended interaction.
  ```css
  button:active { scale: 0.95; @haptic align 0.8; }
  ```
- **B. Standalone `@haptic-trigger`** — a top-level at-rule with a selector prelude; fires when the selector starts matching.
  ```css
  @haptic-trigger button:active { effect: align; intensity: 0.8; }
  ```
- **C. Computed-value property (`haptic-feedback`)** — a standard CSS property; fires when the computed value changes.
  ```css
  button:active { haptic-feedback: align 0.8; }
  ```
- **D. Named-effect property (`@haptic` + `haptic-name`)** — named effects defined in a top-level `@haptic` block, attached via `haptic-name`; fires when the computed `haptic-name` value changes. Despite resembling animation naming, this is a computed-value model, not the proposed event-trigger model.
  ```css
  @haptic bounce-land { effect: align; intensity: 0.8; }
  button:active { haptic-name: bounce-land; }
  ```
- **E. Transition-coupled** — haptic descriptors on the `transition-*` family; fires on `transitionend`.
  ```css
  .sidebar { transition: transform 300ms; transition-haptic-effect: align; }
  ```

#### Evaluation criteria

- **Syntax–semantics match** — Does the syntax read like what it does? Haptics are transient events, not ongoing state. A syntax that reads as state (e.g. `haptic-feedback: tick`) can mislead developers into expecting the value to "stay active."
- **Co-located** — Is the haptic declared in the same rule block as visual styles? Keeping related effects together improves readability.
- **Concise** — How much syntax for the common one-effect case?
- **Re-trigger safe** — Can the same effect fire from distinct state changes (e.g. `tick` on both check and uncheck of a toggle)? Computed-value models collapse to one value per element, so same-value transitions silently drop the haptic.

#### Comparison matrix

✅ = good &ensp; ⚠️ = limitation or workaround &ensp; ❌ = problematic

| Model | Syntax–semantics match | Co-located | Concise | Re-trigger safe |
|---|:---:|:---:|:---:|:---:|
| **A. Nested `@haptic`** | ✅ At-rule signals one-shot action | ✅ Same rule block | ✅ One-liner | ✅ Per-rule tracking |
| **B. Standalone `@haptic-trigger`** | ✅ At-rule signals one-shot action | ❌ Separate block | ❌ Two blocks | ✅ Per-rule tracking |
| **C. Computed-value property** | ❌ Property syntax implies ongoing state | ✅ Same rule block | ✅ Most concise | ❌ Same-value collision |
| **D. Named-effect property** | ⚠️ `haptic-name` reads as state, like C | ⚠️ Split (define + attach) | ❌ Define + attach | ⚠️ Workaround via distinct names |
| **E. Transition-coupled** | ⚠️ Natural with real transitions; synthetic 0ms hack without | ✅ Same rule block | ❌ Synthetic transitions | ⚠️ Needs distinct visual props |

### JavaScript alternatives

- **Extending `navigator.vibrate`** — The existing `vibrate()` API accepts raw duration/pattern arrays (e.g. `navigator.vibrate([100, 50, 200])`) with no way to express semantic intent like "tick" or "align." Adding named effects would require method overloading or a new options-bag signature, complicating an already-shipped interface. Feature detection becomes awkward — `typeof navigator.vibrate` tells you the method exists but not whether it supports named effects. The pattern-based model also encourages developers to hand-tune durations per device, which is the opposite of the platform-adaptive approach this proposal targets. Finally, `vibrate()` lacks broad engine support (absent in Safari/WebKit) and carries existing abuse stigma that could slow adoption of legitimate haptic use cases.
- **Pointer-event based API** ([previous explainer](../HapticsDevice/explainer.md)) — This was the team's earlier proposal and has the advantage of simplicity for pointer-driven interactions — events are familiar to developers and the mapping from input to haptic is explicit. However, it is purely imperative; it tightly couples haptics to specific input events, so common interactions like checkbox toggles and scroll-snap landings always require JavaScript. No declarative path, no cascade composition.

### Other design options

- **HTML attributes** (e.g. `<button haptic-on-activate="tick">`) — While HTML has adopted new attribute families (`aria-*`, `popover`, `inert`), `haptic-on-*` would require a separate attribute per interaction type (`haptic-on-activate`, `haptic-on-toggle`, `haptic-on-focus`, …), each needing dedicated spec text, parsing rules, and IDL definitions. More fundamentally, HTML attributes cannot compose with the cascade, media queries, or pseudo-class selectors — limiting expressiveness compared to a CSS-based approach.
- **Scroll-snap-specific surface** — The nested `@haptic` alternative could use the [`:snapped` pseudo-class](https://drafts.csswg.org/css-scroll-snap-2/#snapped), while a dedicated `scroll-snap-haptic` property could attach effects directly to a scroll container. The proposed event-trigger model instead aims to reuse scroll-snap events through the shared trigger infrastructure, subject to defining eligible event sources.

## Accessibility, Privacy, and Security Considerations

### Accessibility

Haptics are supplementary feedback and must not be the only way a user learns an essential state or outcome. A request may produce no sensation because compatible hardware is unavailable, the current input device cannot play it, or feedback is suppressed. Effects such as `success` and `error` should reinforce visible and programmatically exposed feedback, not replace it, and an interface must remain understandable and operable without haptics.

Users should be able to reduce or disable web-triggered haptics. User agents should respect relevant platform haptics and intensity settings where available and may use throttling, origin suppression, or browser controls to limit repeated or uncomfortable feedback. Authors should avoid excessive triggering and test that interactions remain usable without haptics.

### Privacy

To avoid introducing a new fingerprinting vector, the API does not expose means to query haptics-capable devices, available effects, or whether a haptic was successfully played. No new media features are introduced in v1.

### Security

**Anti-abuse:** Haptics produce no lasting effect — the user can navigate away at any time, immediately ending any haptic output, so the stakes are low. User agents also have several tools available to prevent abuse:

- **Throttling** — rate-limit haptic triggers on both APIs.
- **Origin suppression** — suppress haptics entirely for origins exhibiting abuse patterns.
- **User controls** — expose haptics settings at the global level (e.g., a browser-wide toggle) or per-origin (e.g., site-specific settings), allowing users to manage their haptics experience.

These controls should not be observable by web content to avoid introducing a new fingerprinting vector.

**Imperative API:** Requires sticky user activation. No permission gate.

**Declarative API:** Haptic requests must be causally associated with user input. Selector changes alone do not trigger haptics. CSS Animation Triggers determines which events can fire triggers; haptics-specific activation checks, including treatment of script-generated events, remain to be defined.

**Permissions Policy:** The Web Haptics API is controlled by a [permissions policy](https://w3c.github.io/webappsec-permissions-policy/) with a default allowlist of `"self"`. This means:

- Same-origin iframes inherit haptics access from the top frame.
- Cross-origin iframes are denied haptics by default. The embedder must explicitly delegate access:
  ```html
  <iframe src="https://cross-origin.com/widget" allow="haptics"></iframe>
  ```
- If the permissions policy for haptics is disabled in a given context, imperative `navigator.playHaptics()` calls are silently ignored and declarative `haptic` mappings do not produce haptic output.

## Open Questions

- **Should any [future extension](#future-extensions) be promoted to v1?** Additional predefined effects and other deferred extensions are listed above. If any is critical for initial adoption, we would like to hear which and why.
- **What haptics-specific activation checks should apply to event triggers?** The proposal limits haptics to trigger activations causally associated with user input. How should this requirement interact with Animation Triggers' event eligibility rules, script-generated events, and sticky user activation?
- **Should a user-preference media feature be added in a future phase?** v1 introduces no media query. A possible extension is a coarse preference signal (e.g. `prefers-haptics: reduce | no-preference`) if there is concrete use-case and privacy review support.
- **Which additional event sources should CSS Animation Triggers support for haptics?** Scroll-snap events such as `scrollsnapchanging` are not currently supported trigger sources. Their integration and eligibility rules need to be defined to cover the [photo carousel scenario](#photo-carousel).
- **Feedback on the predefined effect vocabulary?** The current set (`hint`, `edge`, `tick`, `align`, `success`, `error`) is intentionally small. Feedback is needed on whether these effects cover the most common interaction patterns and map well to native haptic primitives across platforms.
- **Should the API return whether haptics was successfully played?** Currently, `playHaptics` always returns `undefined` to avoid exposing device capabilities. Returning a boolean or promise could help developers debug, but risks leaking hardware information.
- **Should global arbitration be standardized beyond same-element resolution?** The proposal resolves simultaneous mappings on one element by list order and suppresses further requests to a target device within the same frame. We welcome feedback on whether a future level should define a deterministic cross-element winner algorithm.
- **Is there developer interest in haptics device enumeration?** Though out of scope, we would like to understand interest. Exposing available devices or capabilities would enable richer experiences but introduces fingerprinting trade-offs that need careful evaluation.

## Reference for Relevant Haptics APIs

This section provides reference to existing web and native haptics APIs to help inform the API design and platform supportability.

Known platform-specific native haptics APIs:

- [Windows: InputHapticsManager](https://learn.microsoft.com/en-us/uwp/api/windows.devices.haptics?view=winrt-26100)
- [macOS: NSHapticFeedbackManager](https://developer.apple.com/documentation/appkit/nshapticfeedbackmanager)
- [iOS: Core Haptics](https://developer.apple.com/documentation/corehaptics)
- [Android: VibrationEffect](https://developer.android.com/reference/android/os/VibrationEffect)

Relevant web APIs:
- [navigator.vibrate](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/vibrate)

Relevant CSS specifications:
- [CSS Transitions](https://drafts.csswg.org/css-transitions-1/)
- [CSS Animations](https://drafts.csswg.org/css-animations-1/)
- [CSS Animation Triggers](https://drafts.csswg.org/animation-triggers-1/)
- [CSS Scroll Snap](https://drafts.csswg.org/css-scroll-snap-1/)
- [CSS Scroll Snap 2](https://drafts.csswg.org/css-scroll-snap-2/)

## Stakeholder Feedback / Opposition

We have heard some early developer interest such as [dragging divider to a snap point in Slack](https://github.com/w3c/webappswg/issues/138#issuecomment-3514522699).

At TPAC 2026, the [CSS Working Group resolved to create an editor's draft using the animation-trigger model](https://github.com/w3c/csswg-drafts/issues/13728#issuecomment-6493033586) and pursue focused issues based on that draft.

We intend to seek feedback via:

- Incubation in WICG.
- Discuss within Device & Sensors Working Group.
- Cross‑share with Haptic Industry Forum (non‑standards venue) to align on primitives vocabulary and invite suppliers/OEMs to comment publicly in WICG issues.
- Continue working with the CSS Working Group on the event-trigger model and focused integration issues.

## References & Acknowledgements

We acknowledge that this design will change and improve through input from browser vendors, standards bodies, accessibility advocates, and developers. Ongoing collaboration is essential to ensure the API meets diverse needs.

We only get here through the contributions of many — thank you to everyone who shares feedback and helps shape this work. Special thanks to:
- Ross Nichols – Contributions to Windows Haptics API design and integration guidance.
- Previous Iteration – [HapticsDevice Explainer (Microsoft Edge)](../HapticsDevice/explainer.md), which served as the foundation for this proposal.