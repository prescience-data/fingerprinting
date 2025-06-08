---
title: window.RTCRtpReceiver
layout: default
---
# window.RTCRtpReceiver
## Purpose
The `RTCRtpReceiver` interface, part of the WebRTC API, is responsible for receiving and decoding a single `MediaStreamTrack` from a remote peer via an `RTCPeerConnection`. It manages the incoming RTP packets for a specific audio or video track.

## Fingerprinting Usage
`RTCRtpReceiver` is a significant source of entropy for browser fingerprinting and a key component in bot detection through consistency checks and anomaly detection.

1.  **Codec & Capability Enumeration**: The static method `RTCRtpReceiver.getCapabilities()` is the primary fingerprinting vector. It returns a detailed object describing the browser's and system's decoding capabilities for a given media kind ('audio' or 'video').
    *   **High Entropy Source**: The returned object includes a list of supported codecs (e.g., VP8, VP9, AV1, H.264), their specific parameters, and supported RTP header extensions. The exact combination, order, and parameters of these codecs are highly dependent on the browser version, operating system, and underlying hardware (especially for hardware-accelerated decoding). This creates a highly unique and stable fingerprint.
    *   **Bot Detection Anomaly**: Headless browsers or bots running in server environments often have a minimal or non-standard set of codecs. For example, a default headless Chrome instance on Linux may lack proprietary codecs like H.264, which are standard on most real user devices (Windows/macOS). The absence of expected codecs or the presence of an unusual combination is a strong signal of a non-standard environment.

2.  **Consistency Checks**: Anti-bot systems use `RTCRtpReceiver` to verify the integrity of the browser environment. The capabilities reported by `RTCRtpReceiver.getCapabilities()` must be consistent with those from other related APIs:
    *   `RTCRtpSender.getCapabilities()`: The sender and receiver capabilities should be logically consistent.
    *   `MediaSource.isTypeSupported()`: A codec reported as supported by WebRTC should also be supported for media playback.
    *   `MediaCapabilities.decodingInfo()`: Provides another source for decoding information that can be cross-referenced.
    A bot that spoofs its User-Agent (e.g., claiming to be Chrome on Windows) but only patches some of these APIs will be detected by the inconsistencies. For instance, reporting a Windows User-Agent while exposing a codec profile typical of Linux is a major red flag.

3.  **Behavioral Signals**: While not a static fingerprint, the behavior of an active `RTCRtpReceiver` instance can be monitored via `RTCPeerConnection.getStats()`. A bot's receiver might exhibit unnaturally perfect network statistics (zero jitter, zero packet loss) if the peer connection is established locally or within a datacenter, which is anomalous compared to a real user's network conditions.

## Sample Code
This snippet demonstrates how to extract a fingerprint from the static `getCapabilities` method without needing to establish a full WebRTC connection.

```javascript
async function getRtcReceiverFingerprint() {
  // The getCapabilities() method is static and can be called directly.
  if (!window.RTCRtpReceiver || !window.RTCRtpReceiver.getCapabilities) {
    return 'WebRTC Receiver API not supported.';
  }

  try {
    const videoCapabilities = RTCRtpReceiver.getCapabilities('video');
    const audioCapabilities = RTCRtpReceiver.getCapabilities('audio');

    // The combination of codecs, their order, and supported header extensions
    // provides a high-entropy fingerprint.
    const fingerprintData = {
      videoCodecs: videoCapabilities.codecs.map(c => ({ mimeType: c.mimeType, sdpFmtpLine: c.sdpFmtpLine })),
      audioCodecs: audioCapabilities.codecs.map(c => c.mimeType),
      headerExtensions: videoCapabilities.headerExtensions.map(h => h.uri),
    };

    // In a real-world scenario, this detailed object would be
    // stringified and hashed (e.g., with MurmurHash) to create a fingerprint ID.
    const fingerprint = JSON.stringify(fingerprintData);
    console.log('RTC Receiver Fingerprint:', fingerprint);
    return fingerprint;

  } catch (e) {
    // An error during enumeration is itself a fingerprintable signal.
    return `Error: ${e.message}`;
  }
}

getRtcReceiverFingerprint();
```