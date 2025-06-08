---
title: window.requestAnimationFrame
layout: default
---
# window.requestAnimationFrame
## Purpose
`window.requestAnimationFrame()` (rAF) tells the browser that you wish to perform an animation and requests that the browser call a specified function to update an animation before the next repaint. It is the standard way to run animations efficiently in JavaScript, as it is synchronized with the browser's rendering loop.

## Fingerprinting Usage
`requestAnimationFrame` is a potent API for both passive hardware fingerprinting and active behavioral bot detection. Its timing characteristics are closely tied to the user's hardware and the browser's state, making it difficult to spoof accurately.

1.  **Hardware Refresh Rate:** The most significant source of entropy is the monitor's refresh rate. By measuring the time delta between consecutive rAF callbacks, a script can accurately determine the screen's refresh rate (e.g., 60Hz, 75Hz, 90Hz, 120Hz, 144Hz). The timestamp provided to the callback is a `DOMHighResTimeStamp` with microsecond precision.
    *   **Signal:** A 60Hz monitor will yield deltas of ~16.67ms. A 144Hz monitor will yield deltas of ~6.94ms. This is a stable hardware attribute.
    *   **Anomaly:** Headless browsers often run on virtual displays (like Xvfb) that default to 60Hz. A user agent claiming to be a high-end gaming machine but reporting a consistent 60Hz refresh rate can be a red flag.

2.  **Behavioral Throttling:** Modern browsers heavily throttle rAF in background tabs or minimized windows to conserve CPU and battery. Callbacks may be delayed or batched, often firing only once per second.
    *   **Detection:** A script can listen for the `visibilitychange` event. If the page's visibility state becomes `hidden`, it can check if rAF continues to fire at a high frequency (e.g., 60 times per second). If it does, it's a strong indicator of a bot or a simplified headless environment that does not correctly implement the page visibility lifecycle.

3.  **Timing Jitter and Precision:** On a real system, the time between rAF callbacks is not perfectly constant. It exhibits minor fluctuations ("jitter") due to OS scheduling, background processes, and rendering load. The timestamp itself has microsecond precision.
    *   **Detection:**
        *   **Unnatural Consistency:** A bot attempting to spoof rAF might return perfectly uniform timestamps (e.g., exactly 16.666666ms every time), which is unnatural for a real system.
        *   **Low Precision:** A spoofed environment might return integer timestamps or timestamps with only millisecond precision, failing to replicate the `DOMHighResTimeStamp` standard.
        *   **Headless Signature:** Some headless browser configurations do not fire rAF at all unless specific flags are enabled (e.g., `--run-all-compositor-stages-before-draw`), making its absence a simple detection vector.

## Sample Code
This snippet demonstrates how to measure the monitor's refresh rate, a primary fingerprinting vector from `requestAnimationFrame`.

```javascript
/**
 * Measures the display's refresh rate using requestAnimationFrame.
 * @returns {Promise<number>} A promise that resolves with the estimated refresh rate in Hz.
 */
function measureRefreshRate() {
  return new Promise(resolve => {
    const sampleCount = 100;
    const timestamps = [];
    let lastTimestamp = 0;

    function step(timestamp) {
      // The first frame's timestamp can be inconsistent, so we start measuring from the second frame.
      if (lastTimestamp > 0) {
        timestamps.push(timestamp - lastTimestamp);
      }
      lastTimestamp = timestamp;

      if (timestamps.length < sampleCount) {
        requestAnimationFrame(step);
      } else {
        // Calculate the average delta between frames
        const avgDelta = timestamps.reduce((a, b) => a + b, 0) / timestamps.length;
        
        // Calculate refresh rate (Hz = frames per second)
        const refreshRate = Math.round(1000 / avgDelta);
        resolve(refreshRate);
      }
    }

    requestAnimationFrame(step);
  });
}

// Usage:
measureRefreshRate().then(rate => {
  console.log(`Estimated Refresh Rate: ${rate} Hz`);
  // Anti-bot systems would send this value to a server for analysis.
  // Example rates: 60, 75, 90, 120, 144
});
```