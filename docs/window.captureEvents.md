---
title: window.captureEvents
layout: default
---
# window.captureEvents
## Purpose
An obsolete, non-standard method originating from Netscape Navigator 4 to register the `window` to capture all events of a specified type. It has been superseded by the W3C standard `element.addEventListener(type, listener, useCapture)`.

## Fingerprinting Usage
The primary fingerprinting value of `window.captureEvents` lies in its obsolescence. Its presence or absence is a key signal.

*   **Anomaly Detection:** Modern browsers, including Chrome, have removed this method. The expected value of `typeof window.captureEvents` is `'undefined'`. If this function exists, it is a powerful indicator of a non-standard environment. This could mean:
    1.  An extremely old, un-updated browser.
    2.  A custom or non-standard browser engine.
    3.  An automation framework (like Selenium, Puppeteer, Playwright) that has been imperfectly configured or modified, failing to hide or remove this legacy artifact.
    4.  A script on the page has polyfilled or defined this function, which is itself an environmental anomaly.

*   **Consistency Check:** Anti-bot systems can check for the presence of `window.captureEvents` and its counterpart `window.releaseEvents`. The presence of one without the other, or the presence of either, can be used to challenge the integrity of the browser environment. The expected state in a genuine, up-to-date Chrome browser is for both to be undefined.

The entropy comes not from its output, but from its very existence, which deviates from the modern browser API surface.

## Sample Code
This snippet demonstrates how a detection script would check for the presence of this obsolete API to flag a potentially non-standard or automated client.

```javascript
function detectLegacyEventModel() {
  const isSuspicious = typeof window.captureEvents === 'function' || 
                       typeof window.releaseEvents === 'function';

  if (isSuspicious) {
    // This is a strong signal that the environment is not a standard,
    // modern Chrome browser. It could be a bot or a very old UA.
    console.warn('Legacy event model API detected. High probability of automation.');
    return {
      risk: 'high',
      reason: 'Found obsolete window.captureEvents or window.releaseEvents.'
    };
  }

  return {
    risk: 'low',
    reason: 'Standard browser environment.'
  };
}

const analysis = detectLegacyEventModel();
// An anti-bot system would use this result to increase the user's risk score.
```