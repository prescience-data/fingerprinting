---
title: window.RTCRtpSender
layout: default
---
# window.RTCRtpSender
## Purpose
The `RTCRtpSender` interface, part of the WebRTC API, manages the encoding and transmission of a single `MediaStreamTrack` to a remote peer. It is created when a track is added to an `RTCPeerConnection` and provides control over the outgoing media stream.

## Fingerprinting Usage
The primary fingerprinting vector associated with `RTCRtpSender` is its static method `getCapabilities()`. This method does not require an active peer connection or user media permissions, making it ideal for passive, covert fingerprinting.

*   **Codec Enumeration:** `RTCRtpSender.getCapabilities('video')` and `RTCRtpSender.getCapabilities('audio')` return a detailed list of codecs the browser can encode. This list includes properties like `mimeType`, `clockRate`, and `sdpFmtpLine`.
*   **High Entropy Source:** The specific set of supported codecs, their parameters, and their order of preference is highly dependent on:
    *   **Browser & Version:** Chrome, Firefox, and Safari support different codec sets (e.g., VP8, VP9, AV1, H.264). Minor browser updates can add, remove, or alter codecs.
    *   **Operating System:** The underlying OS provides media frameworks that influence codec availability. For example, H.264 support is often tied to OS-level libraries.
    *   **Hardware:** The presence of hardware acceleration (e.g., via Intel Quick Sync, Nvidia NVENC) can add or prioritize specific codecs and profiles, making the list unique to the user's hardware configuration.
*   **Bot Detection & Anomaly Detection:**
    *   **Inconsistency:** A bot might spoof its User-Agent to appear as "Chrome on Windows" but its `getCapabilities()` output might reveal a codec profile typical of a Linux server (e.g., lacking proprietary, hardware-accelerated codecs). This mismatch is a strong bot signal.
    *   **Headless Browsers:** Automated browsers like Puppeteer and Playwright often run in headless mode, which typically exposes a minimal or distinct set of codecs compared to a standard GUI browser installation. The absence of expected codecs is a red flag.
    *   **Spoofing Detection:** Anti-bot systems can detect attempts to tamper with `getCapabilities`. They can check if the function is native (`.toString().includes('[native code]')`) or compare the function from the main window with a pristine version from a newly created `iframe`.

## Sample Code
This snippet demonstrates how to collect audio and video codec information without requesting any user permissions, then combine it into a fingerprint.

```javascript
async function getWebRTCCodecFingerprint() {
  try {
    // These static methods can be called without an active PeerConnection.
    const videoCapabilities = RTCRtpSender.getCapabilities('video');
    const audioCapabilities = RTCRtpSender.getCapabilities('audio');

    // Extract MIME types as a simple representation of the codec list.
    // A real-world script would likely use more detail (e.g., sdpFmtpLine).
    const videoCodecs = videoCapabilities.codecs.map(c => c.mimeType).join(',');
    const audioCodecs = audioCapabilities.codecs.map(c => c.mimeType).join(',');

    const fingerprint = `video:${videoCodecs}|audio:${audioCodecs}`;
    
    console.log('WebRTC Codec Fingerprint:', fingerprint);
    // Example Output: 
    // "video:video/VP8,video/VP9,video/AV1,video/H264|audio:audio/opus,audio/G722,audio/PCMU,audio/PCMA"
    
    // This string can be hashed to create a stable identifier.
    return fingerprint;

  } catch (e) {
    // If WebRTC is disabled or unavailable, this itself is a fingerprinting signal.
    console.error('WebRTC capabilities not available.', e);
    return 'webrtc-unavailable';
  }
}

getWebRTCCodecFingerprint();
```