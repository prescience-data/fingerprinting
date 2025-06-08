---
title: window.WebGL2RenderingContext
layout: default
---
# window.WebGL2RenderingContext
## Purpose
The `WebGL2RenderingContext` interface provides the API for WebGL 2.0, a low-level 3D graphics rendering context for the HTML `<canvas>` element. It exposes the system's Graphics Processing Unit (GPU) capabilities to the browser for hardware-accelerated 2D and 3D graphics.

## Fingerprinting Usage
WebGL is one of the most potent sources of entropy for browser fingerprinting. The specific way a device's GPU and driver combination renders graphics is highly unique. Anti-bot systems use this to create a stable, high-entropy identifier and to detect inconsistencies common in automated environments.

Key fingerprinting vectors include:

*   **Hardware and Driver Identification:** The `getParameter()` method can retrieve strings that explicitly identify the GPU vendor, the specific GPU model, and the driver stack. This is the most common and powerful use.
    *   `gl.RENDERER`: Returns a string like `"ANGLE (NVIDIA, NVIDIA GeForce RTX 4090 Direct3D11 vs_5_0 ps_5_0, D3D11)"`. This reveals the GPU, OS rendering layer (ANGLE on Windows), and driver details.
    *   `gl.VENDOR`: Returns the GPU vendor, e.g., `"Google Inc. (NVIDIA)"`.
*   **Capability Enumeration:** The fingerprint is enriched by querying the specific capabilities and limits of the GPU/driver implementation. This includes:
    *   The list of supported extensions (`gl.getSupportedExtensions()`).
    *   Shader precision formats (`gl.getShaderPrecisionFormat()`).
    *   Dozens of implementation-defined limits, such as `MAX_TEXTURE_SIZE`, `MAX_VIEWPORT_DIMS`, and `MAX_VERTEX_ATTRIBS`. The combination of these numeric values creates a detailed hardware signature.
*   **Render Fingerprinting (Canvas Hashing):** This is a more robust technique that is very difficult to spoof.
    1.  A specific, complex 3D scene is rendered to an offscreen canvas using WebGL.
    2.  The `readPixels()` method is used to extract the raw pixel data of the resulting image.
    3.  This pixel data is then hashed.
    Subtle differences in floating-point math, anti-aliasing, and driver-level optimizations across different GPUs and drivers produce unique pixel-level variations in the final image, resulting in a unique hash.
*   **Bot Detection Anomalies:**
    *   **Software Rendering:** Headless browsers (like Puppeteer/Playwright) and servers often default to a software renderer like Google's "SwiftShader" or the open-source "Mesa". The presence of these strings in the `RENDERER` field is a strong signal of a non-human user.
    *   **Inconsistencies:** A browser with a User-Agent for a high-end Windows gaming PC that reports a "VMware SVGA II Adapter" or "llvmpipe" renderer is a major red flag.
    *   **Missing WebGL:** A modern desktop browser that fails to initialize a WebGL context is highly suspicious.
    *   **Spoofing Failure:** A bot might spoof the `RENDERER` string but will fail the render fingerprinting test because its underlying software renderer cannot produce the same image hash as the claimed hardware.

## Sample Code
This snippet demonstrates how to extract basic GPU parameters, which form the core of a WebGL fingerprint.

```javascript
function getWebGLFingerprint() {
  const canvas = document.createElement('canvas');
  let gl;
  try {
    // Use webgl2 for more parameters, fallback to webgl if needed
    gl = canvas.getContext('webgl2') || canvas.getContext('webgl');
  } catch (e) {
    return { error: 'WebGL context creation failed.' };
  }

  if (!gl) {
    return { error: 'WebGL is not supported.' };
  }

  const debugInfo = gl.getExtension('WEBGL_debug_renderer_info');
  const fingerprint = {
    // The UNMASKED_RENDERER_WEBGL reveals the true GPU, bypassing some privacy layers.
    renderer: debugInfo ? gl.getParameter(debugInfo.UNMASKED_RENDERER_WEBGL) : 'N/A',
    vendor: debugInfo ? gl.getParameter(debugInfo.UNMASKED_VENDOR_WEBGL) : 'N/A',
    // A full implementation would also collect dozens of other parameters:
    // maxTextureSize: gl.getParameter(gl.MAX_TEXTURE_SIZE),
    // supportedExtensions: gl.getSupportedExtensions(),
    // ... etc.
  };

  // A real-world script would then hash this collected data.
  // For example: hash(JSON.stringify(fingerprint))
  return fingerprint;
}

const webglFP = getWebGLFingerprint();
console.log(webglFP);
// Example Output on a Windows machine with an NVIDIA card:
// {
//   renderer: "ANGLE (NVIDIA, NVIDIA GeForce RTX 4090 Direct3D11 vs_5_0 ps_5_0, D3D11)",
//   vendor: "Google Inc. (NVIDIA)"
// }
```