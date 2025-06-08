---
title: window.InputDeviceCapabilities
layout: default
---
# window.InputDeviceCapabilities
## Purpose
The `InputDeviceCapabilities` interface provides information about the physical device responsible for generating a UI event. It is accessed via the `sourceCapabilities` read-only property on `UIEvent` objects (like `MouseEvent` or `TouchEvent`). Its primary function is to indicate if the input mechanism (e.g., a mouse) can also generate touch events.

## Fingerprinting Usage
This API is a high-value signal for bot detection due to the inconsistencies it reveals in spoofed environments. Its value is not in providing high entropy, but in validating the authenticity of the user's claimed environment.

*   **Primary Anomaly Detection:** The most common use is to cross-reference the `firesTouchEvents` boolean property with the `User-Agent` string.
    *   **Real Mobile User:** A tap on a mobile device generates a `TouchEvent`. For compatibility, browsers often dispatch a subsequent `MouseEvent` (`click`). This synthetic `MouseEvent` will have `event.sourceCapabilities.firesTouchEvents` set to `true`.
    *   **Desktop Bot:** An automation framework (like Puppeteer or Playwright) running on a server will typically spoof a mobile User-Agent. However, when it simulates a click, it generates a standard `MouseEvent` from a non-touch source. This results in `event.sourceCapabilities.firesTouchEvents` being `false`.
    *   **The Signal:** A `click` event on a page with a mobile User-Agent where `firesTouchEvents` is `false` is a strong indicator of a desktop-based bot faking a mobile environment.

*   **Spoofing Difficulty:** The `InputDeviceCapabilities` object cannot be constructed or modified directly from JavaScript (`new InputDeviceCapabilities()` will throw an error). This makes it difficult for unsophisticated bots to create a fake `MouseEvent` with `firesTouchEvents` set to `true`. They would need to use more advanced browser modification techniques, which are less common.

*   **Behavioral Signal:** On hybrid devices (like touchscreen laptops), a user might generate a mix of events: mouse clicks with `firesTouchEvents: false` and touch taps with `firesTouchEvents: true`. Analyzing the sequence and timing of these different event types can provide a behavioral fingerprint that is difficult for bots to replicate realistically.

## Sample Code
This snippet demonstrates the core logic of checking for a mismatch between the User-Agent and the capabilities of the input device that triggered a click.

```javascript
document.body.addEventListener('click', (event) => {
  // Check if the browser supports the API
  if (!event.sourceCapabilities) {
    console.log('InputDeviceCapabilities not supported.');
    return;
  }

  const isMobileUA = /Mobi/i.test(navigator.userAgent);
  const firesTouchEvents = event.sourceCapabilities.firesTouchEvents;

  console.log(`User-Agent claims mobile: ${isMobileUA}`);
  console.log(`Input device fires touch events: ${firesTouchEvents}`);

  // The core detection logic:
  // If the User-Agent looks mobile, but the click event came from a 
  // device that doesn't fire touch events, it's a major red flag.
  if (isMobileUA && !firesTouchEvents) {
    console.warn('Potential Bot Detected: Mobile User-Agent with non-touch click event.');
    // Trigger anti-bot measures here.
  }
}, true); // Use capture phase to get the event early.
```