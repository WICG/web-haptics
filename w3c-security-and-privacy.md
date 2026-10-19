# Web Haptics JavaScript API: Security and Privacy Questionnaire

This document answers the [W3C Self-Review Questionnaire: Security and
Privacy](https://www.w3.org/TR/security-privacy-questionnaire/) for the
imperative Web Haptics JavaScript API:

```js
navigator.playHaptics(effect, intensity);
```

It covers the JavaScript API described in the [Web Haptics
explainer](./readme.md) and the current Chromium implementation. The proposed
declarative CSS API is out of scope.

## 1. What information does this feature expose, and for what purposes?

1. **New information exposed to a first party:** The presence of
   `navigator.playHaptics` reveals that the user agent implements the API
   surface, but does not imply that the platform has a functional backend. The
   API does not directly expose connected devices, device identity, haptic
   capabilities, supported effects, the selected target, or whether an effect
   played successfully.
2. **New information exposed to a third party:** A third-party frame sees the
   same API surface, but can use it only when the embedding page delegates the
   `haptics` permissions policy. It receives no additional device information.

This API does not return any success or supported device signals on call.
Its bandwidth is bounded by the fixed vocabulary, lack of a playback result,
permissions on the observing API, and any rate limiting or suppression
 a user agent chooses to apply.

## 2. Do features in your specification expose the minimum amount of information necessary to implement the intended functionality?

Yes. The intended functionality requires only a way to request a semantic
effect at an intensity. The API deliberately omits device enumeration,
capability discovery, raw waveform identifiers, playback status, and a
completion callback.

## 3. Do the features in your specification expose personal information, personally-identifiable information (PII), or information derived from either?

No. The API does not expose personal information, PII, or information derived
from either.

## 4. How do the features in your specification deal with sensitive information?

The API does not accept, derive, or return sensitive information. It accepts
only a closed effect name and a numeric intensity.

## 5. Does data exposed by your specification carry related but distinct information that may not be obvious to users?

No data is returned to the page.

## 6. Do the features in your specification introduce state that persists across browsing sessions?

No. The API creates no state, cache, or storage.

A user agent may provide global or per-site haptics controls, but such controls
are user-agent settings rather than state created or exposed by this API. Their
effect should not be observable through the API.

## 7. Do the features in your specification expose information about the underlying platform to origins?

The existence of `navigator.playHaptics` exposes support for the API surface in
the user agent, but does not imply that the platform has a functional backend.
The API does not directly expose whether the platform has haptic hardware,
which device is current, which effects are natively supported, how an effect is
mapped, or whether platform policy suppressed playback.

The method returns `undefined` without a success-dependent response.

## 8. Does this specification allow an origin to send data to the underlying platform?

Yes, in a tightly constrained form. An origin can request:

- one value from a closed semantic effect vocabulary; and
- a normalized intensity between `0.0` and `1.0`.

It cannot send raw waveform data, arbitrary durations, device identifiers,
URLs, files, or platform-specific commands.

In Chromium, the renderer clamps intensity, and the browser process
independently validates that the value is finite and within range before
calling the platform backend. The effect is transported as a closed Mojo enum.
An invalid intensity from a compromised renderer is treated as bad IPC and
terminates that renderer rather than being clamped in the browser process. A
compromised renderer cannot use this interface to send arbitrary platform data.

## 9. Do features in this specification enable access to device sensors?

No sensor access is provided. The API produces output through a haptic
actuator and does not expose sensor readings or input-device events.

## 10. Do features in this specification enable new script execution/loading mechanisms?

No.

## 11. Do features in this specification allow an origin to access other devices?

The API allows bounded output to the haptic actuator of the most recent input
device. This actuator is a shared physical resource and is not scoped to a
particular page. The API does not provide general device access:

- the origin cannot enumerate, select, pair with, or identify devices;
- the origin cannot read data from a device;
- the origin cannot address another connected device; and
- the origin cannot determine whether output was delivered.

If the most recent input device is not haptics-capable, no haptic is played and
the request is not rerouted.

## 12. Do features in this specification allow an origin some measure of control over a user agent's native UI?

No. The API does not change browser chrome, permission UI, security indicators,
window state, or operating-system UI.

## 13. What temporary identifiers do the features in this specification create or expose to the web?

None. The API creates and exposes no temporary or persistent identifiers.

## 14. How does this specification distinguish between behavior in first-party and third-party contexts?

Web Haptics is controlled by the `haptics` permissions policy with a default
allowlist of `self`.

- A top-level document and same-origin child frames are enabled by default.
- A cross-origin child frame is disabled unless the embedding document
  explicitly delegates `haptics`.
- A disabled context receives no hardware or device details, and playback is
  suppressed.

Chromium also suppresses the API in fenced frames.

The document's permissions-policy state is intentionally observable through
the permissions-policy APIs. A denied use may also generate a console message
or a permissions-policy violation report. These diagnostics describe the
document's policy configuration, not haptic hardware.

## 15. How do the features in this specification work in the context of a browser's Private Browsing or Incognito mode?

The API requires no special private-browsing behavior.

## 16. Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

Yes. The explainer has separate [Privacy](./readme.md#privacy) and
[Security](./readme.md#security) subsections under its Accessibility, Privacy,
and Security Considerations section.

## 17. Do features in your specification enable origins to downgrade default security protections?

No. The API does not weaken existing security protections.

## 18. What happens when a document that uses your feature is kept alive in BFCache (instead of getting destroyed) after navigation, and potentially gets reused on future navigations back to the document?

A document in BFCache cannot trigger haptics. After restoration, Chromium
requires a new user gesture before playback. The API has no events or results
to queue while the document is cached.

## 19. What happens when a document that uses your feature gets disconnected?

Calls from a disconnected or destroyed context do not play haptics.

There are no delayed pattern tasks or completion callbacks to retain after
disconnection. A short waveform already dispatched to the platform may finish,
but no further waveform is scheduled by the API.

## 20. Does your spec define when and how new kinds of errors should be raised?

No new errors are defined. Hardware or playback failures are intentionally
silent no-ops that return `undefined`, preventing them from exposing device
capabilities or state. Normal Web IDL errors for invalid arguments still apply.

## 21. Does your feature allow sites to learn about the user's use of assistive technology?

No. The API does not expose whether assistive technology is being used.

## 22. What should this questionnaire have asked?

Can a site learn about a device's haptic capabilities from a playback request?

The API exposes no device information or playback result, and unsupported
requests silently do nothing.

## References

- [Web Haptics explainer](./readme.md)
- [W3C Self-Review Questionnaire: Security and
  Privacy](https://www.w3.org/TR/security-privacy-questionnaire/)
- [Chromium Web Haptics WebIDL](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/modules/haptics/navigator_haptics.idl)
- [Chromium renderer controller](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/modules/haptics/haptics_controller.cc)
- [Chromium browser security boundary](https://source.chromium.org/chromium/chromium/src/+/main:content/browser/haptics/haptics_service_impl.cc)
- [Chromium Windows backend](https://source.chromium.org/chromium/chromium/src/+/main:content/browser/haptics/haptics_manager_impl_win.cc)
