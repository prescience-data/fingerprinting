---
title: window.cancelAnimationFrame
layout: default
---
# window.cancelAnimationFrame
## Purpose
Cancels an animation frame request previously scheduled with `window.requestAnimationFrame()`. This prevents the specified callback function from being executed before the next browser repaint.

## Fingerprinting Usage
`window.cancelAnimationFrame` (cAF) is not a primary source of entropy itself. Its value in fingerprinting and bot detection lies almost exclusively in its relationship with `window.requestAnimationFrame` (rAF) and its role in verifying browser environment integrity.

1.  **Integrity and Consistency Checks:** Anti-bot systems use `cAF` to validate the authenticity of the `rAF` implementation. Sophisticated bots often hook or modify `rAF` to control timing, prevent detection via refresh rate, or run in a detached environment. However, they may fail to correctly spoof the corresponding `cAF` function. Scripts can check:
    *   **Native Code String:** A call to `window.cancelAnimationFrame.toString()` on an unmodified browser function will return a string containing `"[native code]"`. If it returns JavaScript source code, it indicates the function has been hooked or polyfilled, a strong signal of a manipulated environment.
    *   **Presence and Type:** The simple existence of `window.cancelAnimationFrame` as a function is a basic sanity check. Environments that poorly emulate a browser might omit it.

2.  **Behavioral Anomaly Detection:** Anti-bot scripts can perform behavioral tests by scheduling and then immediately cancelling a frame request.
    *   `const id = requestAnimationFrame(() => { /* should not run */ }); cancelAnimationFrame(id);`
    *   In headless browsers (like Puppeteer/Playwright) or emulators that don't have a true GPU-backed rendering loop, the `rAF`/`cAF` implementation might be a shim built on `setTimeout`/`clearTimeout`. While often accurate, subtle timing differences or edge cases in this emulation can be detected by sophisticated scripts that rapidly schedule and cancel many frames, verifying that *only* the non-cancelled callbacks execute. Any failure in this contract suggests a non-standard environment.

In summary, `cAF` is a low-entropy API used as a "tripwire." Its properties and behavior are expected to be consistent with a native browser implementation; any deviation is a high-confidence signal of automation or tampering.

## Sample Code
This snippet demonstrates a common integrity check used by anti-bot scripts to detect if `cancelAnimationFrame` has been tampered with.

```javascript
function isCancelAnimationFrameTampered() {
  // Check 1: Ensure the function exists.
  if (typeof window.cancelAnimationFrame !== 'function') {
    // A very basic bot might not even implement this function.
    return true; 
  }

  // Check 2: Verify it's native code.
  // In a standard browser, this returns "function cancelAnimationFrame() { [native code] }".
  // If a script has overridden it, the source code will be visible.
  try {
    const funcStr = window.cancelAnimationFrame.toString();
    if (!funcStr.includes('[native code]')) {
      // This is a strong indicator of hooking or a polyfill.
      return true;
    }
  } catch (e) {
    // Accessing toString on a proxy or modified object might throw an error.
    return true;
  }

  return false;
}

if (isCancelAnimationFrameTampered()) {
  console.log('Detection: The browser environment appears to be manipulated.');
  // This result would be sent as part of a larger fingerprinting payload.
} else {
  console.log('Check passed: cancelAnimationFrame appears to be a native function.');
}
```