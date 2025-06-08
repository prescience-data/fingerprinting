---
title: window.PluginArray
layout: default
---
# window.PluginArray
## Purpose
Provides an array-like list (`PluginArray`) of `Plugin` objects, each representing a browser plugin (e.g., Adobe Flash, Java) installed in the browser. This API is a remnant of the deprecated NPAPI (Netscape Plugin API) architecture and is effectively obsolete in modern browsers like Chrome (support removed in v88).

## Fingerprinting Usage
The relevance of `window.PluginArray` (accessed via `navigator.plugins`) has shifted from an entropy source to a powerful anomaly detector.

*   **Legacy Entropy Source:** Historically, the list of installed plugins, including their specific versions, names, and descriptions, was a high-entropy fingerprinting vector. The unique combination of plugins on a user's system was highly identifying.

*   **Modern Anomaly Detection:** In current versions of Chrome, `navigator.plugins` is an empty `PluginArray`. Its properties are now used to detect non-standard browser environments:
    1.  **Presence of Plugins:** The most significant signal. If `navigator.plugins.length` is greater than `0`, it's a major red flag. This indicates either a very old, insecure browser or, more commonly, an automation framework (like Puppeteer or Playwright) that is attempting to spoof a legacy environment to appear more "human." Even internal components like Chrome's PDF viewer are not exposed here, so any reported plugin is suspicious.
    2.  **Object Type Inconsistency:** Sophisticated bot detection scripts verify that `navigator.plugins` is a genuine `PluginArray` object, not a standard JavaScript `Array`. A naive bot might replace `navigator.plugins` with a simple array `[]`. This can be detected by checking `Object.prototype.toString.call(navigator.plugins)`, which should return `"[object PluginArray]"`. Anything else (like `"[object Array]"`) indicates tampering.
    3.  **Property Spoofing:** If a bot does report plugins, the properties of the `Plugin` objects (`name`, `description`, `filename`) are often generic or inconsistent, making them easy to flag as fake.

## Sample Code
This snippet demonstrates how to check for anomalies related to the `PluginArray` object, which is a common technique in modern bot detection.

```javascript
function analyzePlugins() {
  const result = {
    pluginCount: 0,
    isTampered: false,
    reportedPlugins: [],
  };

  try {
    // 1. Check if the object type is the expected native PluginArray.
    // Naive bots often replace it with a standard JavaScript Array.
    if (Object.prototype.toString.call(navigator.plugins) !== '[object PluginArray]') {
      result.isTampered = true;
    }

    result.pluginCount = navigator.plugins.length;

    // 2. In modern Chrome, the plugin count should be 0.
    // A non-zero count is a strong signal of a bot or outdated browser.
    if (result.pluginCount > 0) {
      for (let i = 0; i < navigator.plugins.length; i++) {
        const plugin = navigator.plugins[i];
        result.reportedPlugins.push({
          name: plugin.name,
          description: plugin.description,
          filename: plugin.filename,
        });
      }
    }
  } catch (e) {
    // Accessing or iterating might throw an error in a heavily modified environment.
    result.isTampered = true;
  }

  // A high-risk score would be assigned if pluginCount > 0 or isTampered is true.
  return result;
}

const pluginFingerprint = analyzePlugins();
console.log(pluginFingerprint);
// Expected output in modern Chrome:
// { pluginCount: 0, isTampered: false, reportedPlugins: [] }
```