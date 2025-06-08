---
title: window.RTCIceCandidate
layout: default
---
# window.RTCIceCandidate
## Purpose
Represents a candidate network connection path discovered by the ICE (Interactive Connectivity Establishment) framework. It's a core component of WebRTC used to find the best way for two peers to communicate directly, even when they are behind NATs or firewalls.

## Fingerprinting Usage
`RTCIceCandidate` is one of the most powerful and widely used browser fingerprinting vectors. Its primary use is to reveal IP addresses that are not visible in standard HTTP requests.

1.  **Local IP Address Leakage:** The most significant use is the exposure of a user's local (private) IP address (e.g., `192.168.1.100`, `10.0.0.5`). This is a high-entropy signal because it's specific to the user's local network configuration. While not globally unique, it significantly narrows down the user's identity when combined with other metrics. Bots running in simple Docker containers often expose predictable local IPs (e.g., `172.17.0.2`), making them easy to identify.

2.  **Public IP Address Verification:** The ICE process uses STUN (Session Traversal Utilities for NAT) servers to discover the user's public IP address. This creates a "server reflexive" (`srflx`) candidate. Anti-bot systems compare this WebRTC-discovered public IP with the source IP of the HTTP request. A mismatch is a strong indicator of a proxy or VPN, which is common bot behavior. A real user behind a NAT will show a private IP via WebRTC and a public IP via HTTP headers (which is expected), but a bot using a proxy will often show two *different* public IPs.

3.  **Network Configuration Enumeration:** The number, type (`host`, `srflx`, `relay`), and protocol (UDP/TCP) of generated candidates can reveal details about the user's network topology.
    *   A simple home network might generate only a few candidates (IPv4 local, IPv6 local, public srflx).
    *   A corporate network or a device with multiple network interfaces (Wi-Fi, Ethernet, VPN) will generate a more complex and unique set of candidates.
    *   Bots in a datacenter often have a very consistent and minimal set of candidates, which becomes a fingerprint for that bot farm.

4.  **Behavioral Anomaly Detection:** If a browser reports support for WebRTC but fails to generate any ICE candidates (especially when a STUN server is provided), it suggests that the functionality has been disabled or tampered with. This is a common tactic in privacy-oriented browsers or automation frameworks like Puppeteer/Playwright when attempting to hide their identity, making the *absence* of a signal a signal in itself.

## Sample Code
This snippet demonstrates how to initiate an `RTCPeerConnection` to trigger ICE candidate generation and collect any discovered IP addresses.

```javascript
/**
 * Gathers WebRTC ICE candidates to extract local and public IP addresses.
 * @returns {Promise<string[]>} A promise that resolves with an array of unique IP addresses.
 */
function getWebRTCIPs() {
  return new Promise((resolve) => {
    const ips = new Set();
    // Use a public STUN server to find the public IP.
    const pc = new RTCPeerConnection({
      iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
    });

    // The onicecandidate event fires when a new ICE candidate is found.
    pc.onicecandidate = (event) => {
      // The last event will have a null candidate.
      if (!event.candidate) {
        pc.close();
        resolve(Array.from(ips));
        return;
      }

      // The candidate string contains the IP address.
      // Example: "candidate:842163049 1 udp 2122260223 192.168.1.100 58736 typ host ..."
      const candidateString = event.candidate.candidate;
      
      // Use regex to extract IP addresses (IPv4 and IPv6).
      const ipRegex = /([0-9]{1,3}(\.[0-9]{1,3}){3}|[a-f0-9]{1,4}(:[a-f0-9]{1,4}){7})/g;
      const matches = candidateString.match(ipRegex);

      if (matches) {
        matches.forEach(ip => ips.add(ip));
      }
    };

    // Create a dummy data channel to trigger the ICE gathering process.
    pc.createDataChannel('');
    pc.createOffer()
      .then(offer => pc.setLocalDescription(offer))
      .catch(() => resolve([])); // Handle errors gracefully

    // Set a timeout in case the process hangs.
    setTimeout(() => {
        pc.close();
        resolve(Array.from(ips));
    }, 1000);
  });
}

// Usage:
getWebRTCIPs().then(ips => {
  console.log('Discovered IPs:', ips);
  // A fingerprinting script would send this array to a server.
  // Example output: ['192.168.1.100', '74.125.140.127']
});
```