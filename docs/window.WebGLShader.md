---
title: window.WebGLShader
layout: default
---
# window.WebGLShader
## Purpose
Represents a single WebGL shader program (either a vertex or fragment shader). It is used to provide GLSL (OpenGL Shading Language) source code to the WebGL rendering context, which then compiles it for execution on the GPU.

## Fingerprinting Usage
`WebGLShader` is not directly fingerprinted itself, but it is the essential tool for two of the most powerful GPU-based fingerprinting techniques. The uniqueness stems from the fact that shader compilation and execution are handled by the underlying graphics driver and hardware, which vary significantly between systems.

1.  **Shader Precision & Canvas Fingerprinting:** This is a cornerstone of modern fingerprinting. A script creates a vertex and fragment shader to perform complex mathematical operations (e.g., trigonometric, exponential, or logarithmic functions) whose results are sensitive to floating-point precision differences. The shaders render the output as colors to a hidden WebGL canvas. The script then reads the pixel data (`gl.readPixels()`) and hashes it. The resulting hash is a highly stable and high-entropy fingerprint, as minute differences in the GPU hardware, driver version, or OS graphics stack (like ANGLE on Windows) produce different pixel values.

2.  **Shader Compiler Fingerprinting:** A script can intentionally try to compile a malformed or non-standard shader. The resulting error message, retrieved via `gl.getShaderInfoLog()`, is extremely specific to the GPU vendor's compiler (NVIDIA, AMD, Intel, Apple, etc.) and driver version. The format of the error string, the line number reported, and the specific error text create a highly unique identifier. Even successful compilations can sometimes produce warnings in the info log, which also contribute to the fingerprint.

3.  **Feature & Limit Detection:** Scripts can attempt to compile shaders that use specific GLSL features, extensions, or push hardware limits (e.g., number of `varying` vectors). The success or failure of these compilations reveals capabilities of the underlying hardware, contributing bits of information to a larger fingerprint. Bots or virtualized environments often use software renderers (like SwiftShader in headless Chrome) which have different capabilities and performance characteristics than native hardware, making them easy to detect with this method.

## Sample Code
This snippet demonstrates the "Shader Compiler Fingerprinting" technique by attempting to compile a deliberately broken shader and capturing the unique error message.

```javascript
function getShaderCompilerFingerprint() {
  try {
    const canvas = document.createElement('canvas');
    // Use `failIfMajorPerformanceCaveat` to potentially expose software renderers
    const gl = canvas.getContext('webgl', { failIfMajorPerformanceCaveat: true }) || 
               canvas.getContext('experimental-webgl', { failIfMajorPerformanceCaveat: true });

    if (!gl) {
      return 'WebGL not supported';
    }

    // Intentionally malformed shader source to trigger a compiler error
    const vertexShaderSource = `
      attribute vec2 a_position;
      void main() {
        gl_Position = vec4(a_position, 0.0, 1.0);
        // Syntax error on purpose to get a specific compiler message
        malformed_code_to_trigger_error;
      }
    `;

    const shader = gl.createShader(gl.VERTEX_SHADER);
    gl.shaderSource(shader, vertexShaderSource);
    gl.compileShader(shader);

    // The info log from a failed compilation is the fingerprint
    if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
      const errorLog = gl.getShaderInfoLog(shader);
      // The error log is highly specific to the GPU driver and OS.
      // e.g., "ERROR: 0:6: 'malformed_code_to_trigger_error' : undeclared identifier"
      // The exact format, line number, and message text vary significantly.
      return errorLog || 'Empty error log';
    }

    return 'No compilation error detected'; // Anomaly, as an error is expected
  } catch (e) {
    // Catching errors during context creation can also be a signal
    return `Error: ${e.message}`;
  }
}

// The returned string is a potent fingerprinting vector.
const fingerprint = getShaderCompilerFingerprint();
// console.log(fingerprint);
```