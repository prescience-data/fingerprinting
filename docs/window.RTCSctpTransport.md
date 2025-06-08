---
title: window.RTCSctpTransport
layout: default
---
# window.RTCSctpTransport
## Purpose
The `RTCSctpTransport` interface provides information about the Stream Control Transmission Protocol (SCTP) transport layer used for WebRTC data channels (`RTCDataChannel`). It exposes details about the transport's state, capabilities, and configuration.

## Fingerprinting Usage
`RTCSctpTransport` is a component of the larger WebRTC API surface, which is a rich source of fingerprinting entropy. While not as commonly cited as `RTCIceCandidate`, it provides several valuable signals for bot detection.

*   **Existence and Type:** The mere presence of `window.RTCSctpTransport` is a basic check. More importantly, its type (`typeof window.RTCSctpTransport` should be `'function'`) and its `toString()` representation (`"function RTCSctpTransport() { [native code] }"`) are checked. Inconsistent values suggest a tampered or non-standard browser environment.

*   **Instantiation Behavior:** `RTCSctpTransport` cannot be instantiated directly via its constructor (`new RTCSctpTransport()`). Doing so must throw a `TypeError`. Bot frameworks that naively mock browser APIs might incorrectly allow instantiation, which is a strong signal of an automated environment. A real browser only creates instances internally during the `RTCPeerConnection` setup.

*   **Prototype Properties:** The properties available on `RTCSctpTransport.prototype` (e.g., `transport`, `state`, `maxMessageSize`, `onstatechange`) and their order can differ between browser versions and forks (e.g., Brave vs. Chrome). Enumerating these properties provides a fine-grained browser version fingerprint.

*   **Property Values from an Active Connection:** When an `RTCPeerConnection` is established, the properties of the resulting `sctpTransport` instance provide high-entropy signals:
    *   `maxMessageSize`: This value can vary based on the underlying OS, browser build, and specific WebRTC library version (libwebrtc). While often a large default, any deviation is a significant fingerprinting marker. Headless browsers may use a different default value than their headed counterparts.
    *   `state`: Monitoring the `state` property or the `onstatechange` event provides a behavioral signal. A real connection will transition through states (`connecting`, `connected`). A bot might fail to establish a connection (remaining `connecting`) or have unnatural state transition timing if its networking stack is virtualized or restricted.

## Sample Code
This snippet demonstrates a check for the API's instantiation behavior, a common anti-bot technique.

```javascript
function checkSctpTransportBehavior() {
  const result = {
    exists: 'RTCSctpTransport' in window,
    isConstructor: false,
    throwsOnNew: false,
    errorType: null,
  };

  if (!result.exists) {
    return result; // High-confidence bot or old browser
  }

  try {
    // This should fail in any standard browser environment.
    // A bot might incorrectly allow this or throw the wrong error type.
    new window.RTCSctpTransport();
  } catch (e) {
    result.throwsOnNew = true;
    // The expected error is a TypeError. Any other error is suspicious.
    if (e instanceof TypeError) {
      result.isConstructor = true; // Behaves like a native constructor
      result.errorType = 'TypeError';
    } else {
      result.errorType = e.name; // e.g., 'ReferenceError', etc.
    }
  }

  // A real Chrome browser should return:
  // { exists: true, isConstructor: true, throwsOnNew: true, errorType: 'TypeError' }
  // Any other result indicates a potential anomaly.
  return result;
}

console.log(checkSctpTransportBehavior());
```