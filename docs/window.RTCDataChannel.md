---
title: window.RTCDataChannel
layout: default
---
# window.RTCDataChannel
## Purpose
The `RTCDataChannel` interface represents a bi-directional data channel between two peers of an `RTCPeerConnection`. It is part of the WebRTC API and enables the low-latency, peer-to-peer exchange of arbitrary data directly between browsers.

## Fingerprinting Usage
While `RTCDataChannel` itself has few unique properties, its creation is intrinsically linked to `RTCPeerConnection`, which is a high-entropy surface for fingerprinting and a critical component in bot detection.

1.  **IP Address Revelation (ICE Candidates)**: This is the most significant use. To establish a peer-to-peer connection, the browser uses the Interactive Connectivity Establishment (ICE) framework. It gathers connection candidates, which often include:
    *   **Local IP Address**: The user's private IP address on the local network (e.g., `192.168.1.10`).
    *   **Public IP Address**: The user's public-facing IP address, discovered via STUN servers.
    This process can reveal a user's true public IP even if they are behind a proxy or most VPNs, as the request to the STUN server is made directly by the browser, bypassing proxy settings configured at the OS or browser extension level. A mismatch between the HTTP request IP and the WebRTC-revealed IP is a strong signal of proxy/VPN usage.

2.  **Network & Hardware Configuration Fingerprint**: The list of ICE candidates generated is unique to the user's network environment and hardware. The **number, type (`host`, `srflx`, `relay`), and order** of candidates provide a detailed fingerprint of the user's network topology (e.g., behind NAT, multiple network interfaces).

3.  **Implementation Artifacts**: The specific format of the SDP (Session Description Protocol) offer and the ICE candidate strings can vary slightly between browser versions and operating systems, adding another layer of entropy.

4.  **Bot & Evasion Detection**:
    *   **Disabled WebRTC**: Many automation frameworks (like older versions of Puppeteer/Selenium) or privacy-focused browser configurations disable WebRTC to prevent IP leaks. The inability to create a `RTCPeerConnection` or gather ICE candidates is a strong signal of a non-standard or automated environment.
    *   **Inconsistent Behavior**: Headless browsers may have an incomplete or emulated WebRTC stack. They might produce a predictable and minimal set of ICE candidates (e.g., only a single `host` candidate) that looks unnatural compared to a real user's browser on a complex network.
    *   **Timing Analysis**: The time it takes to gather ICE candidates can be a behavioral signal. An immediate failure or an unnaturally fast response can indicate a mocked or manipulated API.

## Sample Code
This snippet demonstrates the primary fingerprinting technique: extracting local and public IP addresses from ICE candidates. The creation of a data channel (`pc.createDataChannel('fp')`) is necessary to trigger the ICE gathering process.

```javascript
/**
 * Attempts to gather WebRTC ICE candidates to find local/public IP addresses.
 * @returns {Promise<string[]>} A promise that resolves with an array of unique IP addresses found.
 */
function getWebRTCIPs() {
  return new Promise((resolve) => {
    const ips = new Set();
    // Use a temporary RTCPeerConnection. The STUN server config is optional but helps find public IPs.
    const pc = new RTCPeerConnection({
      iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
    });

    // Listen for ICE candidates
    pc.onicecandidate = (event) => {
      if (event.candidate && event.candidate.candidate) {
        // Extract IP addresses using a regex from the candidate string
        const matches = event.candidate.candidate.match(/(?:\d{1,3}\.){3}\d{1,3}/g);
        if (matches) {
          matches.forEach(ip => ips.add(ip));
        }
      } else {
        // Candidate gathering is complete
        pc.close();
        resolve(Array.from(ips));
      }
    };

    // Create a dummy data channel to trigger ICE gathering
    pc.createDataChannel('fingerprint');
    
    // Create an offer to start the connection process
    pc.createOffer()
      .then(offer => pc.setLocalDescription(offer))
      .catch(err => {
        console.error("WebRTC offer failed:", err);
        resolve([]); // Resolve with empty array on failure
      });

    // Timeout to handle browsers that might not fire the null candidate event
    setTimeout(() => {
        if (pc.connectionState !== 'closed') {
            pc.close();
            resolve(Array.from(ips));
        }
    }, 1000);
  });
}

// Usage:
getWebRTCIPs().then(ips => {
  console.log('Discovered WebRTC IPs:', ips);
  // Fingerprinting logic would hash or analyze this array.
  // Example: ["192.168.1.101", "74.125.140.127"]
});
```