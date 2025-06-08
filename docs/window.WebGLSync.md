---
title: window.WebGLSync
layout: default
---
# WebGLSync
## Purpose
`WebGLSync` is an interface within the WebGL 2.0 API. It represents a synchronization primitive that allows JavaScript to coordinate with the asynchronous operations of the GPU, primarily by signaling when a set of GPU commands has finished executing.

## Fingerprinting Usage
`WebGLSync` is a powerful tool for bot detection and hardware fingerprinting, primarily through high-precision timing attacks. The core idea is to measure the exact time the GPU takes to complete a standardized task. This duration is a high-entropy signal directly tied to the user's underlying hardware and software stack.

*   **Hardware & Driver Identification:** The time required to render a complex scene and trigger a sync fence is highly dependent on the GPU model (e.g., NVIDIA RTX 4080 vs. Intel Iris Xe vs. Apple M2), the specific driver version, and the operating system's graphics subsystem. These timing variations create a detailed hardware fingerprint.

*   **Headless Browser & VM Detection:** This is its most critical anti-bot application.
    *   **Headless Browsers:** Automated browsers like Puppeteer or Playwright often run in a headless environment that uses a software-based renderer (e.g., SwiftShader in Chrome, Mesa/LLVMpipe on Linux) instead of a physical GPU. A software renderer is orders of magnitude slower than a real hardware GPU. A fingerprinting script can execute a WebGL task and measure the completion time using `WebGLSync`. If the time is exceptionally long (e.g., >100ms for a simple task that takes <5ms on hardware), it's a very strong indicator of a bot running on a server without a GPU.
    *   **Virtual Machines:** VMs use virtualized GPU drivers (e.g., VMware SVGA 3D, VBoxVGA) which have distinct and often slower performance characteristics than their bare-metal counterparts. Timing attacks using `WebGLSync` can reliably differentiate a user on a physical machine from a bot in a VM.

*   **Behavioral Signals:** While a single measurement is a fingerprint, repeated measurements can detect anomalies. For example, if the timing results are *too* consistent, it might suggest a spoofed environment where a bot is returning a fixed value instead of performing the actual computation, which would naturally have minor fluctuations due to system load.

## Sample Code
This snippet demonstrates the logic for a timing attack. It creates a WebGL task, sets a "fence" to mark its completion, and then polls until the fence is passed, measuring the elapsed time.

```javascript
async function getGpuRenderTime() {
  return new Promise((resolve, reject) => {
    try {
      const canvas = document.createElement('canvas');
      const gl = canvas.getContext('webgl2');

      if (!gl) {
        // WebGL2 support is a fingerprinting vector in itself.
        return reject('WebGL2 not supported');
      }

      // 1. Issue a simple but non-trivial command to the GPU.
      gl.clearColor(0.5, 0.5, 0.5, 1.0);
      gl.clear(gl.COLOR_BUFFER_BIT);

      // 2. Create a sync fence in the GPU command queue after the clear command.
      // This object will be signaled when the GPU finishes the work.
      const sync = gl.fenceSync(gl.SYNC_GPU_COMMANDS_COMPLETE, 0);

      // 3. Ensure commands are sent to the GPU to start execution.
      gl.flush();

      const startTime = performance.now();

      // 4. Poll to check the status of the sync object.
      const checkStatus = () => {
        const status = gl.clientWaitSync(sync, 0, 0); // The last '0' makes it non-blocking.

        if (status === gl.ALREADY_SIGNALED || status === gl.CONDITION_SATISFIED) {
          const duration = performance.now() - startTime;
          gl.deleteSync(sync); // Clean up
          // This duration is the fingerprint.
          // e.g., 0.8ms on a real GPU, 120ms in a headless browser.
          resolve(duration);
        } else {
          // If not done, check again on the next frame.
          setTimeout(checkStatus, 0);
        }
      };

      checkStatus();

    } catch (e) {
      reject(e);
    }
  });
}

// Usage:
getGpuRenderTime()
  .then(time => {
    console.log(`GPU task completed in: ${time.toFixed(2)} ms`);
    if (time > 50) {
      console.log('High probability of being a bot (software renderer).');
    }
  })
  .catch(error => console.error(error));
```