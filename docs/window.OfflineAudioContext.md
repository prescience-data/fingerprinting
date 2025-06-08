---
title: window.OfflineAudioContext
layout: default
---
# window.OfflineAudioContext
## Purpose
`OfflineAudioContext` is a part of the Web Audio API used for processing and rendering an audio graph non-real-time. Unlike the standard `AudioContext` which streams to a device (like speakers), `OfflineAudioContext` renders the audio as fast as the CPU allows into an in-memory `AudioBuffer`, making it ideal for tasks like exporting audio files.

## Fingerprinting Usage
`OfflineAudioContext` is a high-entropy source for browser fingerprinting, commonly known as "Audio Fingerprinting". The fingerprint is derived from the subtle variations in how different machines process a standardized audio signal.

The core principle is that the final rendered `AudioBuffer` is the result of complex floating-point calculations. These calculations are sensitive to the underlying hardware and software stack, including:

*   **CPU Architecture:** Different CPUs (e.g., Intel, AMD, Apple M-series) have minor differences in their Floating-Point Units (FPUs) and instruction sets (SSE, AVX).
*   **Operating System:** The OS-level audio and math libraries can have distinct implementations that affect the final output.
*   **Browser Engine:** The specific version of the browser's audio engine (e.g., V8/Blink's implementation) can introduce variations. Small changes in the implementation of audio nodes, like a `DynamicsCompressorNode`, can significantly alter the output.

A fingerprinting script will:
1.  Create an `OfflineAudioContext` with a fixed sample rate.
2.  Construct a specific audio graph, typically involving an `OscillatorNode` to generate a tone and a `DynamicsCompressorNode` to apply complex, non-linear processing. The compressor is key as it amplifies minuscule input variations into more detectable output differences.
3.  Render the audio to a buffer.
4.  Calculate a hash (e.g., SHA-256) or a simple checksum of the raw floating-point data (`Float32Array`) in the resulting `AudioBuffer`.

This final hash is a highly stable and unique identifier for the user's device and software combination.

For bot detection, this technique is powerful because:
*   **Headless Inconsistency:** Headless browsers (like Puppeteer or Playwright) may use different audio processing backends than a standard GUI browser, resulting in a different audio fingerprint than the one expected for the claimed User-Agent.
*   **Spoofing Detection:** A bot attempting to spoof this API by overriding it or returning a static value will produce a fingerprint that is either constant (easily blacklisted) or fails to execute, which is a strong signal of a non-human environment.
*   **Environment Gaps:** Minimalist bot environments may not implement the Web Audio API at all, causing the code to throw an error.

## Sample Code
This snippet demonstrates how to generate a fingerprint by processing a simple audio signal and summing the resulting buffer data. A real-world implementation would use a more robust hashing algorithm.

```javascript
async function getAudioFingerprint() {
  return new Promise((resolve, reject) => {
    try {
      // Use a fixed sample rate and buffer length for consistency.
      const audioCtx = new (window.OfflineAudioContext || window.webkitOfflineAudioContext)(1, 44100, 44100);

      // Create an oscillator to generate a sound wave.
      const oscillator = audioCtx.createOscillator();
      oscillator.type = 'triangle';
      oscillator.frequency.setValueAtTime(10000, audioCtx.currentTime);

      // Create a compressor to process the sound. Its complex math amplifies variations.
      const compressor = audioCtx.createDynamicsCompressor();
      compressor.threshold.setValueAtTime(-50, audioCtx.currentTime);
      compressor.knee.setValueAtTime(40, audioCtx.currentTime);
      compressor.ratio.setValueAtTime(12, audioCtx.currentTime);
      compressor.attack.setValueAtTime(0, audioCtx.currentTime);
      compressor.release.setValueAtTime(0.25, audioCtx.currentTime);

      // Connect the nodes: oscillator -> compressor -> destination
      oscillator.connect(compressor);
      compressor.connect(audioCtx.destination);

      oscillator.start(0);
      audioCtx.startRendering();

      audioCtx.oncomplete = (event) => {
        const renderedBuffer = event.renderedBuffer;
        // Get the raw PCM data from the first channel.
        const channelData = renderedBuffer.getChannelData(0);
        
        // Calculate a simple sum of the buffer's values as the fingerprint.
        // A real library would use a cryptographic hash.
        let fingerprint = 0;
        for (let i = 0; i < channelData.length; i++) {
          fingerprint += Math.abs(channelData[i]);
        }
        resolve(fingerprint);
      };
    } catch (error) {
      // If the API is not supported or blocked, it's a signal itself.
      reject('AudioContext fingerprinting failed: ' + error);
    }
  });
}

// Usage:
getAudioFingerprint()
  .then(fingerprint => {
    console.log('Audio Fingerprint:', fingerprint);
    // Example output: 124.04347527516074
  })
  .catch(error => {
    console.error(error);
  });
```