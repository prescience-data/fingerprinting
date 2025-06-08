---
title: window.RTCDTMFSender
layout: default
---
# window.RTCDTMFSender
## Purpose
The `RTCDTMFSender` interface, part of the WebRTC API, enables the sending of Dual-Tone Multi-Frequency (DTMF) signals through an `RTCPeerConnection`. It is used to transmit telephone dialing tones (0-9, A-D, #, *) over an active audio stream.

## Fingerprinting Usage
While not a high-entropy source on its own, `RTCDTMFSender` is a valuable API for detecting inconsistencies common in bot and headless browser environments. Detection focuses on implementation artifacts and behavioral deviations rather than unique user values.

1.  **API Surface & Integrity:** The mere existence and properties of the `RTCDTMFSender` constructor and its prototype are checked. Bots often use JavaScript shims or proxies to emulate browser APIs. These shims can be detected because they don't perfectly replicate the native C++ implementation in V8.
    *   **Native Code Check:** `window.RTCDTMFSender.toString()` must return a string containing `"[native code]"`. A JavaScript function string indicates tampering.
    *   **Constructor Properties:** `RTCDTMFSender.name` should be `"RTCDTMFSender"` and `RTCDTMFSender.length` (arity) should be `0`. Deviations suggest a fake implementation.
    *   **Prototype Chain:** The prototype chain (`Object.getPrototypeOf(RTCDTMFSender.prototype)`) must correctly link to `EventTarget.prototype`.

2.  **Behavioral Anomalies (Error Handling):** The most powerful detection method is to test the API's behavior with invalid input. Native browser implementations have very specific error-handling logic defined by web standards, which bots often fail to replicate perfectly.
    *   **Invalid Tones:** Calling `insertDTMF()` with a character that is not a valid DTMF tone (e.g., `'g'`, `'z'`) must throw a `DOMException` with the specific `name` property of `"InvalidCharacterError"`. A bot might throw a generic `TypeError`, a different `DOMException`, or fail to throw an error at all. This is a strong signal of a non-native environment.
    *   **State Management:** Attempting to use the sender on a closed `RTCPeerConnection` or on a track that is not active should result in specific, predictable errors. Bots may not correctly manage the internal state of the WebRTC connection, leading to different or missing errors.

## Sample Code
This snippet demonstrates how an anti-bot script might check for both static properties and the crucial error-handling behavior of `RTCDTMFSender`.

```javascript
async function analyzeRTCDTMFSender() {
  const results = {
    exists: 'RTCDTMFSender' in window,
    isNative: false,
    errorTypeOnInvalidTone: 'n/a',
  };

  if (!results.exists) {
    return results; // API missing is a strong signal
  }

  // 1. Static Integrity Check
  // A tampered function will not include "[native code]".
  results.isNative = window.RTCDTMFSender.toString().includes('[native code]');

  // 2. Behavioral Check (Error Handling)
  let pc = null;
  try {
    // A minimal WebRTC setup is required to get a DTMF sender instance.
    pc = new RTCPeerConnection();
    const dtmfSender = pc.addTransceiver('audio').sender.dtmf;

    // This call should throw a specific DOMException.
    dtmfSender.insertDTMF('123-invalid-tone-X'); 
    
    // If code reaches here, the browser did not throw, which is a red flag.
    results.errorTypeOnInvalidTone = 'no_error_thrown';

  } catch (e) {
    // A genuine Chrome/V8 implementation throws a DOMException named "InvalidCharacterError".
    // A bot might throw a generic TypeError or something else.
    results.errorTypeOnInvalidTone = e.name; // Expected: "InvalidCharacterError"
  } finally {
    if (pc) {
      pc.close();
    }
  }

  return results;
}

// Example of analysis:
// analyzeRTCDTMFSender().then(fingerprint => {
//   console.log(fingerprint);
//   // {
//   //   exists: true,
//   //   isNative: true,
//   //   errorTypeOnInvalidTone: "InvalidCharacterError"
//   // }
// });
```