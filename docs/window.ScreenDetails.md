---
title: window.ScreenDetails
layout: default
---
# window.ScreenDetails
## Purpose
The `ScreenDetails` API, part of the Window Management API, provides web applications with detailed information about all screens connected to a user's device, not just the screen the browser window currently occupies. This is designed for multi-monitor applications, allowing them to place windows and content on specific displays.

## Fingerprinting Usage
`ScreenDetails` is a high-entropy source for fingerprinting and a powerful tool for bot detection due to the richness and complexity of the data it exposes.

1.  **High-Entropy Configuration:** The combination of the number of screens, their individual resolutions (`width`, `height`), available space (`availWidth`, `availHeight`), color/pixel depths, device pixel ratios, and relative positions (`left`, `top`) creates a highly unique signature. A dual-monitor setup with one 4K and one 1080p screen in a specific arrangement is far more identifiable than a single screen's resolution.

2.  **Bot Environment Anomaly Detection:**
    *   **Default State:** Most headless browsers (Puppeteer, Playwright) and bot frameworks run in a default, single-screen emulated environment. The presence and specific values from `ScreenDetails` can immediately differentiate a default bot from a real user with a multi-monitor setup.
    *   **Inconsistent Spoofing:** A bot attempting to spoof a multi-monitor setup can be detected by inconsistencies. For example, if `window.screen` reports properties that do not match any of the screens listed in the `ScreenDetails.screens` array, it's a strong signal of manipulation. Another check is comparing the browser's `window.innerWidth`/`innerHeight` against the `availWidth`/`availHeight` of the screen it claims to be on.
    *   **Unrealistic Values:** Bots may generate unrealistic hardware configurations, such as screens with non-standard resolutions, impossible `devicePixelRatio` values for a given resolution, or screen layouts that physically overlap in nonsensical ways. The `isInternal` flag (e.g., for a laptop's built-in display) is another detail that bots often fail to spoof correctly.

3.  **Behavioral Signals:** The API fires a `screenschange` event when a monitor is connected or disconnected. A fingerprinting script can listen for this event. While its absence isn't definitive, its presence is a strong liveness signal that is very difficult for a bot to fake convincingly. Furthermore, tracking which screen the window is currently on (`ScreenDetails.currentScreen`) can reveal user behavior (moving windows between monitors) that bots do not emulate.

## Sample Code
This snippet demonstrates how to asynchronously gather screen details to generate a fingerprinting hash. It handles the necessary permission query and aggregates data from all available screens.

```javascript
async function getMultiScreenFingerprint() {
  // Check if the advanced API is available
  if (!window.getScreenDetails) {
    // Fallback to the legacy window.screen for basic info
    return {
      screenCount: 1,
      screens: [{
        width: window.screen.width,
        height: window.screen.height,
        devicePixelRatio: window.devicePixelRatio,
        colorDepth: window.screen.colorDepth,
      }]
    };
  }

  try {
    // The API requires a permission grant for "window-management"
    const screenDetails = await window.getScreenDetails();

    const fingerprintData = {
      screenCount: screenDetails.screens.length,
      // Map over each screen to collect its unique properties
      screens: screenDetails.screens.map(screen => ({
        width: screen.width,
        height: screen.height,
        availWidth: screen.availWidth,
        availHeight: screen.availHeight,
        devicePixelRatio: screen.devicePixelRatio,
        colorDepth: screen.colorDepth,
        isPrimary: screen.isPrimary,
        isInternal: screen.isInternal,
        // Relative position provides layout information
        left: screen.left,
        top: screen.top,
      })),
      // Also capture the current screen for consistency checks
      currentScreenIndex: screenDetails.screens.indexOf(screenDetails.currentScreen)
    };

    // The resulting JSON can be stringified and hashed for a stable fingerprint
    return JSON.stringify(fingerprintData);

  } catch (err) {
    // User may have denied permission, or another error occurred.
    console.error("Could not get screen details:", err);
    return null;
  }
}

// Usage:
getMultiScreenFingerprint().then(fingerprint => {
  if (fingerprint) {
    console.log("Screen Fingerprint:", fingerprint);
    // Example output for a dual-monitor setup:
    // "{"screenCount":2,"screens":[{"width":2560,"height":1440,... "isPrimary":true},{"width":1920,"height":1080,... "left":2560}],"currentScreenIndex":0}"
  }
});
```