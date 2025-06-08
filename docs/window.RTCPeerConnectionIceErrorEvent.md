---
title: window.RTCPeerConnectionIceErrorEvent
layout: default
---
# window.RTCPeerConnectionIceErrorEvent
## Purpose
Represents an event dispatched on an `RTCPeerConnection` object when an error occurs during the Interactive Connectivity Establishment (ICE) candidate gathering process. This process is essential for discovering network paths for WebRTC connections.

## Fingerprinting Usage
While the primary fingerprinting vector in WebRTC is the leakage of local and public IP addresses via successful ICE candidate gathering, the `RTCPeerConnectionIceErrorEvent` serves as a powerful secondary signal for detecting bots and evasive configurations. Its usage is centered on anomaly and behavior detection.

*   **Detecting Evasion:** Many bots or privacy-focused browser configurations attempt to disable WebRTC or block connections to STUN servers to prevent IP address leakage. This interference often prevents the ICE process from completing successfully, triggering an `RTCPeerConnectionIceErrorEvent`. The occurrence of this event, especially a timeout, is a strong signal that something is actively blocking standard WebRTC functionality.

*   **Error Code Analysis:** The `errorCode` property of the event provides specific, valuable entropy. For example, an error code of `701` typically indicates that a STUN or TURN server was unreachable or timed out. Anti-bot systems can correlate a high incidence of `701` errors with users behind restrictive firewalls or, more commonly, bots using proxies that block the necessary UDP traffic.

*   **Behavioral Signal:** A real browser on a typical network will almost always successfully gather at least a local (`host`) ICE candidate. A fingerprinting script can set a short timeout. If no `icecandidate` events fire and an `icecandidateerror` event fires instead, it strongly suggests a non-standard or manipulated environment. The timing and sequence of events (or lack thereof) become part of the behavioral fingerprint.

*   **Consistency Checks:** The specific `errorText` and `errorCode` can vary between browser vendors, versions, and operating systems for the same underlying network issue. While this provides some entropy, its main value is in detecting inconsistencies. A browser that reports a Chrome user-agent but produces Firefox-specific ICE error messages is clearly spoofed.

## Sample Code
This snippet attempts to initiate ICE gathering and listens for errors. A bot blocking STUN requests would likely trigger the `icecandidateerror` listener.

```javascript
function checkWebRTCBehavior() {
  return new Promise(resolve => {
    const pc = new RTCPeerConnection({
      iceServers: [{
        urls: 'stun:stun.l.google.com:19302'
      }]
    });

    // Listener for errors during ICE gathering
    pc.onicecandidateerror = (event) => {
      // This is a strong signal of a bot or restrictive network.
      // Real browsers on standard networks rarely trigger this.
      const botSignal = {
        isBot: true,
        reason: 'ICE_CANDIDATE_ERROR',
        errorCode: event.errorCode,
        errorText: event.errorText,
        url: event.url
      };
      // In a real scenario, this data would be sent for analysis.
      console.warn('Potential Bot Detected:', botSignal);
      resolve(botSignal);
      pc.close();
    };

    // Listener for successful candidates
    pc.onicecandidate = (event) => {
      // A null candidate means gathering is complete.
      if (!event.candidate) {
        console.log('ICE gathering completed successfully.');
        resolve({ isBot: false, reason: 'ICE_SUCCESS' });
        pc.close();
      }
    };
    
    // Set a timeout to catch environments where WebRTC is completely broken
    // and neither success nor error events fire.
    setTimeout(() => {
        if (pc.connectionState !== 'closed') {
            console.warn('Potential Bot Detected: ICE process timed out.');
            resolve({ isBot: true, reason: 'ICE_TIMEOUT' });
            pc.close();
        }
    }, 2000);

    // Trigger the ICE gathering process
    pc.createDataChannel('');
    pc.createOffer()
      .then(offer => pc.setLocalDescription(offer))
      .catch(err => {
        // An error here can also indicate a manipulated environment.
        resolve({ isBot: true, reason: 'WEBRTC_SETUP_FAILED' });
      });
  });
}

checkWebRTCBehavior().then(result => {
  // Anti-bot logic would analyze this result.
});
```