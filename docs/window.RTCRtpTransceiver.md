---
title: window.RTCRtpTransceiver
layout: default
---
# window.RTCRtpTransceiver
## Purpose
The `RTCRtpTransceiver` interface is a core component of the WebRTC API. It represents a single media stream, bundling an `RTCRtpSender` for sending data and an `RTCRtpReceiver` for receiving data, which are managed together within an `RTCPeerConnection`.

## Fingerprinting Usage
While the `RTCRtpTransceiver` object itself has few unique properties, its creation and the components it manages are central to several powerful fingerprinting and bot detection vectors.

1.  **Codec Capabilities (High Entropy):** The primary fingerprinting value comes from the sender and receiver capabilities associated with a transceiver. Calling `RTCRtpSender.getCapabilities('audio'/'video')` reveals a detailed list of supported media codecs (e.g., Opus, VP9, H.264, AV1). This list, including the specific parameters for each codec (`clockRate`, `channels`, `sdpFmtpLine`), is highly dependent on:
    *   **Browser & Version:** Chrome, Firefox, and Safari expose different codec sets.
    *   **Operating System:** The underlying OS media framework affects available codecs.
    *   **Hardware:** The presence of specific hardware (like a GPU with video encoding/decoding acceleration) dramatically changes the available video codecs.
    This combination creates a stable, high-entropy fingerprint that is difficult to spoof accurately.

2.  **IP Address Leakage via ICE:** Creating an `RTCRtpTransceiver` is a necessary step to initiate the Interactive Connectivity Establishment (ICE) process. When an `RTCPeerConnection` gathers ICE candidates, it can expose the user's **local/private IP address** (e.g., `192.168.x.x`) and, in some network configurations, their **public IP address**. This is a classic de-anonymization technique that can bypass proxies and VPNs, revealing the user's true network environment.

3.  **Behavioral & Anomaly Detection:** The *way* a transceiver is created provides behavioral signals.
    *   **Legitimate Use:** In a real application (e.g., a video chat), a transceiver is often created implicitly when a `MediaStreamTrack` from `navigator.mediaDevices.getUserMedia()` is added to the connection.
    *   **Bot/Fingerprinting Heuristic:** A script that programmatically creates a "dummy" transceiver (e.g., via `pc.addTransceiver('audio')`) without any user interaction or preceding `getUserMedia` call is highly suspicious. Anti-bot systems flag this behavior as it's a common pattern for silently triggering the ICE process to harvest IP addresses and codec information.

## Sample Code
This snippet demonstrates how to create a transceiver to extract both the local IP address and the detailed codec capabilities.

```javascript
async function getRtcFingerprint() {
  const fingerprint = {
    localIPs: new Set(),
    audioCodecs: [],
  };

  // Use a public STUN server to trigger ICE candidate gathering.
  const pc = new RTCPeerConnection({
    iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
  });

  // 1. Create a dummy transceiver to trigger the process.
  // This is a key behavioral signal for bot detection.
  const transceiver = pc.addTransceiver('audio');

  // 2. Extract detailed codec capabilities via the transceiver's sender.
  // This provides a high-entropy hardware/software fingerprint.
  try {
    const caps = transceiver.sender.getCapabilities('audio');
    fingerprint.audioCodecs = caps.codecs;
  } catch (e) {
    console.error('Could not get codec capabilities:', e);
  }

  // 3. Listen for ICE candidates to harvest local IP addresses.
  pc.onicecandidate = (event) => {
    if (event.candidate && event.candidate.candidate) {
      // Regex to find IP addresses in the SDP candidate string.
      const ipRegex = /([0-9]{1,3}(\.[0-9]{1,3}){3}|[a-f0-9]{1,4}(:[a-f0-9]{1,4}){7})/g;
      const matches = event.candidate.candidate.match(ipRegex);
      if (matches) {
        matches.forEach(ip => fingerprint.localIPs.add(ip));
      }
    }
  };

  // Create an offer to start the connection and ICE gathering process.
  await pc.createOffer();

  // Allow some time for candidates to be gathered.
  await new Promise(resolve => setTimeout(resolve, 500));
  
  pc.close();

  // Convert Set to Array for easier consumption.
  fingerprint.localIPs = Array.from(fingerprint.localIPs);
  return fingerprint;
}

// Example usage:
getRtcFingerprint().then(fp => {
  console.log("WebRTC Fingerprint:", fp);
  // fp.localIPs might contain: ["192.168.1.10", ...]
  // fp.audioCodecs will be an array of codec-specific objects.
});
```