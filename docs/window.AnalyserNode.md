---
title: window.AnalyserNode
layout: default
---
# window.AnalyserNode
## Purpose
The `AnalyserNode` is a component of the Web Audio API that provides real-time frequency and time-domain analysis information. Its primary intended use is for data analysis and creating audio visualizations.

## Fingerprinting Usage
`AnalyserNode` is a cornerstone of a powerful fingerprinting technique known as **Audio Fingerprinting** or **AudioContext Fingerprinting**. The method exploits the fact that the final processed audio data is not identical across different machines, even when the input signal is exactly the same.

This technique works by generating a known audio signal (e.g., a sine or triangle wave from an `OscillatorNode`) and passing it through an `AnalyserNode`. The resulting frequency data is then captured and summed or hashed to produce a fingerprint.

**Entropy Sources & Anomalies:**
1.  **Hardware Stack:** The most significant source of entropy comes from the unique combination of a user's hardware (CPU, audio card) and software (OS audio stack, drivers). Digital signal processing (DSP) algorithms performed by the browser can have minute floating-point variations depending on the underlying architecture.
2.  **Browser Implementation:** Subtle differences in how different browser versions or vendors implement the Web Audio API's DSP functions can contribute to the fingerprint.
3.  **Bot Detection:** This is a highly effective bot detection signal.
    *   **Headless Browsers:** Headless environments like Puppeteer or Playwright often lack a real audio device or use a "null" audio output. When the audio analysis is run, the `AnalyserNode` may return a constant, predictable value (e.g., an array of all zeros, `-Infinity`, or a fixed pattern), which is a strong indicator of a non-human user.
    *   **Spoofing Difficulty:** While bots can attempt to spoof this API by returning pre-computed values, it is extremely difficult to perfectly mimic the subtle noise and floating-point inaccuracies of a specific, legitimate hardware/OS/browser combination. An unnaturally "clean" or known-bad value is a reliable bot signal.

The resulting fingerprint is highly stable for a given user and device, making it a valuable data point for both unique user identification and bot detection.

## Sample Code
This snippet demonstrates the core logic of Audio Fingerprinting using an `OfflineAudioContext` to ensure a sterile, reproducible test environment, free from system sounds or microphone input.

```javascript
/**
 * Generates a fingerprint based on the device's audio processing stack.
 * @returns {Promise<number>} A numeric hash derived from the audio data.
 */
async function getAudioFingerprint() {
  try {
    // Use OfflineAudioContext for a consistent, isolated environment.
    // Parameters are (channels, length, sampleRate).
    const audioCtx = new (window.OfflineAudioContext || window.webkitOfflineAudioContext)(1, 44100, 44100);

    // Create a simple oscillator to generate a known sound wave.
    const oscillator = audioCtx.createOscillator();
    oscillator.type = 'triangle';
    oscillator.frequency.setValueAtTime(10000, audioCtx.currentTime);

    // Create an analyser node to process the oscillator's output.
    const analyser = audioCtx.createAnalyser();
    analyser.fftSize = 1024;

    // Connect the nodes: Oscillator -> Analyser -> Destination
    oscillator.connect(analyser);
    analyser.connect(audioCtx.destination);

    oscillator.start(0);
    
    // Begin rendering the audio. This is an async operation.
    const renderedBuffer = await audioCtx.startRendering();

    // Get the frequency data from the analyser.
    const frequencyData = new Float32Array(analyser.frequencyBinCount);
    analyser.getFloatFrequencyData(frequencyData);

    // Sum the frequency data to create a simple numeric fingerprint.
    // Real-world implementations would use a more robust hashing algorithm.
    let fingerprint = 0;
    for (let i = 0; i < frequencyData.length; i++) {
      fingerprint += Math.abs(frequencyData[i]);
    }

    return fingerprint;
  } catch (e) {
    // An error may indicate a headless environment or disabled API.
    return -1; // Return a specific value for error cases.
  }
}

// Usage:
getAudioFingerprint().then(fingerprint => {
  console.log('Audio Fingerprint:', fingerprint);
  // Example output on a real machine: 142.09918212890625
  // Example output in some headless browsers: 0 or -1
});
```