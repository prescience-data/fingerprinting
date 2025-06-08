---
title: window.TouchEvent
layout: default
---
# window.TouchEvent
## Purpose
The `TouchEvent` interface represents an event sent when the state of contacts with a touch-sensitive surface changes. It is the foundation for handling user input on touch-enabled devices (smartphones, tablets, touch-screen laptops) through events like `touchstart`, `touchmove`, and `touchend`.

## Fingerprinting Usage
The `TouchEvent` API is a powerful signal for differentiating device types and detecting inconsistencies in a browser's claimed identity.

1.  **API Presence Check:** The most fundamental test is the existence of the `window.TouchEvent` constructor.
    *   `typeof window.TouchEvent !== 'undefined'` strongly indicates the browser is running on a device with touch capability.
    *   Its absence is a strong signal for a traditional desktop environment.
    *   **Bot Detection:** This creates a critical consistency check. A bot or headless browser (like Puppeteer or Playwright) might spoof its User-Agent string to appear as a mobile device (e.g., "iPhone" or "Android"). However, these environments typically run on servers without touch hardware, so `window.TouchEvent` will be undefined. This discrepancy between the claimed User-Agent and the available hardware APIs is a major red flag for anti-bot systems.

2.  **Behavioral Analysis:** For browsers that *do* support `TouchEvent`, analyzing the properties of actual events provides high-entropy behavioral signals. Real human touches are physically nuanced, whereas automated events are often unnaturally perfect.
    *   **Touch Properties (`force`, `radiusX`, `radiusY`):** A real finger press has variable pressure (`force`) and a non-uniform contact area (`radiusX`, `radiusY`). Automated touch events often have a constant `force` of `1` (or `0` if unsupported) and a fixed, uniform radius. A stream of `touchmove` events with identical force and radius values is highly suspicious.
    *   **Event Timing:** The timing and sequence of `touchstart`, `touchmove`, and `touchend` events from a human are irregular. Bots may fire these events with programmatic, perfectly regular intervals or with zero delay, which is non-human behavior.
    *   **Movement Path:** The path traced by a series of `touchmove` events from a human finger exhibits slight jitter and is never a mathematically perfect line or curve. Automated scripts often generate flawless paths.

3.  **Hardware Capability (`navigator.maxTouchPoints`):** While not part of `TouchEvent` itself, `navigator.maxTouchPoints` is a closely related property that reveals the maximum number of simultaneous touch points the hardware supports. This value contributes to the device fingerprint (e.g., most phones support 5 or 10 points, while a non-touch device reports 0).

## Sample Code
This snippet demonstrates a common consistency check used in bot detection. It verifies if a browser claiming to be mobile actually has the corresponding touch APIs.

```javascript
function analyzeTouchConsistency() {
  const fingerprint = {
    isSuspicious: false,
    reason: '',
    hasTouchEventAPI: typeof window.TouchEvent !== 'undefined',
    maxTouchPoints: navigator.maxTouchPoints || 0,
    claimsMobile: /iphone|ipad|ipod|android/i.test(navigator.userAgent)
  };

  // Primary red flag: The browser's User-Agent claims it's a mobile device,
  // but it lacks the fundamental TouchEvent API and reports no touch points.
  // This is a classic sign of a headless browser spoofing its identity.
  if (fingerprint.claimsMobile && !fingerprint.hasTouchEventAPI && fingerprint.maxTouchPoints === 0) {
    fingerprint.isSuspicious = true;
    fingerprint.reason = 'User-Agent/TouchEvent API mismatch.';
  }

  // A less common but still possible scenario
  if (!fingerprint.claimsMobile && fingerprint.hasTouchEventAPI) {
    // This could be a touch-screen Windows laptop, which is usually legitimate.
    // However, it's still a data point for fingerprinting.
    fingerprint.reason = 'Desktop User-Agent with Touch support.';
  }

  return fingerprint;
}

const analysisResult = analyzeTouchConsistency();
console.log(analysisResult);

// On a real iPhone:
// { isSuspicious: false, reason: '', hasTouchEventAPI: true, maxTouchPoints: 5, claimsMobile: true }

// On a desktop Chrome faking an iPhone User-Agent:
// { isSuspicious: true, reason: 'User-Agent/TouchEvent API mismatch.', hasTouchEventAPI: false, maxTouchPoints: 0, claimsMobile: true }
```