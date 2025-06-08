---
title: window.RTCEncodedAudioFrame
layout: default
---
# window.RTCEncodedAudioFrame
## Purpose
Represents a single frame of encoded audio data within a WebRTC stream. It is a core component of the WebRTC Insertable Streams API, which allows developers to intercept and manipulate encoded media frames before they are sent or after they are received.

## Fingerprinting Usage
`RTCEncodedAudioFrame` provides a high-entropy fingerprinting surface and is used in sophisticated bot detection by testing the integrity and characteristics of the browser's media processing pipeline.

1.  **Codec Fingerprinting**: The primary use is to fingerprint the browser's built-in audio codec (e.g., Opus). A script can generate a standardized raw audio signal (like a sine wave), feed it into the WebRTC stack, and intercept the resulting `RTCEncodedAudioFrame`. The binary content of the `data` property is then hashed. This hash is highly specific because the output of a codec depends on:
    *   The exact version and implementation of the codec in the browser.
    *   The underlying Operating System's audio stack.
    *   CPU architecture and specific instruction sets used for optimization.
    *   Subtle differences in how V8 or the browser's media engine handles the data.
    This produces a stable and high-entropy identifier, similar to AudioContext fingerprinting but at the encoded layer.

2.  **API Availability and Integrity**: The mere existence and correct functioning of the `RTCEncodedAudioFrame` constructor and the associated Insertable Streams API (`RTCRtpSender.createEncodedStreams`) is a signal. Many automation frameworks and older browsers lack support for this modern API. A script can attempt to set up an encoded transform stream; if it fails or throws unexpected errors, it indicates a non-standard or manipulated environment.

3.  **Behavioral and Timing Analysis**: The process of encoding an audio frame is computationally non-trivial. Anti-bot systems can measure the time it takes for a frame to pass through the encoding pipeline.
    *   Bots that fake the media pipeline might return data too quickly.
    *   Emulated or virtualized environments might exhibit significantly slower and more jittery timing than a real user's machine.
    Analyzing the `timestamp` and metadata of consecutive frames can also reveal unnatural patterns inconsistent with a real microphone and codec.

4.  **Probing for Headless Environments**: Sophisticated headless browsers like Puppeteer or Playwright may support WebRTC, but their implementation can have subtle differences. Probing for specific byte patterns or metadata in the encoded frame can help distinguish their default audio stack (e.g., a virtual audio device) from that of a genuine user device.

## Sample Code
This snippet demonstrates the logic of capturing an encoded audio frame and hashing its data to generate a fingerprint. It requires user permission for the microphone.

```javascript
async function getAudioCodecFingerprint() {
  try {
    // 1. Check for API support, a fingerprinting signal in itself.
    if (typeof window.RTCRtpSender === 'undefined' ||
        typeof RTCRtpSender.prototype.createEncodedStreams !== 'function') {
      return 'Error: Insertable Streams not supported';
    }

    const stream = await navigator.mediaDevices.getUserMedia({ audio: true, video: false });
    const track = stream.getAudioTracks()[0];

    // 2. Set up a peer connection to activate the encoding pipeline.
    const pc = new RTCPeerConnection();
    const sender = pc.addTrack(track, stream);

    // 3. Access the encoded streams. This is the key API.
    const { readable, writable } = sender.createEncodedStreams();

    const fingerprintPromise = new Promise((resolve, reject) => {
      const transformStream = new TransformStream({
        transform(encodedFrame, controller) {
          // 4. Intercept the first encoded frame.
          const frameData = new Uint8Array(encodedFrame.data);

          // 5. Hash the binary data of the encoded frame.
          // The resulting hash is a high-entropy fingerprint.
          crypto.subtle.digest('SHA-256', frameData)
            .then(hashBuffer => {
              const hashArray = Array.from(new Uint8Array(hashBuffer));
              const hashHex = hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
              resolve(hashHex);
            })
            .catch(reject);

          // Clean up immediately after capturing one frame.
          track.stop();
          pc.close();
          controller.terminate();
        }
      });

      readable.pipeThrough(transformStream).pipeTo(writable).catch(err => {});
    });

    return await fingerprintPromise;

  } catch (error) {
    // Errors (e.g., no permissions, no device) are also signals.
    return `Error: ${error.name}`;
  }
}

// Usage:
// getAudioCodecFingerprint().then(fp => console.log('Audio Codec Fingerprint:', fp));
```