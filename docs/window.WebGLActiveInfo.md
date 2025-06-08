---
title: window.WebGLActiveInfo
layout: default
---
# WebGLActiveInfo
## Purpose
`WebGLActiveInfo` is not a function but an object type returned by the `WebGLRenderingContext.getActiveAttrib()` and `WebGLRenderingContext.getActiveUniform()` methods. It provides metadata about the active attributes (per-vertex data) and uniforms (global variables) within a compiled WebGL shader program.

## Fingerprinting Usage
The `WebGLActiveInfo` object is a high-entropy source for fingerprinting because the information it contains is determined by the GLSL compiler within the system's graphics driver, not just the browser. The same shader source code can be compiled differently across various combinations of GPU hardware, driver versions, and operating systems.

*   **Name Mangling:** The most significant source of entropy is the `name` property. Graphics drivers may rename, shorten, or otherwise alter the names of uniforms and attributes for optimization. For example, a uniform array declared as `uniform vec4 u_data[2];` might be reported with the name `u_data` by one driver and `u_data[0]` by another. This variability is a powerful fingerprinting signal.

*   **Attribute/Uniform Optimization:** A driver's compiler might optimize away an unused attribute or uniform entirely. The presence or absence of certain active variables after compilation serves as a behavioral fingerprint of that specific compiler and driver version.

*   **Consistency with Renderer String:** Advanced anti-bot systems cross-reference the fingerprint derived from `WebGLActiveInfo` with the reported `UNMASKED_RENDERER_WEBGL` string. A bot might spoof its renderer to appear as a common consumer GPU (e.g., "NVIDIA GeForce RTX 3080"). However, if the detailed attribute/uniform information does not match the known signature for that GPU and its typical drivers, it is a strong indication of spoofing.

*   **Headless/VM Detection:** Headless browsers (like Puppeteer/Playwright) and virtual machines often use software renderers (e.g., Google's SwiftShader, Mesa) or standardized server GPUs. These environments produce highly consistent and easily identifiable `WebGLActiveInfo` fingerprints that differ significantly from the diverse fingerprints of real users' consumer hardware.

## Sample Code
This snippet compiles a simple shader, then extracts the names of its active uniforms. The resulting list of names is a component of a WebGL fingerprint.

```javascript
function getUniformFingerprint() {
  const canvas = document.createElement('canvas');
  const gl = canvas.getContext('webgl');
  if (!gl) {
    return 'no-webgl';
  }

  // Simple shaders with attributes and uniforms
  const vsSource = 'attribute vec4 a_pos; void main() { gl_Position = a_pos; }';
  const fsSource = `
    precision mediump float;
    uniform vec4 u_color;
    uniform vec2 u_resolution;
    uniform float u_data[2]; // Array uniform is a good source of entropy
    void main() { gl_FragColor = u_color; }
  `;

  const vertexShader = gl.createShader(gl.VERTEX_SHADER);
  gl.shaderSource(vertexShader, vsSource);
  gl.compileShader(vertexShader);

  const fragmentShader = gl.createShader(gl.FRAGMENT_SHADER);
  gl.shaderSource(fragmentShader, fsSource);
  gl.compileShader(fragmentShader);

  const program = gl.createProgram();
  gl.attachShader(program, vertexShader);
  gl.attachShader(program, fragmentShader);
  gl.linkProgram(program);

  const uniformCount = gl.getProgramParameter(program, gl.ACTIVE_UNIFORMS);
  const uniformNames = [];

  for (let i = 0; i < uniformCount; i++) {
    // getActiveUniform returns a WebGLActiveInfo object
    const activeUniform = gl.getActiveUniform(program, i);
    if (activeUniform) {
      // The 'name' property is highly variable across drivers
      uniformNames.push(activeUniform.name);
    }
  }

  // The sorted list of names is the fingerprint component.
  // e.g., ["u_color", "u_data[0]", "u_resolution"] on one machine
  // vs.    ["u_color", "u_data", "u_resolution"] on another.
  return uniformNames.sort().join(',');
}

// Example usage:
const fingerprint = getUniformFingerprint();
console.log(fingerprint);
```