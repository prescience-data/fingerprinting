---
title: window.PerformanceElementTiming
layout: default
---
# window.PerformanceElementTiming
## Purpose
Part of the Performance API, `PerformanceElementTiming` provides high-precision timing data for the rendering of specific, developer-marked elements (e.g., images, text blocks) on a web page. It measures the time from navigation start until the element is first painted to the screen.

## Fingerprinting Usage
The primary value of this API for fingerprinting lies in its ability to expose performance characteristics of the user's underlying hardware and software stack. The time it takes to render an element is not a constant; it is a function of the CPU, GPU, OS graphics subsystem, and browser rendering engine.

*   **Hardware Fingerprinting:** The `renderTime` metric is a high-entropy signal. Two different devices, even with the same browser version, will produce slightly different `renderTime` values for the same element due to variations in GPU model, CPU speed, system load, and driver versions. Aggregating timings for several elements creates a robust hardware-level fingerprint.

*   **Bot Detection Anomalies:**
    *   **Headless Environments:** Headless browsers like Puppeteer or Playwright often run in virtualized cloud environments (e.g., AWS EC2, Docker). These environments have standardized and often non-consumer-grade GPUs (or software-emulated GPUs). This results in `renderTime` values that are either suspiciously fast, unusually slow, or unnaturally consistent across many bot instances, creating a detectable signature that differs from the wide, noisy distribution of real user hardware.
    *   **Inconsistent Fingerprints:** A bot may spoof its User-Agent to claim it's a high-end Windows machine with an NVIDIA GPU. However, if its `renderTime` values are consistent with a Linux server running a software renderer, this contradiction is a strong indicator of deception.
    *   **Lack of Rendering:** Simpler bots or HTTP clients that do not execute JavaScript or have a rendering pipeline will fail to produce any `PerformanceElementTiming` entries at all. The *absence* of this data, when expected, is a signal.

## Sample Code
This snippet demonstrates how a script would collect rendering time for a critical image. This data would then be sent to a server for analysis.

**HTML:**
```html
<!-- An element is marked for timing measurement -->
<img src="/images/hero-banner.jpg" elementtiming="hero-image" alt="Hero Banner">
```

**JavaScript:**
```javascript
// Array to hold performance signals for server-side analysis
const fingerprintSignals = {};

// Check for API support (a low-entropy signal itself)
if ('PerformanceObserver' in window && 'PerformanceElementTiming' in window) {
  const observer = new PerformanceObserver((entryList) => {
    for (const entry of entryList.getEntries()) {
      // Collect the renderTime for the specific element.
      // This value is highly sensitive to the user's hardware (CPU/GPU).
      fingerprintSignals[entry.name] = {
        renderTime: entry.renderTime,
        loadTime: entry.loadTime,
        identifier: entry.identifier
      };

      console.log('Collected timing for:', entry.name, 'Render Time:', entry.renderTime);
      
      // In a real scenario, this data is batched and sent to a server
      // for comparison against known bot and real user profiles.
      // sendFingerprintData(fingerprintSignals);
    }
  });

  // Start observing for elements with the 'elementtiming' attribute.
  // 'buffered: true' ensures we get entries that occurred before this script ran.
  observer.observe({ type: 'element-timing', buffered: true });
}
```