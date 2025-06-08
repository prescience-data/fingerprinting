---
title: window.ShadowRoot
layout: default
---
# window.ShadowRoot
## Purpose
The `ShadowRoot` interface is the root node of a DOM subtree that is rendered separately from a document's main DOM tree. It is a key part of the Web Components standard, used to encapsulate the internal structure, style, and behavior of a custom element.

## Fingerprinting Usage
The primary use of `window.ShadowRoot` in bot detection is not to gather high-entropy data, but to perform integrity checks on the browser environment. Automation frameworks (like Puppeteer, Playwright) and browser-spoofing tools sometimes modify or hook native browser APIs to hide their presence. `ShadowRoot` is a common target for these checks because it is a modern, non-trivial API that is difficult to emulate perfectly.

1.  **Native Code Check:** This is the most common and effective test. In unmodified browsers, calling `toString()` on a native function implemented by the engine (like the `ShadowRoot` constructor) returns a string containing `"[native code]"`. If an automation script overrides this function in JavaScript to control its behavior, the `toString()` result will reveal the source code of the JavaScript function instead. This is a strong signal of a manipulated environment.

2.  **API Consistency:** A detection script can verify the presence and properties of `ShadowRoot` in conjunction with related APIs. For example, it can check for the existence of both `window.ShadowRoot` and `Element.prototype.attachShadow`. If one exists without the other, or if one has been modified while the other hasn't, it indicates an inconsistent and likely spoofed environment.

3.  **Error Message Analysis:** The specific text and properties of `DOMException` errors thrown when misusing the Shadow DOM API can be a source of entropy. For instance, attempting to attach a shadow root to an element that doesn't support it (e.g., `<img>`) will throw an error. The exact error message can differ between browser engines (V8, SpiderMonkey, JavaScriptCore) and, more importantly, between a real browser and a less sophisticated bot or emulator that tries to fake the API.

## Sample Code
This snippet demonstrates the "Native Code Check" to detect if the `ShadowRoot` constructor has been tampered with.

```javascript
function isShadowRootTampered() {
  // Check if the API exists. Its absence is a signal for very old or non-standard clients.
  if (typeof window.ShadowRoot !== 'function') {
    return true; // Or handle as a specific fingerprinting signal
  }

  // The core check: Native browser functions will include "[native code]" in their string representation.
  // Overridden functions in JavaScript will not.
  try {
    const isNative = /\[native code\]/.test(window.ShadowRoot.toString());
    return !isNative;
  } catch (e) {
    // An error during the check is itself a strong signal of a strange environment.
    return true;
  }
}

const tamperingDetected = isShadowRootTampered();

if (tamperingDetected) {
  console.log("High confidence: Browser environment has been manipulated. Potential bot.");
  // This client would be flagged for further scrutiny or blocking.
} else {
  console.log("Low confidence: ShadowRoot API appears to be native and unmodified.");
}
```