---
title: window.RTCTrackEvent
layout: default
---
# window.RTCTrackEvent
## Purpose
Represents the `track` event, which fires on an `RTCPeerConnection` when a new `MediaStreamTrack` is received from a remote peer. This is fundamental for receiving and displaying remote audio or video streams in a WebRTC session.

## Fingerprinting Usage
The `RTCTrackEvent` is not a source of static entropy itself. Instead, its primary value is in **behavioral analysis and capability probing**. Anti-bot systems use it to verify that a browser has a complete and functional WebRTC media stack, which is complex for bots and modified headless browsers to emulate correctly.

1.  **Capability Probing:** An anti-bot system can initiate a silent, background WebRTC connection from a server to the client. The server then adds an audio or video track to the connection. A genuine browser will fire the `ontrack` event, delivering a valid `RTCTrackEvent`. The simple occurrence of this event is a high-confidence signal of a real browser.

2.  **Detecting Headless Browsers:** Standard headless browsers (like Puppeteer/Playwright) may not have the necessary media codecs or a full media pipeline enabled by default. They might successfully establish the data channel part of a WebRTC connection but will often fail to process an incoming media track, meaning the `ontrack` event never fires. A timeout on waiting for this event is a strong indicator of a bot.

3.  **API Inconsistencies:** Sophisticated bots may try to hook or spoof the `RTCPeerConnection` API to fake a successful connection. However, correctly emulating the `RTCTrackEvent` and the associated `MediaStreamTrack` object is difficult. Detection scripts can check for anomalies:
    *   Does the `event.track` object exist and have expected properties (`kind`, `id`, `readyState`)?
    *   Does the `track.readyState` transition correctly from `live` to `ended` when the connection is closed?
    *   Does calling methods on the received track, like `getSettings()`, return realistic values or throw errors?

In summary, `RTCTrackEvent` is the final step in a WebRTC media-capability test. Failure to trigger it, or triggering it with an inconsistent object, effectively distinguishes full-featured browsers from emulated or incomplete clients.

## Sample Code
This snippet demonstrates how an anti-bot script might test for the `ontrack` event. It assumes a server-side component is handling the WebRTC signaling (offer/answer exchange) to send a track to the client.

```javascript
function probeWebRtcMediaStack() {
  return new Promise((resolve, reject) => {
    let peerConnection;
    try {
      // Configuration can be empty for this test.
      peerConnection = new RTCPeerConnection();
    } catch (e) {
      // The constructor itself being blocked or failing is a major red flag.
      return reject({ reason: 'CONSTRUCTOR_FAILED' });
    }

    const detectionTimeout = setTimeout(() => {
      if (peerConnection.connectionState !== 'connected') {
        reject({ reason: 'TIMEOUT' });
        peerConnection.close();
      }
    }, 3000); // Set a 3-second timeout for the test.

    peerConnection.ontrack = (event) => {
      clearTimeout(detectionTimeout);
      // The event fired, a strong positive signal.
      // Further checks can be done on the track itself.
      if (event.track && event.track.kind === 'audio') {
        resolve({
          status: 'SUCCESS',
          trackId: event.track.id,
          readyState: event.track.readyState
        });
      } else {
        // Event fired, but the track is malformed or not what was expected.
        reject({ reason: 'MALFORMED_TRACK' });
      }
      peerConnection.close();
    };

    // In a real-world scenario, code here would exchange SDP with a server.
    // The server would create an offer and send a media track.
    // e.g., initiateSignaling(peerConnection);
    // This example focuses only on the client's 'ontrack' handler.
  });
}

// Example of running the probe
probeWebRtcMediaStack()
  .then(result => console.log('Probe Result: Genuine browser detected.', result))
  .catch(error => console.error('Probe Result: Bot or anomalous environment detected.', error));
```