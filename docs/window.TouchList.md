---
title: window.TouchList
layout: default
---
# window.TouchList
## Purpose
`window.TouchList` is a DOM interface representing a list of `Touch` objects. It is not instantiated directly but is the type returned by properties of a `TouchEvent` (e.g., `event.touches`), where each `Touch` object represents a single point of contact on a touch-sensitive surface.

## Fingerprinting Usage
The Touch Events API is a high-value source for both passive fingerprinting and active behavioral analysis to distinguish humans from bots.

1.  **Capability Fingerprinting**: The mere existence of `window.TouchList` and related event handlers like `window.ontouchstart` is a primary indicator of a touch-capable device. Anti-bot systems cross-reference this with the User-Agent string. A browser claiming to be a mobile device (e.g., Android or iOS) but lacking touch event support is a major red flag for an improperly configured automation framework like Selenium or Puppeteer. The value of `navigator.maxTouchPoints` is often checked in conjunction.

2.  **Behavioral Biometrics**: Sophisticated bots may attempt to fire fake touch events. However, the properties of the `Touch` objects within the `TouchList` are difficult to replicate with human-like variance.
    *   **Pressure & Size**: The `force`, `radiusX`, and `radiusY` properties of a `Touch` object are key signals. Human touches have variable pressure and create an imperfect ellipse on the screen. Bots often send static values (e.g., `force: 1.0`, `radiusX: 1.0`, `radiusY: 1.0`) or fail to provide them, resulting in `0` or `undefined`. A sequence of `touchmove` events with identical force and radius values is highly indicative of automation.
    *   **Event Trajectory & Velocity**: The sequence of coordinates from a `TouchList` during a `touchmove` event stream forms a path. Human-generated paths have natural jitter, slight curvature, and non-uniform velocity. Bot-generated swipes are often perfectly linear or follow a simplistic, predictable easing function.
    *   **Inconsistencies**: A bot might successfully spoof a `touchstart` event but fail to correctly populate the `TouchList` in subsequent `touchmove` or `touchend` events, creating detectable anomalies in the event lifecycle.

## Sample Code
This snippet demonstrates checking for touch capability and then analyzing the properties of a touch event for signs of automation.

```javascript
// 1. Basic fingerprinting: Check for touch capability
const hasTouchSupport = ('TouchList' in window) || ('ontouchstart' in window) || (navigator.maxTouchPoints > 0);

// This boolean contributes to the device fingerprint.
// A mismatch with the User-Agent is a strong bot signal.
console.log(`Touch Support Detected: ${hasTouchSupport}`);


// 2. Behavioral analysis: Listen for touch events and inspect properties
document.addEventListener('touchstart', (event) => {
  // event.touches is a TouchList
  if (event.touches.length > 0) {
    const touch = event.touches[0]; // Get the first Touch object

    // Anti-bot systems collect these values to build a behavioral profile.
    const touchProperties = {
      force: touch.force,
      radiusX: touch.radiusX,
      radiusY: touch.radiusY,
      isPerfectCircle: touch.radiusX === touch.radiusY && touch.radiusX > 0,
      isMaxForce: touch.force === 1,
    };

    console.log('Touch Properties:', touchProperties);

    // A real-world system would analyze a sequence of these events.
    // For example, if 'isMaxForce' is always true during a swipe,
    // it's a strong indicator of a bot.
  }
}, { passive: true });
```