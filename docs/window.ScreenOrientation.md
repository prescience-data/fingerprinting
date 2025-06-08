---
title: window.ScreenOrientation
layout: default
---
# window.ScreenOrientation
## Purpose
The Screen Orientation API provides information about the current orientation of the viewport (e.g., portrait or landscape) and allows scripts to be notified when the orientation changes. It is primarily designed for mobile devices and tablets.

## Fingerprinting Usage
The `ScreenOrientation` API is a valuable tool for bot detection due to its environmental and behavioral signals. Its utility comes less from unique entropy and more from detecting inconsistencies and anomalies common in automated environments.

1.  **Consistency Check:** This is the most powerful use case. Anti-bot systems cross-reference the reported orientation `type` with the screen dimensions (`screen.width` and `screen.height`). For example, if `screen.width` is greater than `screen.height`, the device is physically in a landscape layout. If `screen.orientation.type` reports `portrait-primary`, it's a strong signal of a spoofed environment. Headless browsers like Puppeteer or Playwright often fail this check, as they may spoof a mobile user-agent (implying portrait) while running in a default server environment with a landscape-like resolution.

2.  **Behavioral Analysis:** The `onchange` event is a high-quality human interaction signal. A real user on a mobile device might rotate their screen during a session, which fires this event. Bots almost never simulate this behavior. A script can listen for this event, and its occurrence can significantly increase the trust score of a session claiming to be from a mobile device. The absence of this event over a long session on a declared "mobile" client is a weak negative signal.

3.  **Environment Anomaly:** The mere presence and values of the `screen.orientation` object can be an anomaly. A browser identifying as a desktop (e.g., via `navigator.platform`) but reporting a `portrait-primary` orientation is highly suspicious, as desktops are almost universally `landscape-primary`. Furthermore, some automation frameworks may return `null` or default values for `type` and `angle`, which differ from the typical values (`portrait-primary`, `0`) found on a real mobile device upon page load.

4.  **Minor Entropy Source:** The initial values of `type` (e.g., `portrait-primary`, `landscape-primary`) and `angle` (0, 90, 180, 270) contribute a few bits to a device's overall fingerprint, helping to differentiate between device configurations.

## Sample Code
This snippet demonstrates the core consistency check between screen dimensions and the reported orientation type.

```javascript
function checkOrientationConsistency() {
  const result = {
    consistent: true,
    reason: 'N/A',
    type: 'unsupported',
  };

  if (!window.screen || !window.screen.orientation) {
    result.consistent = false;
    result.reason = 'ScreenOrientation API not supported.';
    return result;
  }

  const { type } = window.screen.orientation;
  const { width, height } = window.screen;
  
  result.type = type;

  // Check if screen dimensions (physical layout) match the reported orientation
  const isPhysicallyLandscape = width > height;
  const isReportedAsLandscape = type.includes('landscape');

  if (isPhysicallyLandscape !== isReportedAsLandscape) {
    result.consistent = false;
    result.reason = `Inconsistency detected: Screen dimensions (${width}x${height}) suggest a ${isPhysicallyLandscape ? 'landscape' : 'portrait'} layout, but orientation API reports '${type}'.`;
  }

  return result;
}

const orientationCheck = checkOrientationConsistency();

if (!orientationCheck.consistent) {
  // This is a strong signal of a misconfigured bot or spoofed environment.
  console.warn('Potential bot detected:', orientationCheck.reason);
} else {
  console.log('Orientation consistency check passed.');
}
```