---
title: window.RTCErrorEvent
layout: default
---
# window.RTCErrorEvent
## Purpose
Represents an event dispatched when an error occurs during the operation of an `RTCPeerConnection`. It provides detailed, structured information about the nature of the WebRTC failure, beyond a simple error message.

## Fingerprinting Usage
While the primary WebRTC fingerprint comes from leaking IP addresses via `RTCPeerConnection` and `onicecandidate`, `RTCErrorEvent` serves as a powerful secondary signal for bot detection through behavioral analysis.

1.  **Detecting Spoofing and Blocking:** Many privacy extensions, custom browser builds, and bot frameworks (e.g., Puppeteer, Playwright) attempt to disable or spoof WebRTC to prevent IP leakage. These modifications often result in non-standard or predictable errors when a WebRTC connection is attempted. An anti-bot script can initiate a connection and listen for a specific `RTCErrorEvent` whose `error.errorDetail` string (e.g., `"sctp-failure"`, `"dtls-failure"`) is characteristic of a known blocking mechanism rather than a natural network failure.

2.  **Environment Inconsistency:** Headless browsers or virtualized environments may lack the necessary system libraries, media devices, or network stack configurations for WebRTC to function correctly. This can lead to unique error patterns and `RTCErrorEvent` payloads that are inconsistent with the claimed `User-Agent`. For example, a browser claiming to be Chrome on Windows might throw an error typical of a Linux container without a proper audio subsystem.

3.  **Implementation-Specific Error Probing:** Anti-bot systems can intentionally create malformed `RTCPeerConnection` configurations to probe the browser's error-handling logic. The exact properties, error codes, and messages within the resulting `RTCErrorEvent` can differ subtly between browser versions, rendering engines (Blink, Gecko, WebKit), and, most importantly, between a genuine V8 engine and a modified or automated one. These differences in error reporting provide a high-entropy signal to detect tampering.

## Sample Code
This snippet demonstrates how an anti-bot script might intentionally trigger a WebRTC error to analyze the resulting `RTCErrorEvent` for anomalies.

```javascript
async function getRtcErrorProfile() {
  return new Promise(resolve => {
    try {
      // Intentionally use an invalid ICE server to force an error.
      // A real-world script might use more complex, malformed configs.
      const pc = new RTCPeerConnection({
        iceServers: [{ urls: 'stun:invalid.server.invalid:1' }]
      });

      // The key listener for fingerprinting.
      pc.onerror = (event) => {
        // event is an instance of RTCErrorEvent.
        const errorDetails = {
          name: event.error.name,
          message: event.error.message,
          errorDetail: event.error.errorDetail,
          // The exact string format and content are compared against a
          // database of known-good browser fingerprints.
        };
        pc.close();
        resolve(errorDetails);
      };

      // Set a timeout in case onerror doesn't fire (e.g., if WebRTC is completely disabled).
      setTimeout(() => resolve({ error: 'Timeout' }), 1000);

      // This sequence triggers the ICE gathering process, which will fail.
      pc.createDataChannel('probe');
      pc.createOffer().then(offer => pc.setLocalDescription(offer));

    } catch (e) {
      // A synchronous error can also be a signal (e.g., constructor is patched).
      resolve({ syncError: e.name, message: e.message });
    }
  });
}

// An anti-bot system would call this function and send the result to its server.
getRtcErrorProfile().then(profile => {
  console.log('RTC Error Profile:', profile);
  // Example output on Chrome:
  // { name: 'RTCError', message: '...', errorDetail: 'ice-server-resolve-failure' }
});
```