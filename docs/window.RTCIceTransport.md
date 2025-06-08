---
title: window.RTCIceTransport
layout: default
---
# window.RTCIceTransport
## Purpose
`RTCIceTransport` is a low-level API within the WebRTC (Web Real-Time Communication) stack. Its core function is to manage the Interactive Connectivity Establishment (ICE) process, which involves discovering, negotiating, and managing network paths (ICE candidates) to establish a peer-to-peer connection.

## Fingerprinting Usage
`RTCIceTransport` itself is not directly instantiated but is the underlying mechanism for the most well-known and potent browser fingerprinting vector: the WebRTC Local IP Address Leak. Anti-bot systems heavily rely on analyzing the output of the ICE process.

1.  **Local IP Address Leakage**: The primary fingerprinting value comes from the ICE candidate gathering process, which is initiated by `RTCPeerConnection`. To find the most efficient path, the browser collects `host` candidates, which contain the device's local (private) IP addresses (e.g., `192.168.1.100`). This information is exceptionally valuable because:
    *   It bypasses proxies and VPNs, revealing the true internal network IP of the device.
    *   A device's local IP is often stable, providing a consistent identifier for return visits from the same local network.

2.  **Network Configuration Enumeration**: A user may have multiple network interfaces (e.g., Wi-Fi, Ethernet, virtual adapters for VPNs or virtualization software). The ICE process will enumerate candidates for *all* active interfaces. The resulting list of local IPs, their order, and their types creates a rich, high-entropy fingerprint of the user's network hardware and software configuration.

3.  **Bot & Headless Browser Detection**:
    *   **Predictable IPs**: Bots running in containerized environments (like Docker) often expose predictable or common default local IPs (e.g., from the `172.17.0.0/16` range). Detecting these specific IPs is a strong signal of a non-human environment.
    *   **Absence of Candidates**: Headless browsers (like Puppeteer or Playwright) may be configured to disable WebRTC or may run in environments with no network interfaces that produce `host` candidates. A complete failure to gather any `host` candidates is anomalous for a typical user device and can be flagged.
    *   **Inconsistent Environment**: An anti-bot script can compare the public IP (from the server request or a STUN server) with the local IP. If a user appears to be in a residential IP block but their browser exposes a datacenter-range local IP, it's a major red flag for a bot or proxy setup.

4.  **Behavioral Analysis**: The timing and success of the ICE candidate gathering process can be measured. Tampering with the `RTCPeerConnection` constructor or its methods to prevent IP leakage can be detected by checking if the API has been modified or if it throws non-standard errors.

## Sample Code
This code demonstrates how to initiate the ICE process to collect local IP addresses. It doesn't use `RTCIceTransport` directly but triggers the underlying mechanism it controls.

```javascript
/**
 * Gathers local IP addresses by initiating a WebRTC peer connection.
 * @returns {Promise<string[]>} A promise that resolves with an array of unique local IP addresses.
 */
function getLocalIpAddresses() {
  return new Promise((resolve) => {
    const ips = new Set();
    // RTCPeerConnection is the main entry point to WebRTC.
    // The empty configuration is sufficient to trigger local candidate gathering.
    const pc = new RTCPeerConnection({ iceServers: [] });

    // The 'onicecandidate' event is fired for each potential network path.
    pc.onicecandidate = (event) => {
      if (!event || !event.candidate || !event.candidate.candidate) {
        return;
      }
      
      // The candidate string contains the IP address. We look for host candidates.
      // Example candidate string: "candidate:1234 1 udp 2122260223 192.168.1.10 54321 typ host ..."
      const ipRegex = /(192\.168\.[0-9]{1,3}\.[0-9]{1,3}|10\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}|172\.(1[6-9]|2[0-9]|3[0-1])\.[0-9]{1,3}\.[0-9]{1,3})/g;
      const matches = event.candidate.candidate.match(ipRegex);

      if (matches) {
        matches.forEach(ip => ips.add(ip));
      }
    };

    // To trigger candidate gathering, we need to create a data channel
    // and then create an offer. This starts the SDP negotiation process.
    pc.createDataChannel('');
    pc.createOffer()
      .then(offer => pc.setLocalDescription(offer))
      .catch(err => console.error('Error creating WebRTC offer:', err));

    // Set a timeout to resolve the promise, as there's no definitive "end" event
    // for candidate gathering in all browsers.
    setTimeout(() => {
      pc.close();
      resolve(Array.from(ips));
    }, 500);
  });
}

// Usage:
getLocalIpAddresses().then(ips => {
  if (ips.length > 0) {
    console.log('Detected Local IPs:', ips);
    // A fingerprinting script would hash this array or send it to a server.
    // An anti-bot system would check these IPs against known bot signatures.
  } else {
    console.log('No local IPs found (WebRTC might be disabled or blocked).');
  }
});
```