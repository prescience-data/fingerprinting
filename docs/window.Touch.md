---
title: window.Touch
layout: default
---
# window.Touch
## Purpose
The `window.Touch` constructor is a core component of the Touch Events API. It is used to create `Touch` objects, each representing the state of a single point of contact (e.g., a finger or stylus) on a touch-sensitive surface.

## Fingerprinting Usage
The `window.Touch` API is a high-value signal for fingerprinting and bot detection, primarily through consistency checks rather than generating a unique ID.

1.  **Existence as a Primary Signal:** The mere presence or absence of the `window.Touch` constructor is a significant piece of entropy. A script can quickly determine if a browser environment claims to support touch events. `typeof window.Touch === 'undefined'` is a strong indicator of a desktop environment.

2.  **Inconsistency Detection:** This is the most critical use case for anti-bot systems. Many automated browsers (e.g., Puppeteer, Playwright) are configured to emulate mobile devices by setting a mobile User-Agent string and viewport dimensions. However, by default, the underlying browser instance running on a server does not have touch event capabilities enabled. An anti-bot script can detect this discrepancy:
    *   **Check 1:** Read `navigator.userAgent`. If it contains "iPhone", "Android", or "Mobi", the device claims to be mobile.
    *   **Check 2:** Check for touch support via `typeof window.Touch !== 'undefined'` or `'ontouchstart' in window`.
    *   **Anomaly:** A browser with a mobile User-Agent but no `window.Touch` API is almost certainly a bot or a misconfigured emulator. Real mobile browsers will always expose this API.

3.  **Behavioral Analysis:** Even if a sophisticated bot spoofs the `window.Touch` API by defining a dummy object, it cannot easily generate realistic touch event data. Real user interactions produce a stream of `touchmove` events with changing coordinates, pressure (`force`), and contact area (`radiusX`, `radiusY`). The absence of these events on an interactive page, or the presence of perfectly linear, machine-generated event data, can be used to flag automated behavior.

## Sample Code
This snippet demonstrates the core inconsistency check between the User-Agent and the presence of touch APIs.

```javascript
function isTouchConfigurationConsistent() {
  // 1. Check if the User-Agent string claims to be a mobile device.
  const ua = navigator.userAgent;
  const claimsMobile = /Mobi|Android|iPhone|iPad|iPod/i.test(ua);

  // 2. Check for the actual presence of the Touch constructor in the window object.
  // This is a direct check for the browser's native touch event implementation.
  const hasTouchAPI = typeof window.Touch !== 'undefined';

  // 3. A mobile UA without the corresponding Touch API is a major red flag.
  if (claimsMobile && !hasTouchAPI) {
    console.warn('Anomaly Detected: Mobile User-Agent without Touch API support.');
    return false; // High probability of being a bot or emulator.
  }

  // 4. A non-mobile UA with a Touch API is also unusual (e.g., touchscreen laptop)
  // but less suspicious than the inverse.
  if (!claimsMobile && hasTouchAPI) {
     console.info('Note: Non-mobile User-Agent with Touch API support (e.g., touchscreen laptop).');
  }

  return true; // Configuration appears consistent.
}

// Example usage:
if (!isTouchConfigurationConsistent()) {
  // Trigger additional checks or flag the session.
}
```