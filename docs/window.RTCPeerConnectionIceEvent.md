---
title: window.RTCPeerConnectionIceEvent
layout: default
---
# window.RTCPeerConnectionIceEvent
## Purpose
Represents an event that occurs when the local WebRTC ICE (Interactive Connectivity Establishment) agent has gathered a network candidate. These candidates are potential addresses (IP and port) used to establish a peer-to-peer connection for real-time communication.

## Fingerprinting Usage
The `icecandidate` event, which carries an `RTCPeerConnectionIceEvent` object, is a cornerstone of advanced browser fingerprinting and bot detection. Its utility comes from exposing detailed network configuration information that is difficult to spoof consistently.

1.  **IP Address Revelation (WebRTC Leak):** This is the most significant use. The `event.candidate.candidate` string can contain:
    *   **Local IP Address:** The user's private IP address on their local network (e.g., `192.168.1.10`). This provides high entropy about the user's LAN configuration.
    *   **Public IP Address:** The user's public-facing IP address, discovered via a STUN server. This is powerful because it can bypass proxies and VPNs, revealing the user's true IP address. A mismatch between the IP seen by the server from the HTTP request and the IP revealed by WebRTC is a strong signal of proxy usage.

2.  **Network Topology Enumeration:** The number, type (`host`, `srflx`, `relay`), and order of candidates generated reveal information about the user's network environment (e.g., behind a simple NAT, a corporate firewall, or using a TURN relay). Bots running in cloud environments often have a very simple and identifiable network signature (e.g., only one public `host` candidate).

3.  **Behavioral and Consistency Signals:**
    *   **Timing:** The time elapsed between calling `createOffer()` and the firing of the first `icecandidate` event can be a behavioral biometric. Automated browsers may exhibit unnaturally fast or consistent timings compared to real users on diverse hardware and networks.
    *   **Spoofing Detection:** Anti-bot systems can check for inconsistencies. For example, if a browser's user-agent reports macOS, but no `.local` mDNS candidates are generated (which is typical for macOS/Bonjour), it suggests spoofing.
    *   **Headless Detection:** Headless browsers like Puppeteer or Playwright may have WebRTC disabled by default or may require specific flags (`--no-sandbox`) to function, leading to predictable failures or an absence of candidates. If candidates are generated, their characteristics might be typical of a datacenter IP range, immediately flagging the session as non-human.

## Sample Code
This snippet demonstrates how to initiate an RTCPeerConnection to trigger the `icecandidate` event and parse the resulting candidate string to extract IP addresses.

```javascript
async function getWebRTCIPs() {
  const ips = new Set();
  // Use a public STUN server to facilitate candidate gathering.
  const pc = new RTCPeerConnection({
    iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
  });

  // The 'icecandidate' event fires for each discovered candidate.
  pc.onicecandidate = (event) => {
    // The event is RTCPeerConnectionIceEvent.
    // The last event will have a null candidate.
    if (!event || !event.candidate || !event.candidate.candidate) {
      return;
    }

    // The candidate string contains the IP address.
    // Example: "candidate:842163049 1 udp 2122260223 192.168.1.10 57005 typ host ..."
    const candidateString = event.candidate.candidate;
    
    // Regex to find IP addresses in the candidate string.
    const ipRegex = /([0-9]{1,3}(\.[0-9]{1,3}){3}|[a-f0-9]{1,4}(:[a-f0-9]{1,4}){7})/;
    const match = ipRegex.exec(candidateString);
    
    if (match) {
      ips.add(match[1]);
      console.log('Found IP:', match[1]);
    }
  };

  // This is a necessary step to trigger the ICE gathering process.
  pc.createDataChannel('');
  await pc.createOffer()
    .then(offer => pc.setLocalDescription(offer))
    .catch(err => console.error('Error creating offer:', err));

  // Note: This is an asynchronous process. The IPs are found over time.
  // A real fingerprinting script would wait a short period (e.g., 2 seconds)
  // to collect all candidates before closing the connection.
}

getWebRTCIPs();
```