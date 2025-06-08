---
title: window.RTCStatsReport
layout: default
---
# window.RTCStatsReport
## Purpose
The `RTCStatsReport` object is a map-like collection of detailed statistics for a WebRTC `RTCPeerConnection`. It is returned by the `getStats()` method and is used to monitor and debug the performance, quality, and characteristics of a real-time media session.

## Fingerprinting Usage
`RTCStatsReport` is a high-entropy source used for both fingerprinting and behavioral bot detection. While the IP address leakage via ICE candidates is the most famous part of WebRTC fingerprinting, the `getStats()` report provides a much richer, more subtle surface.

1.  **Implementation Fingerprinting**: The exact set of statistics available in the report, their names, and their `type` values (`candidate-pair`, `inbound-rtp`, `codec`, `media-source`, etc.) vary significantly between browser vendors (Chrome vs. Firefox), versions, and underlying operating systems. The presence or absence of proprietary stats (e.g., Chrome's `goog*` prefixed stats like `googJitterBufferMs`) is a strong signal of the true browser engine, making it difficult for bots to spoof a user agent string accurately.

2.  **Network Anomaly Detection**: The report contains detailed network metrics that can unmask automated environments.
    *   **Round Trip Time (RTT) & Jitter**: A bot running in a datacenter connecting to a nearby STUN/TURN server will exhibit near-zero RTT, jitter, and packet loss. A real user on a residential Wi-Fi or cellular network will almost always have non-zero, fluctuating values for these metrics. An RTT of `< 1ms` is a major red flag for bot detection.
    *   **Candidate Type**: The `candidate-pair` stats reveal the type of connection (`host`, `srflx`, `prflx`, `relay`). Bots in cloud environments often have a simple `host` connection with a public IP, whereas most real users are behind a NAT and will use a `srflx` (server reflexive) candidate.

3.  **Hardware & Driver Inconsistencies**: For bots that fake media streams (e.g., a virtual webcam), the `media-source` stats can reveal anomalies. A real webcam's frame rate (`framesPerSecond`) fluctuates slightly, and metrics like `jitterBufferDelay` will be present and dynamic. A faked stream may present a perfectly constant FPS (e.g., `30.0000`) and zero jitter, which is unnatural.

4.  **Consistency Cross-Checks**: The data in `RTCStatsReport` can be cross-referenced with other browser signals. If an IP address leaked via an ICE candidate belongs to a residential ISP, but the `RTCStatsReport` shows datacenter-grade network performance, the user is flagged as deceptive.

## Sample Code
This snippet demonstrates how to retrieve the stats report and check for common bot-like network characteristics.

```javascript
async function analyzeRtcStats() {
  try {
    // A minimal RTCPeerConnection is sufficient to generate stats.
    // No signaling server or remote peer is needed for this fingerprinting.
    const pc = new RTCPeerConnection({
      iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
    });

    // Create a dummy data channel to trigger ICE candidate gathering.
    pc.createDataChannel('');
    pc.createOffer().then(offer => pc.setLocalDescription(offer));

    // Wait a moment for ICE candidates to be gathered and connections to be attempted.
    await new Promise(resolve => setTimeout(resolve, 500));

    const report = await pc.getStats();
    const fingerprint = {
      statTypes: new Set(),
      isPristineNetwork: false,
      rtt: null
    };

    report.forEach(stat => {
      // 1. Collect all unique stat types. The set of types is a fingerprint.
      fingerprint.statTypes.add(stat.type);

      // 2. Check for datacenter-like network conditions.
      if (stat.type === 'candidate-pair' && stat.state === 'succeeded') {
        // currentRoundTripTime is measured in seconds.
        if (typeof stat.currentRoundTripTime === 'number') {
          fingerprint.rtt = stat.currentRoundTripTime * 1000; // convert to ms
          if (fingerprint.rtt < 5) {
            // RTT < 5ms is highly indicative of a bot in a datacenter.
            fingerprint.isPristineNetwork = true;
          }
        }
      }
    });
    
    pc.close();
    console.log('RTC Stats Fingerprint:', fingerprint);
    return fingerprint;

  } catch (e) {
    // WebRTC might be disabled or unavailable (e.g., in some headless environments).
    console.error('WebRTC stats collection failed:', e);
    return null;
  }
}

analyzeRtcStats();
```