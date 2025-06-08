---
title: window.getScreenDetails
layout: default
---
# window.getScreenDetails
## Purpose
Part of the Window Management API, `window.getScreenDetails()` is an asynchronous method that allows a web application to get detailed properties of all screens connected to the user's device. It is designed to enable rich multi-window experiences, such as placing content on a specific secondary monitor.

## Fingerprinting Usage
`window.getScreenDetails` is a high-entropy source for fingerprinting and a powerful tool for bot detection due to its detailed, hardware-specific output and permission-based nature.

1.  **High-Entropy Hardware Enumeration:** Unlike the legacy `window.screen` object which only describes the screen the current window is on, `getScreenDetails` provides an array of all connected displays. The number of screens, their individual resolutions (`width`, `height`), color depths (`colorDepth`, `pixelDepth`), available workspace (`availWidth`, `availHeight`), and hardware labels (`label`, e.g., "DELL U2721DE") create a highly unique hardware signature.

2.  **Discrepancy Detection:** This is a primary anti-bot technique. Automation tools like Puppeteer or Playwright often patch the legacy `window.screen` object to spoof a common resolution. However, they may fail to consistently patch the newer, more complex `getScreenDetails` API. An anti-bot script can call both APIs and check for inconsistencies between `window.screen` and the primary screen reported by `getScreenDetails`. A mismatch is a strong signal of environment spoofing.

3.  **Behavioral Signals & Permission Gating:** The API requires explicit user permission via a prompt. This creates several detection vectors:
    *   **Permission State:** A script can check the permission status (`navigator.permissions.query({ name: 'window-management' })`). If a bot's environment is configured to auto-deny, this is a detectable state.
    *   **Interaction Timing:** The time between the API call and the promise resolution (either fulfilled or rejected) can be measured. A real user takes time to read and click the prompt, whereas automation might grant or deny permission instantly.
    *   **Error Handling:** A bot script might not correctly handle the `NotAllowedError` promise rejection if permission is denied, leading to unhandled exceptions that can be detected.

4.  **Headless Environment Anomaly:** Headless browsers typically run in a virtualized environment with a single, virtual display. The output of `getScreenDetails` in such an environment will be simple and generic (e.g., one screen, common resolution, no label). This contrasts sharply with the complex multi-monitor setups of many power users, making it easy to segment traffic. The `isInternal` property can also reveal anomalies if a bot claims to be a laptop but its display is not marked as internal.

## Sample Code
This snippet demonstrates a discrepancy check between the legacy `window.screen` and the modern `getScreenDetails` API, a common anti-bot technique.

```javascript
async function analyzeScreenConfiguration() {
  // Check for API support first.
  if (!window.getScreenDetails) {
    console.log("Window Management API not supported.");
    return;
  }

  try {
    // This call triggers a user permission prompt.
    const screenDetails = await window.getScreenDetails();

    // Find the primary display from the detailed list.
    const primaryScreen = screenDetails.screens.find(s => s.isPrimary);

    if (!primaryScreen) {
      console.log("No primary screen found in getScreenDetails().");
      return;
    }

    // --- FINGERPRINTING LOGIC ---
    // Compare legacy screen properties with the modern API's primary screen.
    const widthMismatch = window.screen.width !== primaryScreen.width;
    const heightMismatch = window.screen.height !== primaryScreen.height;

    if (widthMismatch || heightMismatch) {
      console.warn("SPOOFING DETECTED: Discrepancy found between window.screen and getScreenDetails().");
      console.log(`Legacy screen: ${window.screen.width}x${window.screen.height}`);
      console.log(`Primary screen (getScreenDetails): ${primaryScreen.width}x${primaryScreen.height}`);
    } else {
      console.log("Screen details are consistent.");
    }

    // Additional entropy: Log the number of screens and their labels.
    console.log(`Total screens detected: ${screenDetails.screens.length}`);
    screenDetails.screens.forEach((s, i) => {
      console.log(`Screen ${i}: Label='${s.label}', Res=${s.width}x${s.height}, Internal=${s.isInternal}`);
    });

  } catch (err) {
    // The error itself is a signal (e.g., user denied permission).
    if (err.name === 'NotAllowedError') {
      console.log("Fingerprint signal: User denied window management permission.");
    } else {
      console.error("An unexpected error occurred:", err);
    }
  }
}

// Run the analysis.
analyzeScreenConfiguration();
```