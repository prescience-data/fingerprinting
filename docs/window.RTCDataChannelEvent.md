---
title: window.RTCDataChannelEvent
layout: default
---
# window.RTCDataChannelEvent
## Purpose
Represents the event that occurs when an `RTCDataChannel` is added to an `RTCPeerConnection`. This event is fired on the receiving peer when the remote peer initiates the creation of a data channel, signaling that a direct data communication path is ready.

## Fingerprinting Usage
The `RTCDataChannelEvent` itself has low entropy. Its primary value is not in the properties of the event object, but as a confirmation signal in a larger WebRTC stack integrity probe. Anti-bot systems use it to verify that the browser has a complete and unmodified WebRTC implementation.

1.  **WebRTC Stack Integrity:** Many bots and automation frameworks (like older versions of Puppeteer or Playwright) either disable WebRTC entirely or provide incomplete shims. A fingerprinting script can create two local `RTCPeerConnection` objects and attempt to connect them to each other. The successful firing of the `ondatachannel` event on the receiving peer is a high-confidence signal that the full WebRTC stack is present and functional, which is characteristic of a standard browser. Failure to trigger this event, or a timeout, strongly suggests a non-standard or manipulated environment.

2.  **Behavioral Telemetry:** The timing and behavior surrounding the event can be measured.
    *   **Handshake Latency:** The time elapsed between calling `createDataChannel()` on one peer and the `RTCDataChannelEvent` firing on the other can be measured. This latency can be influenced by the browser engine's implementation, CPU load, and OS scheduler, providing a subtle behavioral fingerprint.
    *   **Data Transfer Characteristics:** Once the channel is established (confirmed by this event), the script can send data packets and measure round-trip time (RTT) and throughput. Anomalies here (e.g., near-zero latency) can indicate that the "peers" are running within the same process in a sandboxed environment, rather than behaving like a real network connection.

3.  **Consistency Checks:** The properties of the `event.channel` object (`label`, `protocol`, `id`) must match what the initiating peer created. A poorly implemented browser shim might fail to propagate these properties correctly, revealing its presence.

## Sample Code
This snippet demonstrates a local peer-to-peer connection to probe the WebRTC stack. The successful firing of `ondatachannel` is the key signal.

```javascript
async function probeRtcDataChannel() {
  try {
    // Create two "peers" within the same browser context.
    const pc1 = new RTCPeerConnection();
    const pc2 = new RTCPeerConnection();

    // Exchange ICE candidates between peers.
    pc1.onicecandidate = e => e.candidate && pc2.addIceCandidate(e.candidate);
    pc2.onicecandidate = e => e.candidate && pc1.addIceCandidate(e.candidate);

    const promise = new Promise((resolve, reject) => {
      // Set a timeout to detect non-functional WebRTC stacks.
      const timeout = setTimeout(() => reject('RTCDataChannel probe timed out.'), 1000);

      // The key listener: if this fires, the WebRTC stack is likely genuine.
      pc2.ondatachannel = event => {
        clearTimeout(timeout);
        console.log('RTCDataChannelEvent received. High confidence signal.');
        console.log('Channel Label:', event.channel.label); // 'probe-channel'
        resolve(true);
      };
    });

    // Initiate the data channel on the first peer.
    pc1.createDataChannel('probe-channel');

    // Complete the SDP offer/answer handshake.
    const offer = await pc1.createOffer();
    await pc1.setLocalDescription(offer);
    await pc2.setRemoteDescription(offer);
    const answer = await pc2.createAnswer();
    await pc2.setLocalDescription(answer);
    await pc1.setRemoteDescription(answer);

    return promise;
  } catch (error) {
    console.error('RTCDataChannel probe failed:', error);
    return false; // Failure is a strong signal of a blocked or modified environment.
  }
}

// Run the probe
probeRtcDataChannel().then(isGenuine => {
  if (isGenuine) {
    // This browser passed a significant anti-bot check.
  }
});
```