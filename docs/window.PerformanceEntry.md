---
title: window.PerformanceEntry
layout: default
---
# window.PerformanceEntry
## Purpose
`PerformanceEntry` is the base interface for individual performance metrics collected by the Performance Timeline API. It represents a single, discrete event in the browser's performance timeline, such as a resource fetch, a navigation event, or a custom measurement.

## Fingerprinting Usage
`PerformanceEntry` and its associated APIs are a high-entropy source for fingerprinting and a critical tool for bot detection. The timings they provide are high-resolution (microseconds) and directly reflect the performance characteristics of the underlying hardware, OS, and browser engine.

1.  **Hardware & Environment Profiling:** The core usage is to measure the execution time of specific tasks. Anti-bot scripts create custom `PerformanceMark`s before and after a computationally intensive JavaScript or rendering task (e.g., complex math, canvas operations). The resulting `duration` from `performance.measure` is a powerful fingerprinting metric that varies significantly across different CPUs, GPUs, OS schedulers, and even browser versions. A cluster of bots running on identical cloud hardware will often produce nearly identical timings.

2.  **High-Precision Timer Detection:** The `startTime` and `duration` properties are derived from `performance.now()`, which is a high-resolution monotonic clock. The precision of this timer itself can be a fingerprinting vector. Furthermore, bots attempting to spoof or disable this API can be detected. Anti-bot scripts check for anomalies such as:
    *   Non-increasing timestamps between consecutive entries.
    *   Durations that are exactly `0`, negative, or suspiciously round integers (e.g., `100.000000000000`), suggesting tampering with `performance.now()` or the `PerformanceEntry` prototype.
    *   Discrepancies between `performance.now()` and lower-resolution timers like `Date.now()`.

3.  **Behavioral Analysis:** The sequence and timing of `PerformanceEntry` objects of type `event` (Event Timing API) can reveal non-human behavior. A bot's interaction events (e.g., `pointerdown`, `click`) often occur with programmatic speed and consistency, whereas human interactions have natural jitter and variance in their timing.

4.  **Network Fingerprinting:** `PerformanceResourceTiming` entries provide a detailed breakdown of network requests (DNS, TCP, TLS, TTFB). This data can be used to:
    *   Identify the use of proxies or VPNs, which often introduce specific latency patterns.
    *   Detect inconsistencies, such as a resource loading with a `0` duration, which may indicate the bot is using a local cache or intercepting requests in a non-standard way.

## Sample Code
This snippet demonstrates active fingerprinting by measuring the execution speed of a CPU-intensive task. The resulting duration is a stable and high-entropy value specific to the user's hardware and software environment.

```javascript
async function getCpuBenchmark() {
  const measureName = 'cpu-benchmark-task';
  const startMark = 'start-cpu-bm';
  const endMark = 'end-cpu-bm';

  // Ensure a clean slate for measurement
  performance.clearMarks();
  performance.clearMeasures();

  return new Promise(resolve => {
    // Use requestAnimationFrame to ensure the main thread is ready
    requestAnimationFrame(() => {
      performance.mark(startMark);

      // A CPU-intensive loop
      let result = 0;
      for (let i = 0; i < 2000000; i++) {
        result += Math.tan(Math.sqrt(i));
      }

      performance.mark(endMark);

      // Create a PerformanceEntry from the marks
      performance.measure(measureName, startMark, endMark);
      
      const entry = performance.getEntriesByName(measureName)[0];
      resolve(entry ? entry.duration : -1);
    });
  });
}

// Anti-bot systems would collect this value and send it to the server.
getCpuBenchmark().then(duration => {
  console.log(`CPU Benchmark Duration: ${duration}ms`);
  // A value of 0, -1, or an extremely low number could indicate a bot or spoofing.
  // A consistent value across many "users" suggests a bot farm.
});
```