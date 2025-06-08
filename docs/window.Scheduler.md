---
title: window.scheduler
layout: default
---
# window.Scheduler
## Purpose
The `Scheduler` API provides a low-level mechanism to schedule tasks on the main thread with different priorities (`user-blocking`, `user-visible`, `background`). Its primary goal is to help developers manage long-running scripts without blocking the UI, improving page responsiveness.

## Fingerprinting Usage
The `window.Scheduler` API is a potent tool for both fingerprinting and bot detection due to its behavioral and implementation-specific nature.

1.  **API Presence:** The mere existence of `window.scheduler` is a strong signal. It is a modern, Chromium-specific API. Its absence in a browser claiming a Chrome User-Agent is a red flag, while its presence rules out Firefox, Safari, and older browser versions.

2.  **Task Scheduling Behavior:** This is the most valuable signal. Anti-bot systems can measure the time between when a task is posted with `scheduler.postTask()` and when it actually executes. This timing reveals subtle characteristics of the environment:
    *   **Priority Implementation:** A key test is to post tasks with all three priorities (`user-blocking`, `user-visible`, `background`) and measure their execution deltas. In a genuine Chrome browser, there should be a measurable and logical difference in scheduling latency; `user-blocking` tasks should execute significantly faster than `background` tasks under normal load. Headless browsers or emulators may fail to correctly implement this priority queue, resulting in near-identical timings for all priorities, which is a strong bot signal.
    *   **Timing Jitter and Consistency:** The exact delay for a task is non-deterministic and depends on the V8 task scheduler, OS scheduler, and current system load. This "jitter" creates a behavioral signature. A real user's machine will produce a noisy but statistically consistent timing profile. In contrast, a bot environment might be:
        *   **Too perfect:** Timings are overly consistent or have an unnaturally low standard deviation, suggesting an emulated or non-standard event loop.
        *   **Too chaotic:** Timings show no correlation with the specified priority, indicating a flawed or overwhelmed scheduler, common in virtualized server environments running many bot instances.
    *   **System Load Signature:** The timing profile indirectly measures the performance and load of the underlying machine. A server CPU running hundreds of bot instances will exhibit a different scheduling latency profile compared to a typical consumer desktop or mobile device. This contributes entropy to the overall device fingerprint.

## Sample Code
This snippet demonstrates how to measure the execution delay for different task priorities. The resulting `timings` array is a behavioral fingerprint that can be analyzed for anomalies.

```javascript
async function getSchedulerFingerprint() {
  // 1. Check for API existence (Chromium vs. others)
  if (!window.scheduler || typeof window.scheduler.postTask !== 'function') {
    return { error: 'Scheduler API not supported.' };
  }

  const priorities = ['user-blocking', 'user-visible', 'background'];
  const timings = [];

  // A function to measure the delay for a given priority
  const measureTaskDelay = (priority) => {
    return new Promise(resolve => {
      const startTime = performance.now();
      
      window.scheduler.postTask(() => {
        const endTime = performance.now();
        const delay = endTime - startTime;
        resolve({ priority, delay });
      }, { priority });
    });
  };

  // 2. Measure and collect timing data for each priority
  for (const priority of priorities) {
    const result = await measureTaskDelay(priority);
    timings.push(result);
  }

  // 3. The 'timings' array is the fingerprint.
  // A detection script would analyze this data.
  // For example, it would check if user-blocking delay < background delay.
  // It would also check if delays are unnaturally low or consistent.
  console.log('Scheduler Timings Fingerprint:', timings);
  return timings;
}

// Example usage:
getSchedulerFingerprint().then(fingerprint => {
  if (fingerprint.error) {
    console.log(fingerprint.error);
  } else {
    // In a real scenario, this data would be sent to a server for analysis.
    // Is fingerprint[0].delay (user-blocking) > fingerprint[2].delay (background)?
    // If so, this is a strong anomaly signal.
  }
});
```