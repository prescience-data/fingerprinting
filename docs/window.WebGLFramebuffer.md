---
title: window.WebGLFramebuffer
layout: default
---
# window.WebGLFramebuffer
## Purpose
`WebGLFramebuffer` is a WebGL API object that represents an off-screen render target. It allows developers to render a scene to a texture or renderbuffer, which can then be read back or used in subsequent rendering passes, without displaying it directly on the visible canvas.

## Fingerprinting Usage
`WebGLFramebuffer` is a critical component of WebGL fingerprinting, a highly effective and high-entropy identification technique. It enables the "canvas" part of this method: rendering an image and reading its pixel data back for analysis.

The process works as follows:
1.  A specific 3D scene, defined by vertices and shaders, is rendered off-screen into a `WebGLFramebuffer`.
2.  The `gl.readPixels()` method is then used to read the raw pixel data from the framebuffer into a `TypedArray`.
3.  This array of pixel values is hashed (e.g., using SHA-256) to produce a stable fingerprint.

The resulting hash is unique across different hardware and software configurations due to subtle variations in how the graphics pipeline is implemented:
*   **GPU Hardware:** Different GPUs (NVIDIA, AMD, Intel, Apple) and even different models from the same vendor will produce slightly different rasterization and shading results.
*   **Graphics Driver:** The driver version directly impacts the implementation of OpenGL/DirectX/Metal calls, leading to minor rendering differences.
*   **Operating System:** The OS graphics stack influences rendering. On Windows, Chrome uses the ANGLE project to translate WebGL calls to DirectX, adding another layer of variation.
*   **Browser Implementation:** The browser's specific WebGL implementation can introduce unique artifacts.

For bot detection, this is a powerful signal:
*   **Headless Detection:** Headless browsers often use software renderers like SwiftShader (in Chrome) or Mesa/LLVMpipe (on Linux servers), which produce fingerprints vastly different from a typical user's GPU.
*   **Inconsistency:** A bot might spoof its User-Agent to claim it's Chrome on Windows with an NVIDIA GPU, but its WebGL fingerprint will reveal the underlying Linux server or software renderer, exposing the deception.
*   **Errors & Lack of Support:** Simpler bots or automation frameworks may not have WebGL support, causing the context creation or rendering to fail. This failure is a strong signal of a non-standard environment.

## Sample Code
This snippet demonstrates the core logic of creating a framebuffer, rendering a simple shape, reading the pixels, and generating a hash.

```javascript
async function getWebglFingerprint() {
  try {
    const canvas = document.createElement('canvas');
    const gl = canvas.getContext('webgl') || canvas.getContext('experimental-webgl');

    if (!gl) {
      return 'WebGL not supported';
    }

    // Create a framebuffer
    const framebuffer = gl.createFramebuffer();
    gl.bindFramebuffer(gl.FRAMEBUFFER, framebuffer);

    // Attach a texture to the framebuffer
    const texture = gl.createTexture();
    gl.bindTexture(gl.TEXTURE_2D, texture);
    gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, 16, 16, 0, gl.RGBA, gl.UNSIGNED_BYTE, null);
    gl.framebufferTexture2D(gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT0, gl.TEXTURE_2D, texture, 0);

    // --- Complex rendering logic would go here ---
    // This part is simplified for the example. Real-world fingerprinting
    // uses complex shaders and geometry to maximize entropy.
    gl.clearColor(0.25, 0.5, 0.75, 1.0);
    gl.clear(gl.COLOR_BUFFER_BIT);
    // gl.drawArrays(...) or gl.drawElements(...) would be here.

    // Read the pixels back from the framebuffer
    const pixels = new Uint8Array(16 * 16 * 4);
    gl.readPixels(0, 0, 16, 16, gl.RGBA, gl.UNSIGNED_BYTE, pixels);

    // Hash the pixel data to create a fingerprint
    const pixelString = Array.from(pixels).join(',');
    const hashBuffer = await crypto.subtle.digest('SHA-256', new TextEncoder().encode(pixelString));
    const hashArray = Array.from(new Uint8Array(hashBuffer));
    const hashHex = hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
    
    return hashHex;

  } catch (e) {
    return `Error: ${e.message}`;
  }
}

// Usage:
// getWebglFingerprint().then(fp => console.log(fp));
```