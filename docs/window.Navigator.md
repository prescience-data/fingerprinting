---
title: window.navigator
layout: default
---
# window.Navigator
## Purpose
The `Navigator` object provides a global interface containing information about the browser's identity, state, and the user's environment. It acts as a primary entry point for querying browser capabilities, hardware specifications, and user settings.

## Fingerprinting Usage
The `navigator` object is a cornerstone of browser fingerprinting due to the high-entropy data it exposes. Anti-bot systems heavily scrutinize its properties for signs of automation or spoofing.

*   **Direct Automation Flag:** `navigator.webdriver` is a boolean property that is explicitly set to `true` when the browser is controlled by an automation tool like Selenium or Puppeteer. This is a primary and direct signal for bot detection. Sophisticated bots must use patched browser builds or specific launch arguments (`--disable-blink-features=AutomationControlled`) to hide this flag.

*   **Classic Identifiers & Inconsistencies:** The combination of `navigator.userAgent`, `navigator.platform`, `navigator.vendor`, and `navigator.language`/`languages` provides a foundational fingerprint. While these values are easily spoofed, anti-bot systems focus on inconsistencies. For example, a `userAgent` string indicating Windows (`Win64`) while `navigator.platform` returns `Linux x86_64` is a strong indicator of a poorly configured bot.

*   **Hardware-Level Entropy:** Properties like `navigator.hardwareConcurrency` (number of logical CPU cores) and `navigator.deviceMemory` (approximate device RAM) are highly effective for fingerprinting. These values are stable for a given device and difficult to spoof believably. Bots often run in virtualized environments with common, low-resource configurations (e.g., 2 or 4 cores), making atypical or default values suspicious.

*   **Modern Client Hints (`userAgentData`):** The `navigator.userAgentData` API is the modern replacement for the `userAgent` string. While intended to reduce passive fingerprinting, its `getHighEntropyValues()` method can be used to actively query specific, high-entropy details like platform version, architecture (`x86`, `arm`), and the full browser version list. The specific set of "brands" and their versions provides a rich, structured fingerprint.

*   **Plugin and MIME Type Enumeration:** `navigator.plugins` and `navigator.mimeTypes`, while less relevant since the deprecation of Flash, still contribute to entropy. The list of installed plugins (e.g., PDF Viewer) and their properties (name, filename, description) is unique to a user's setup. Headless browsers typically return empty lists for both, which is a significant signal in itself.

## Sample Code
This snippet demonstrates collecting several key properties from the `navigator` object, including a modern asynchronous call to `userAgentData`, to build a fingerprint profile.

```javascript
async function getNavigatorFingerprint() {
  const fingerprint = {};

  try {
    // Direct bot detection flag
    fingerprint.isAutomated = navigator.webdriver;

    // Hardware characteristics
    fingerprint.cpuCores = navigator.hardwareConcurrency;
    fingerprint.deviceMemory = navigator.deviceMemory;

    // Classic identifiers
    fingerprint.platform = navigator.platform;
    fingerprint.languages = navigator.languages;
    
    // Modern, high-entropy client hints
    if (navigator.userAgentData) {
      const highEntropyValues = await navigator.userAgentData.getHighEntropyValues([
        "platform",
        "platformVersion",
        "architecture",
        "model",
        "uaFullVersion"
      ]);
      fingerprint.clientHints = highEntropyValues;
    }

    // Plugin data
    fingerprint.pluginCount = navigator.plugins.length;
    fingerprint.plugins = Array.from(navigator.plugins).map(p => p.name);

  } catch (error) {
    fingerprint.error = error.message;
  }

  return fingerprint;
}

getNavigatorFingerprint().then(fp => {
  console.log("Navigator Fingerprint Profile:");
  console.log(JSON.stringify(fp, null, 2));
});
```