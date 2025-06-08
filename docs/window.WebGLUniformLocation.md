---
title: window.WebGLUniformLocation
layout: default
---
# window.WebGLUniformLocation
## Purpose
A `WebGLUniformLocation` is an opaque object that serves as a reference to the memory location of a "uniform" variable inside a compiled WebGL shader program. Uniforms are global variables passed from JavaScript to the shaders, allowing for dynamic control over rendering without recompiling the shader code.

## Fingerprinting Usage
The primary fingerprinting value of `getUniformLocation` (the method that returns a `WebGLUniformLocation`) lies not in the object itself, but in its return value: either a valid location object or `null`. This behavior directly exposes the results of the GPU driver's internal shader compiler optimizations.

*   **Shader Compiler Optimization:** When a WebGL program is linked, the GPU driver's shader compiler aggressively optimizes the code. If a declared uniform variable is not actually used in the final calculation of `gl_Position` (in the vertex shader) or `gl_FragColor` (in the fragment shader), the compiler may remove it entirely to save resources.
*   **Entropy Source:** Different GPU vendors (NVIDIA, AMD, Intel, Apple), driver versions, and even operating systems have unique shader compilers with distinct optimization strategies. A fingerprinting script can create a complex shader with numerous uniforms, some used directly, some indirectly, and some in convoluted ways. The script then calls `getUniformLocation` for each uniform. The specific set of uniforms that are optimized away (returning `null`) versus those that remain (returning a valid object) creates a highly specific and stable signature of the user's hardware and software stack.
*   **Bot Detection Anomaly:** This provides a powerful method for detecting inconsistencies. A bot might spoof its User-Agent string to claim it is a macOS device with an Apple M2 GPU. However, if the pattern of optimized uniforms matches that of a common NVIDIA GPU on a Linux server, the deception is revealed. This check is difficult to fake without running a real browser on the claimed hardware, as it requires emulating the proprietary behavior of a specific GPU driver's compiler.

## Sample Code
This snippet demonstrates how to probe the shader compiler's optimization behavior. It defines three uniforms: one that is clearly used, one that is clearly unused, and one whose usage is non-obvious, making it a good candidate for divergent optimization behavior across different drivers.

```javascript
function getShaderOptimizationFingerprint() {
  const canvas = document.createElement('canvas');
  const gl = canvas.getContext('webgl');
  if (!gl) {
    return 'no_webgl';
  }

  // A fragment shader with uniforms designed to test the compiler.
  const fragmentShaderSource = `
    precision mediump float;
    uniform float u_used;          // Will be used directly.
    uniform float u_unused;        // Declared but never referenced.
    uniform vec4 u_subtly_unused;  // Used in a calculation whose result is discarded.

    void main() {
      vec4 temp = u_subtly_unused * 0.5; // A smart compiler sees 'temp' is unused.
      gl_FragColor = vec4(u_used, 0.0, 0.0, 1.0);
    }
  `;

  const vertexShaderSource = `
    attribute vec2 a_position;
    void main() {
      gl_Position = vec4(a_position, 0.0, 1.0);
    }
  `;

  // Standard WebGL program setup
  const program = gl.createProgram();
  const vShader = gl.createShader(gl.VERTEX_SHADER);
  const fShader = gl.createShader(gl.FRAGMENT_SHADER);
  gl.shaderSource(vShader, vertexShaderSource);
  gl.shaderSource(fShader, fragmentShaderSource);
  gl.compileShader(vShader);
  gl.compileShader(fShader);
  gl.attachShader(program, vShader);
  gl.attachShader(program, fShader);
  gl.linkProgram(program);

  // The fingerprinting logic: check which uniforms survived compilation.
  const used_loc = gl.getUniformLocation(program, 'u_used');
  const unused_loc = gl.getUniformLocation(program, 'u_unused');
  const subtly_unused_loc = gl.getUniformLocation(program, 'u_subtly_unused');

  // Create a fingerprint string from the results.
  // Expected on most modern compilers: "found_null_null"
  const fingerprint = [
    used_loc ? 'found' : 'null',
    unused_loc ? 'found' : 'null',
    subtly_unused_loc ? 'found' : 'null'
  ].join('_');

  return fingerprint;
}

// Example usage:
// const fp = getShaderOptimizationFingerprint();
// console.log(fp); // e.g., "found_null_null" on one machine,
                  // "found_null_found" on another (less optimized one).
```