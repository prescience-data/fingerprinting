---
title: window.WebGLContextEvent
layout: default
---
# window.WebGLContextEvent
## Purpose
The `WebGLContextEvent` interface represents an event that occurs when the state of a WebGL rendering context changes. It is the underlying type for events indicating that the context has been lost (`webglcontextlost`) or restored (`webglcontextrestored`).

## Fingerprinting Usage
While the mere existence of the API is a basic check, its primary value lies in actively probing the browser and underlying hardware to reveal high-entropy identifiers and behavioral anomalies.

1.  **GPU Resource Limit Probing:** This is the most powerful use case. A script can repeatedly create WebGL contexts on different canvas elements until the browser can no longer allocate GPU resources, triggering a `webglcontextlost` event. The number of contexts successfully created before this event fires is a highly specific fingerprint. This value is determined by:
    *   Available Video RAM (VRAM).
    *   The GPU driver's resource management policies.
    *   The browser's internal limits on WebGL contexts.
    This active test can effectively differentiate between different hardware configurations. Headless browsers or virtualized environments often have very different, and sometimes fixed, limits compared to real user hardware.

2.  **Behavioral Anomaly Detection:** Real user browsers can lose the WebGL context for organic reasons, such as the OS reclaiming resources, a GPU driver crash/update, or the system entering sleep mode. Bots running in sterile, controlled environments rarely experience these events. A long-running session that never reports a context loss, especially under simulated memory pressure, can be flagged as suspicious.

3.  **Context Restoration Timing:** After a `webglcontextlost` event, a `webglcontextrestored` event may follow. The time delay between these two events can be measured. This timing is influenced by the driver and hardware. Bots or emulators may not implement this behavior correctly, failing to restore the context at all, or doing so with an unnaturally fast or consistent timing profile that deviates from real-world patterns.

4.  **Environment Emulation Detection:** Many botting frameworks and headless browsers (especially older or poorly configured ones) do not properly emulate GPU behavior. Forcing a context loss (as described in point 1) and observing that no `webglcontextlost` event is ever fired is a strong signal of a non-standard or emulated environment.

## Sample Code
This snippet demonstrates how to probe the maximum number of active WebGL contexts, a high-entropy fingerprinting vector.

```javascript
async function getWebGLContextLimit() {
  return new Promise(resolve => {
    const contexts = [];
    let count = 0;
    const maxContexts = 100; // Safety limit to prevent browser crash

    const canvas = document.createElement('canvas');
    canvas.addEventListener('webglcontextlost', (event) => {
      event.preventDefault();
      // The limit is the number of contexts created *before* the loss event.
      resolve(count); 
    }, { once: true });

    try {
      for (let i = 0; i < maxContexts; i++) {
        const ctx = canvas.getContext('webgl');
        if (!ctx) {
          // Browser returned null before firing the event
          resolve(count);
          return;
        }
        contexts.push(ctx); // Keep reference to prevent GC
        count++;
      }
    } catch (e) {
      // Some browsers might throw an error instead of firing an event
      resolve(count);
      return;
    }
    
    // If loop finishes without loss, the limit is at least maxContexts
    resolve(count);
  });
}

// Usage:
getWebGLContextLimit().then(limit => {
  console.log(`WebGL Context Limit: ${limit}`);
  // This 'limit' value is a fingerprint.
  // e.g., A real RTX 3080 might report 16, while a bot reports 8 or fails.
});
```