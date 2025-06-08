---
title: window.RTCPeerConnection
layout: default
---
# window.RTCPeerConnection
## Purpose
The core component of the WebRTC API. It enables the creation of a peer-to-peer connection between browsers to stream audio, video, and arbitrary data directly, without requiring a central server for data relay (after the initial connection setup).

## Fingerprinting Usage
`RTCPeerConnection` is one of the most potent browser fingerprinting surfaces, primarily due to its ability to reveal network-level information that is otherwise inaccessible to JavaScript.

*   **IP Address Leakage (VPN/Proxy Bypass):** This is the most significant use case. To establish a peer-to-peer connection, the browser must discover all possible network paths. It does this by gathering ICE (Interactive Connectivity Establishment) candidates, which involves querying STUN (Session Traversal Utilities for NAT) servers. This process reveals the user's **true public IP address** and **local network IP addresses** (e.g., `192.168.1.100`). This can effectively bypass proxies and many VPNs, as the WebRTC request goes directly to the STUN server, not through the proxy configured for HTTP traffic. A mismatch between the IP seen by the web server and the IP revealed by WebRTC is a strong signal of a proxy or bot.

*   **High-Entropy Network Configuration:** The list of ICE candidates itself is a source of high entropy. It can reveal:
    *   The number and type of network interfaces (Ethernet, Wi-Fi, VPN).
    *   The specific local IP addresses assigned to each interface.
    *   The presence and configuration of IPv6.
    *   In some cases, mDNS hostnames (`.local` addresses) can be exposed, which may contain unique identifiers or the user's computer name (e.g., `<uuid>.local`).

*   **Behavioral and Environmental Signals:**
    *   **Bot Environment Detection:** Bots running in cloud environments (AWS, GCP, Azure) will expose IP addresses from known datacenter ranges. This is a major red flag when the browser's user agent claims to be a standard consumer device.
    *   **Inconsistent Environment:** A browser spoofing a mobile user agent should not expose a typical `192.168.x.x` home router IP. It should reveal a carrier-grade NAT (CGNAT) IP. This inconsistency is a reliable detection vector.
    *   **API Tampering:** Naive bots may disable WebRTC by setting `window.RTCPeerConnection = undefined`. The absence of this API in a modern browser that should have it is easily detectable. More advanced bots may try to spoof the `onicecandidate` callback, but this can be detected by checking for malformed candidates or an inability to establish a test connection.

## Sample Code
This snippet demonstrates how to trigger ICE candidate gathering to harvest local and public IP addresses without requesting microphone or camera permissions.

```javascript
/**
 * Gathers all unique IP addresses found via WebRTC ICE candidates.
 * @returns {Promise<Set<string>>} A Promise that resolves with a Set of IP addresses.
 */
function getWebRTCIPs() {
  return new Promise((resolve, reject) => {
    try {
      const ips = new Set();
      // Use a public STUN server from Google.
      const pc = new RTCPeerConnection({ iceServers: [{ urls: "stun:stun.l.google.com:19302" }] });

      // The 'onicecandidate' event is triggered for each potential connection path found.
      pc.onicecandidate = (event) => {
        // The last candidate will be null to signal the end of gathering.
        if (!event || !event.candidate || !event.candidate.candidate) {
          // When gathering is complete, resolve the promise if IPs were found.
          if (ips.size > 0) {
            resolve(ips);
          }
          return;
        }

        // The candidate string contains the IP address.
        // Example: "candidate:842163049 1 udp 16777215 192.168.1.100 53475 typ srflx ..."
        const ipRegex = /([0-9]{1,3}(\.[0-9]{1,3}){3}|[a-f0-9]{1,4}(:[a-f0-9]{1,4}){7})/gi;
        const matches = event.candidate.candidate.match(ipRegex);

        if (matches) {
          matches.forEach(ip => ips.add(ip));
        }
      };

      // To trigger candidate gathering, we need to create a data channel and an offer.
      // This does not require any user permissions.
      pc.createDataChannel("");
      pc.createOffer()
        .then(offer => pc.setLocalDescription(offer))
        .catch(err => reject(err));

      // Set a timeout in case the process fails silently.
      setTimeout(() => {
        if (ips.size > 0) {
          resolve(ips);
        } else {
          // If no IPs are found after a timeout, it could indicate WebRTC is disabled or blocked.
          reject(new Error("WebRTC IP gathering timed out."));
        }
      }, 1000);

    } catch (e) {
      // RTCPeerConnection is not supported or is blocked.
      reject(e);
    }
  });
}

// Usage:
getWebRTCIPs()
  .then(ips => {
    console.log("Detected IPs:", Array.from(ips));
    // Example Output: ["192.168.1.100", "93.184.216.34"]
    // A bot detection script would compare these IPs with the server-side IP.
  })
  .catch(err => {
    console.error("Could not get WebRTC IPs:", err.message);
  });
```