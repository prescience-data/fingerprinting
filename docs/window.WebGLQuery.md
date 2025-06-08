---
title: window.WebGLQuery
layout: default
---
# window.WebGLQuery
## Purpose
The `WebGLQuery` object is part of the WebGL 2.0 API and allows for asynchronous queries of GPU-specific information. This includes performance timings, occlusion testing results (e.g., how many pixels were drawn), and other GPU pipeline statistics, without stalling the CPU.

## Fingerprinting Usage
`WebGLQuery` is a high-value API for both fingerprinting and bot detection due to the detailed, hardware-specific information it exposes.

1.  **Performance Timing (High Entropy):** The most potent use is measuring GPU performance. By using the `EXT_disjoint_timer_query_webgl2` extension, a script can precisely measure the time (in nanoseconds) it takes the GPU to render a standardized, complex scene. This timing is a highly specific signature of the user's hardware stack:
    *   **GPU Model:** Different GPUs complete the same task at different speeds.
    *   **Driver Version:** Driver updates can change performance characteristics.
    *   **Operating System:** The OS's graphics subsystem affects performance.
    *   **System Load & Power State:** The current load on the GPU and the device's power mode (e.g., laptop on battery) influence the result.
    The resulting nanosecond value is a high-entropy fingerprinting vector.

2.  **Hardware & Driver Anomaly Detection:** The results of `WebGLQuery` can be used to validate the browser environment and detect inconsistencies common in bots.
    *   **Software Rendering:** Headless browsers (like Puppeteer/Playwright) often default to or are configured with a software renderer (e.g., SwiftShader). A `TIME_ELAPSED` query will return drastically slower times compared to a real GPU, immediately flagging the environment as non-standard.
    *   **VM/Emulator Detection:** Virtualized GPUs in VMs have very distinct and often poor performance profiles that are easily identifiable with timing queries.
    *   **User-Agent Spoofing:** A bot may spoof its User-Agent to claim it's on a high-end machine (e.g., with an NVIDIA RTX 4090). If a `WebGLQuery` performance test returns a slow result inconsistent with that hardware, the bot's deception is revealed.

3.  **Behavioral Signals & Driver Quirks:**
    *   **Timing Consistency:** A real user's machine has fluctuating background GPU load from the OS and other applications. A bot running in a sterile, dedicated environment may produce timing results that are *too* consistent. Running a query multiple times and measuring the variance can be a behavioral signal.
    *   **Occlusion Query Results:** Queries like `ANY_SAMPLES_PASSED` count the number of fragments or samples that pass the depth test. Due to subtle differences in rasterization algorithms across GPU vendors and drivers, rendering the exact same 3D scene can produce slightly different pixel counts. This variation can be used as a fingerprinting signal.

## Sample Code
This snippet demonstrates how to measure GPU execution time, a common fingerprinting technique. It uses the `EXT_disjoint_timer_query_webgl2` extension to get a nanosecond-precision measurement for a simple draw call.

```javascript
async function getGpuTimeFingerprint() {
  const canvas = document.createElement('canvas');
  const gl = canvas.getContext('webgl2');
  if (!gl) {
    return 'WebGL2 not supported';
  }

  // This extension is crucial for performance timing
  const ext = gl.getExtension('EXT_disjoint_timer_query_webgl2');
  if (!ext) {
    return 'Timer query extension not supported';
  }

  const query = gl.createQuery();
  if (!query) {
    return 'Failed to create query object';
  }

  // Begin the asynchronous query
  gl.beginQuery(ext.TIME_ELAPSED_EXT, query);

  // --- Perform some standardized GPU work to measure ---
  // (A real fingerprinting script would use a much more complex scene)
  const vs = `void main() { gl_Position = vec4(0,0,0,1); gl_PointSize = 1.0; }`;
  const fs = `void main() { gl_FragColor = vec4(1,0,0,1); }`;
  const program = gl.createProgram();
  const vsShader = gl.createShader(gl.VERTEX_SHADER);
  gl.shaderSource(vsShader, vs);
  gl.compileShader(vsShader);
  const fsShader = gl.createShader(gl.FRAGMENT_SHADER);
  gl.shaderSource(fsShader, fs);
  gl.compileShader(fsShader);
  gl.attachShader(program, vsShader);
  gl.attachShader(program, fsShader);
  gl.linkProgram(program);
  gl.useProgram(program);
  gl.drawArrays(gl.POINTS, 0, 1);
  // --- End of GPU work ---

  gl.endQuery(ext.TIME_ELAPSED_EXT);

  // Poll until the query result is available
  return new Promise((resolve) => {
    function check() {
      const available = gl.getQueryParameter(query, gl.QUERY_RESULT_AVAILABLE);
      const disjoint = gl.getParameter(ext.GPU_DISJOINT_EXT);

      if (available && !disjoint) {
        const timeElapsed = gl.getQueryParameter(query, gl.QUERY_RESULT);
        // The result is in nanoseconds, a high-entropy value
        resolve(`GPU Time: ${timeElapsed} ns`);
      } else if (disjoint) {
        // Disjoint event means the timing is unreliable.
        resolve('GPU timing disjoint');
      } else {
        // Wait and check again
        setTimeout(check, 50);
      }
    }
    check();
  });
}

// Usage:
getGpuTimeFingerprint().then(fingerprint => {
  console.log(fingerprint); // e.g., "GPU Time: 18345 ns"
});
```