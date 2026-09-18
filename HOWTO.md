# How to experiment with the Web Haptics API in Microsoft Edge

The imperative Web Haptics API, `navigator.playHaptics()`, is available
experimentally in Microsoft Edge on Windows, and on other Chromium-based
browsers. Availability and enablement may vary by browser.

The API and these instructions are subject to change as the proposal develops.

The declarative CSS `@haptic` API described in the [Web Haptics
explainer](readme.md#declarative-api-css) is not implemented yet.

## Requirements

- Windows 11, version 24H2 or later.
- A haptics-capable mouse, touchpad, or pen.
- Microsoft Edge 156.0.4284.0 or later.
  - (Earliest builds can be accessed in Edge Canary)

Edge sends haptic feedback only to the most recent input device. Use the
haptics-capable device to interact with the test page. Edge does not reroute the
request if the most recent device does not support haptics.

## Enable the API

There is no dedicated `edge://flags` entry for Web Haptics. Enable it with
this Blink command-line parameter:

```text
--enable-blink-features=WebHaptics
```

Close all Microsoft Edge windows, add the parameter to your preferred Edge
launch method, and restart Edge. Open `edge://version` and confirm that the
**Command Line** field contains `--enable-blink-features=WebHaptics`.

Alternatively, open `edge://flags`, enable **Experimental Web Platform
features**, and restart Edge. This enables Web Haptics along with other
experimental features, which may introduce additional instability. Use the
command-line flag above to enable only Web Haptics.

## Try the API

Save the following example as an HTML file and open it in the Edge instance
launched above:

```html
<!doctype html>
<html lang="en">
  <meta charset="utf-8">
  <title>Web Haptics test</title>
  <button id="play-haptic">Play a tick</button>
  <p id="status"></p>
  <script>
    const button = document.querySelector("#play-haptic");
    const status = document.querySelector("#status");

    if (typeof navigator.playHaptics !== "function") {
      button.disabled = true;
      status.textContent = "Web Haptics is not available.";
    } else {
      button.addEventListener("click", () => {
        navigator.playHaptics("tick", 0.7);
      });
    }
  </script>
</html>
```

The explainer may include proposed effects that are not yet implemented for the
DevTrial. The current implementation accepts one of the effects `"hint"`,
`"edge"`, `"tick"`, or `"align"`. The optional intensity is a number from `0.0`
to `1.0` and defaults to `1.0`. See the [effect
vocabulary](readme.md#effect-vocabulary) for the full proposed vocabulary and
guidance on choosing an effect.

The API requires sticky user activation, so call it after a user interaction
such as the button click above. The call returns `undefined` and intentionally
does not report whether the device played the effect.

## Cross-origin frames

The API is controlled by the `"haptics"` permissions policy and is available to
same-origin frames by default. A top-level page must explicitly delegate access
to a cross-origin frame:

```html
<iframe src="https://example.com" allow="haptics"></iframe>
```

## Troubleshooting

- If `navigator.playHaptics` is missing, check the Edge version and the
  command line shown in `edge://version`.
- If the API is present but no haptic plays, confirm that Windows is version
  24H2 or later and use the haptics-capable device for the interaction.
- The API silently ignores calls without sticky user activation, calls blocked
  by permissions policy, and calls for a most recent input device that is not
  haptics-capable.

For the full API proposal, see the [Web Haptics
explainer](readme.md#proposed-approach). For Chromium implementation details,
see the [Web Haptics API design
document](https://github.com/kyerebo/chromium-design-docs/blob/main/web-haptics/web-haptics-design-doc.md).
