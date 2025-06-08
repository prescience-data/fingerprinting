---
title: window.DynamicsCompressorNode
layout: default
---
# window.DynamicsCompressorNode
## Purpose
The `DynamicsCompressorNode` is an interface of the Web Audio API. Its core function is to apply dynamic range compression to an audio signal, lowering the volume of the loudest parts and raising the volume of the quietest parts.

## Fingerprinting Usage
The `DynamicsCompressorNode` is a cornerstone of the highly effective "Audio Fingerprint" technique. The fingerprint is not derived from the API's existence but from the unique output it produces when processing a standardized audio signal.

The process involves creating an `OfflineAudioContext` (which processes audio silently and as fast as possible), generating a simple waveform with an `OscillatorNode`, passing it through a `DynamicsCompressorNode` with specific parameters, and then capturing the resulting audio buffer.

The key sources of entropy are:
1.  **Hardware Differences:** The underlying CPU and audio hardware perform floating-point calculations with minute variations. The complex, non-linear math of the compression algorithm amplifies these tiny differences, resulting in a distinct output buffer.
2.  **Software Stack:** The specific implementation of the compression algorithm within the browser (e.g., Chrome's Blink engine), combined with variations in the operating system's audio libraries and drivers, contributes significantly to the final output.
3.  **Inconsistency Detection:** Bots often struggle to replicate this fingerprint accurately.
    *   A bot might claim to be Chrome on Windows via its User-Agent, but produce an audio fingerprint identical to Firefox on Linux, revealing the spoofing attempt.
    *   Headless browsers or sandboxed environments may lack a complete Web Audio API implementation, causing the test to fail or return a null/zero value—a strong signal of automation.
    *   Anti-fingerprinting tools that add random noise or return a constant value can be detected by analyzing the statistical properties of the output.

The final fingerprint is typically a hash (e.g., MurmurHash3) of the rendered `AudioBuffer`'s sample data.

## Sample Code
This code generates an audio fingerprint by processing a sine wave through a compressor and summing the resulting sample data.

```javascript
async function getAudioFingerprint() {
  try {
    // Use OfflineAudioContext to render audio silently and quickly.
    const audioCtx = new (window.OfflineAudioContext || window.webkitOfflineAudioContext)(1, 44100, 44100);

    // Create a simple oscillator to generate a predictable sound wave.
    const oscillator = audioCtx.createOscillator();
    oscillator.type = 'triangle';
    oscillator.frequency.setValueAtTime(10000, audioCtx.currentTime);

    // Create the compressor node. The specific parameters are crucial for consistency.
    const compressor = audioCtx.createDynamicsCompressor();
    compressor.threshold.setValueAtTime(-50, audioCtx.currentTime);
    compressor.knee.setValueAtTime(40, audioCtx.currentTime);
    compressor.ratio.setValueAtTime(12, audioCtx.currentTime);
    compressor.attack.setValueAtTime(0, audioCtx.currentTime);
    compressor.release.setValueAtTime(0.25, audioCtx.currentTime);

    // Connect the nodes: Oscillator -> Compressor -> Destination
    oscillator.connect(compressor);
    compressor.connect(audioCtx.destination);

    // Start the oscillator and render the audio graph.
    oscillator.start(0);
    const renderedBuffer = await audioCtx.startRendering();

    // Extract the raw audio data from the buffer.
    const channelData = renderedBuffer.getChannelData(0);

    // Calculate a simple fingerprint by summing the samples.
    // Real-world libraries use a more robust hashing algorithm.
    let fingerprint = 0;
    for (let i = 0; i < channelData.length; i++) {
      fingerprint += Math.abs(channelData[i]);
    }

    return fingerprint;
  } catch (error) {
    // If the API is blocked or fails, this is a signal in itself.
    return null;
  }
}

// Usage:
getAudioFingerprint().then(fp => {
  console.log('Audio Fingerprint:', fp);
  // Example output: 124.04347527516074 (This value will be consistent on the same machine/browser but vary across different ones)
});
```