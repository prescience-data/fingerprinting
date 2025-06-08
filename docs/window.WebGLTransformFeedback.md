---
title: window.WebGLTransformFeedback
layout: default
---
# window.WebGLTransformFeedback
## Purpose
The `WebGLTransformFeedback` interface is part of the WebGL 2 API. It provides a mechanism to capture the output of a vertex shader into buffer objects, effectively "feeding back" transformed data before it goes through the rasterization pipeline.

## Fingerprinting Usage
The primary value of `WebGLTransformFeedback` in fingerprinting is not its mere existence, but its use as a tool to execute specific vertex shader programs and capture their output for analysis. The resulting data is a rich source of entropy.

*   **Hardware & Driver Signature:** The core fingerprinting technique involves running a standardized set of vertex transformations. The precise floating-point values of the output vertices are read back from the buffer. These values are highly sensitive to the underlying hardware and software stack, including:
    *   GPU vendor and model (NVIDIA, AMD, Intel, Apple Silicon, etc.)
    *   GPU driver version
    *   Operating System's graphics API (e.g., DirectX via ANGLE on Windows, Metal on macOS)
    *   Browser's specific graphics backend implementation

    The captured floating-point values are then hashed to produce a stable, high-entropy identifier often called a "WebGL Fingerprint."

*   **Bot & Environment Detection:** Anti-bot systems leverage this to uncover inconsistencies and identify non-standard environments.
    *   **Software vs. Hardware Rendering:** Headless browsers (like Puppeteer/Playwright) often default to using a software renderer (e.g., SwiftShader) instead of a physical GPU. A software renderer produces results that are distinctly different and more uniform across machines than a hardware-accelerated GPU. This is a very strong signal of a bot.
    *   **API Inconsistency:** A bot might spoof a user-agent string of a modern browser that supports WebGL 2, but its underlying engine may not have `WebGLTransformFeedback` implemented or enabled. The absence of the API when it should be present is a clear red flag.
    *   **Error Generation:** The complex setup required for a transform feedback operation can fail in specific ways on misconfigured browsers or bot environments. Capturing the exact error messages thrown can be used as an additional fingerprinting vector.

## Sample Code
This snippet demonstrates the logical flow of using `WebGLTransformFeedback` to generate a fingerprint. It omits the full boilerplate for brevity.

```javascript
async function getTransformFeedbackFingerprint() {
  try {
    // 1. Check for WebGL2 and Transform Feedback support
    if (!window.WebGL2RenderingContext || !window.WebGLTransformFeedback) {
      return "WebGL2_Unsupported";
    }

    const canvas = document.createElement('canvas');
    const gl = canvas.getContext('webgl2');
    if (!gl) {
      return "WebGL2_Context_Failed";
    }

    // 2. Define a vertex shader with specific calculations
    const vsSource = `#version 300 es
      in vec2 a_position;
      out vec4 v_color;
      void main() {
        // These calculations produce hardware/driver-specific results
        gl_Position = vec4(a_position * 0.987, 0.0, 1.0);
        v_color = vec4(fract(a_position), 0.0, 1.0);
      }
    `;
    
    // NOTE: Fragment shader is minimal as it's not used for transform feedback
    const fsSource = `#version 300 es
      precision highp float;
      out vec4 outColor;
      void main() {
        outColor = vec4(0,0,0,1);
      }
    `;

    // 3. Setup program, buffers, and transform feedback object
    // ... (Full setup code for shaders, program linking is omitted)
    const program = /* ... create and link program ... */;
    gl.useProgram(program);

    const tf = gl.createTransformFeedback();
    gl.bindTransformFeedback(gl.TRANSFORM_FEEDBACK, tf);

    const outputBuffer = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, outputBuffer);
    gl.bufferData(gl.ARRAY_BUFFER, 16 * 4, gl.STATIC_DRAW); // Allocate space
    gl.bindBuffer(gl.ARRAY_BUFFER, null);

    // Bind the buffer to the transform feedback stream
    gl.bindBufferBase(gl.TRANSFORM_FEEDBACK_BUFFER, 0, outputBuffer);

    // 4. Execute the draw call with transform feedback enabled
    gl.beginTransformFeedback(gl.POINTS);
    gl.drawArrays(gl.POINTS, 0, 1); // Run shader on a single vertex
    gl.endTransformFeedback();

    // 5. Read the data back from the buffer
    const outputData = new Float32Array(16);
    gl.getBufferSubData(gl.TRANSFORM_FEEDBACK_BUFFER, 0, outputData);

    // 6. Hash the result to create a stable fingerprint
    const fingerprint = outputData.toString(); // In reality, a robust hash function is used
    return fingerprint;

  } catch (e) {
    return `Error: ${e.message}`;
  }
}

// Example usage:
getTransformFeedbackFingerprint().then(fp => {
  console.log("Transform Feedback Fingerprint:", fp);
});
```