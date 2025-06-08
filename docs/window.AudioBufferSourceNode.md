---
title: window.AudioBufferSourceNode
layout: default
---
# window.AudioBufferSourceNode
## Purpose
An interface of the Web Audio API that represents an audio source consisting of in-memory audio data stored in an `AudioBuffer`. It is a foundational node used to play, process, and analyze pre-decoded audio within an `AudioContext` graph.

## Fingerprinting Usage
`AudioBufferSourceNode` is not a source of entropy itself, but it is a critical component in the widely used **AudioContext Fingerprinting** technique. This method exploits subtle differences in how audio is processed across different machines and browsers.

The process works as follows:
1.  An `OfflineAudioContext` is created to process audio without audible output, making it fast and invisible to the user.
2.  An `AudioBuffer` is programmatically filled with a known signal (e.g., a sine wave or white noise).
3.  An `AudioBufferSourceNode` is created and fed this buffer, providing a consistent input signal.
4.  The source node is connected to one or more processing nodes, most commonly a `DynamicsCompressorNode`, which has complex internal floating-point calculations.
5.  The output of the processing graph is rendered to a final buffer.
6.  A hash (e.g., SHA-256) or a simple sum of the samples in the final buffer is calculated.

**Entropy & Anomaly Sources:**
*   **Hardware/Driver Variation:** The primary source of entropy is the final rendered buffer. Minor variations in CPU architecture, operating system audio libraries, and audio hardware drivers lead to minute differences in floating-point calculations. This results in a highly stable and unique fingerprint for a specific browser/OS/hardware combination.
*   **Browser Engine Implementation:** Different browser versions or rendering engines (Blink, Gecko, WebKit) implement the audio processing algorithms differently, yielding distinct fingerprints.
*   **Bot Detection:**
    *   **Null/Zero Output:** Headless browsers or bot environments lacking a proper audio stack may fail to process the audio, resulting in a null, empty, or zero-filled buffer. This is a strong bot signal.
    *   **Known Signatures:** Some automation frameworks (e.g., older versions of Puppeteer or Playwright) produce a standardized, predictable fingerprint because they use a software-based renderer without hardware-specific variations. These known values can be blocklisted.
    *   **Inconsistent Behavior:** If a bot attempts to spoof the fingerprint by hooking methods like `getChannelData`, it may return a static value. A detection script can run the test multiple times with slightly different parameters (e.g., changing the compressor's threshold) and check if the output changes in a mathematically plausible way. A static or incorrect response indicates manipulation.

## Sample Code
This snippet demonstrates the core logic of AudioContext fingerprinting using an `AudioBufferSourceNode` and an `OfflineAudioContext`.

```javascript
async function getAudioFingerprint() {
  try {
    // Use OfflineAudioContext for silent, fast rendering.
    const context = new OfflineAudioContext(1, 44100, 44100);

    // Create a buffer to hold a simple generated signal.
    const buffer = context.createBuffer(1, context.sampleRate, context.sampleRate);
    const data = buffer.getChannelData(0);
    for (let i = 0; i < data.length; i++) {
      // Fill with a simple sine wave.
      data[i] = Math.sin(2 * Math.PI * 440 * i / context.sampleRate);
    }

    // Create the source node and assign the buffer.
    const source = context.createBufferSource();
    source.buffer = buffer;

    // Use a DynamicsCompressorNode as its processing is sensitive to implementation.
    const compressor = context.createDynamicsCompressor();
    compressor.threshold.setValueAtTime(-50, context.currentTime);
    compressor.knee.setValueAtTime(40, context.currentTime);
    // ... other compressor settings

    // Build the audio graph: source -> compressor -> destination
    source.connect(compressor);
    compressor.connect(context.destination);

    source.start(0);
    const renderedBuffer = await context.startRendering();

    // Extract the final processed samples.
    const finalSamples = renderedBuffer.getChannelData(0);

    // Calculate a simple sum as the fingerprint (a real implementation would use a hash).
    let fingerprint = 0;
    for (let i = 0; i < finalSamples.length; i++) {
      fingerprint += Math.abs(finalSamples[i]);
    }

    // A zero or very low value can indicate a bot environment.
    if (fingerprint === 0) {
        return "Error: Audio processing failed.";
    }

    return fingerprint.toString(); // e.g., "124.0434785541147"
  } catch (e) {
    // Errors during context creation are a strong bot/privacy signal.
    return "Error: " + e.message;
  }
}

// Usage:
getAudioFingerprint().then(fp => console.log("Audio Fingerprint:", fp));
```