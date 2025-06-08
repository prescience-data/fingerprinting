---
title: window.WebGLProgram
layout: default
---
# window.WebGLProgram
## Purpose
A `WebGLProgram` object represents the fully compiled and linked executable code of a vertex and fragment shader pair. This program runs on the GPU to process vertices and color pixels within a WebGL context.

## Fingerprinting Usage
The `WebGLProgram` is a potent source of fingerprinting entropy because its creation and linking process is highly dependent on the underlying hardware and software stack. Anti-bot systems exploit the subtle differences in how different systems compile and link shader code.

*   **Shader Compiler/Linker Output:** The primary vector is the `getProgramInfoLog()` method. When a GLSL (OpenGL Shading Language) shader program is linked, the GPU driver's compiler generates informational messages, warnings, or errors. The exact text of these logs is extremely specific to the combination of:
    *   GPU Hardware (NVIDIA, AMD, Intel, Apple, etc.)
    *   GPU Driver Version
    *   Operating System
    *   Browser's WebGL implementation (e.g., ANGLE on Windows, which translates WebGL calls to DirectX)
    This results in a highly unique string that can serve as a stable identifier. For example, linking the exact same shader might produce an empty string on one driver, `"No errors."` on another, and a verbose success message on a third.

*   **Linking and Validation Status:** The `getProgramParameter()` method, when queried for `LINK_STATUS` and `VALIDATE_STATUS`, can reveal differences. A slightly malformed or complex shader might fail to link or validate on one GPU/driver combination but succeed on another, providing a strong binary signal.

*   **Bot & Headless Browser Detection:**
    *   **Software Rendering:** Headless browsers or virtualized environments may use a software rasterizer (like SwiftShader) instead of a real GPU. Software rasterizers produce completely different and easily identifiable program info logs and may fail to compile shaders that a real GPU would handle.
    *   **Inconsistency:** Bots often spoof high-end GPU strings (e.g., via `WEBGL_debug_renderer_info`). A detection script can compile a known shader and check if the resulting `programInfoLog` is consistent with the logs produced by the claimed hardware. A mismatch is a strong indicator of a bot.
    *   **API Hooking:** Sophisticated bots may hook `getProgramInfoLog` to return a fake, "clean" value. This can be detected by checking the integrity of the function itself (e.g., `WebGLProgram.prototype.getProgramInfoLog.toString()` should return `"[native code]"`) or by testing with a shader known to produce a specific error and verifying that the expected error message is returned.

## Sample Code
This snippet demonstrates how to create a minimal WebGL program, link it, and extract the info log for fingerprinting.

```javascript
function getShaderFingerprint() {
  try {
    const canvas = document.createElement('canvas');
    const gl = canvas.getContext('webgl') || canvas.getContext('experimental-webgl');
    if (!gl) {
      return 'WebGL not supported';
    }

    const vertexShaderSource = `
      attribute vec2 a_position;
      void main() {
        gl_Position = vec4(a_position, 0.0, 1.0);
      }
    `;

    const fragmentShaderSource = `
      precision mediump float;
      void main() {
        gl_FragColor = vec4(1.0, 0.0, 0.0, 1.0);
      }
    `;

    // Compile shaders
    const vertexShader = gl.createShader(gl.VERTEX_SHADER);
    gl.shaderSource(vertexShader, vertexShaderSource);
    gl.compileShader(vertexShader);

    const fragmentShader = gl.createShader(gl.FRAGMENT_SHADER);
    gl.shaderSource(fragmentShader, fragmentShaderSource);
    gl.compileShader(fragmentShader);

    // Create and link the program
    const program = gl.createProgram();
    gl.attachShader(program, vertexShader);
    gl.attachShader(program, fragmentShader);
    gl.linkProgram(program);

    // --- Fingerprinting Vector ---
    // The info log is highly specific to the GPU/driver/OS stack.
    const programInfoLog = gl.getProgramInfoLog(program);
    
    // Clean up resources
    gl.deleteProgram(program);
    gl.deleteShader(vertexShader);
    gl.deleteShader(fragmentShader);

    return programInfoLog || 'no_log'; // Return 'no_log' if the string is empty
  } catch (e) {
    return `Error: ${e.message}`;
  }
}

// Example usage:
const shaderFingerprint = getShaderFingerprint();
console.log('Shader Program Info Log:', shaderFingerprint);
```