---
title: window.WebGLRenderingContext
layout: default
---
# window.WebGLRenderingContext
## Purpose
The `WebGLRenderingContext` interface provides the JavaScript API for rendering interactive 2D and 3D graphics within a `<canvas>` element. It exposes low-level graphics capabilities by providing a state machine and drawing functions that map closely to the underlying OpenGL ES 2.0 API, allowing direct interaction with the user's Graphics Processing Unit (GPU).

## Fingerprinting Usage
WebGL is one of the most potent sources of entropy for browser fingerprinting due to its direct access to the host's graphics hardware and driver stack.

*   **GPU and Driver Identification:** The `WEBGL_debug_renderer_info` extension allows direct querying of the GPU vendor and the specific renderer model string. These strings (e.g., "NVIDIA Corporation", "NVIDIA GeForce RTX 4090/PCIe/SSE2") are extremely high-entropy identifiers. Headless browsers and virtual machines often expose generic software rasterizers like "Google SwiftShader" or "Mesa/Gallium", which is a strong signal for bot detection.

*   **Canvas Hashing:** This is a classic technique where a specific, complex 3D scene is rendered to an offscreen canvas. The resulting pixel data is then extracted using `readPixels()` and hashed. Subtle differences in GPU hardware, driver versions, floating-point precision, and anti-aliasing algorithms across different machines will produce unique pixel outputs, resulting in a highly stable and unique hash.

*   **Capabilities and Parameters:** The specific set of supported WebGL extensions (`getSupportedExtensions()`), along with the values of various implementation-defined parameters (`getParameter()`), creates a detailed fingerprint. This includes details like shader precision (`getShaderPrecisionFormat`), maximum texture sizes, and aliased line width ranges, all of which vary based on the underlying graphics hardware and driver.

*   **Performance Anomalies:** The performance of WebGL rendering can be a behavioral signal. Bots running in VMs or using software rendering (like SwiftShader) will exhibit significantly slower rendering times for complex scenes compared to a real user's hardware-accelerated GPU. Timing these operations can effectively distinguish automated environments from legitimate user devices.

## Sample Code
This snippet demonstrates extracting the renderer information and generating a simplified canvas hash, two core WebGL fingerprinting techniques.

```javascript
function getWebGLFingerprint() {
  const fingerprint = {
    renderer: 'n/a',
    vendor: 'n/a',
    canvasHash: 'n/a',
  };

  try {
    const canvas = document.createElement('canvas');
    const gl = canvas.getContext('webgl') || canvas.getContext('experimental-webgl');

    if (!gl) {
      return fingerprint; // WebGL not supported is a fingerprintable attribute
    }

    // 1. Get GPU Vendor and Renderer information
    const debugInfo = gl.getExtension('WEBGL_debug_renderer_info');
    if (debugInfo) {
      fingerprint.vendor = gl.getParameter(debugInfo.UNMASKED_VENDOR_WEBGL);
      fingerprint.renderer = gl.getParameter(debugInfo.UNMASKED_RENDERER_WEBGL);
    }

    // 2. Generate a canvas hash from a rendered scene
    // A real-world implementation would use complex shaders and geometry.
    // This is a simplified demonstration.
    gl.clearColor(0.2, 0.5, 0.8, 1.0);
    gl.clear(gl.COLOR_BUFFER_BIT);
    
    // Add a simple triangle to create some pixel variation
    const vertexShader = gl.createShader(gl.VERTEX_SHADER);
    gl.shaderSource(vertexShader, 'attribute vec2 pos; void main() { gl_Position = vec4(pos, 0, 1); }');
    gl.compileShader(vertexShader);

    const fragmentShader = gl.createShader(gl.FRAGMENT_SHADER);
    gl.shaderSource(fragmentShader, 'void main() { gl_FragColor = vec4(1, 0, 0.5, 1); }');
    gl.compileShader(fragmentShader);

    const program = gl.createProgram();
    gl.attachShader(program, vertexShader);
    gl.attachShader(program, fragmentShader);
    gl.linkProgram(program);
    gl.useProgram(program);

    const buffer = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, buffer);
    gl.bufferData(gl.ARRAY_BUFFER, new Float32Array([ -0.2, -0.2, 0.2, -0.2, 0, 0.2 ]), gl.STATIC_DRAW);
    
    const position = gl.getAttribLocation(program, 'pos');
    gl.vertexAttribPointer(position, 2, gl.FLOAT, false, 0, 0);
    gl.enableVertexAttribArray(position);
    gl.drawArrays(gl.TRIANGLES, 0, 3);

    // The dataURL is a string representation of the rendered image
    fingerprint.canvasHash = canvas.toDataURL();

  } catch (e) {
    // Errors during WebGL context creation or rendering are also signals
    console.error('WebGL fingerprinting error:', e);
  }

  return fingerprint;
}

// Example usage:
const webglFP = getWebGLFingerprint();
console.log(webglFP);
// {
//   renderer: "Apple M1 Pro",
//   vendor: "Apple",
//   canvasHash: "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA..."
// }
```