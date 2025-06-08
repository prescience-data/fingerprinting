---
title: window.CanvasRenderingContext2D
layout: default
---
# window.CanvasRenderingContext2D
## Purpose
Provides the 2D drawing context for an HTML `<canvas>` element. It is the primary interface for drawing shapes, text, and images onto a bitmap surface within the browser.

## Fingerprinting Usage
Canvas fingerprinting is a highly effective and widely used technique. It works by instructing the browser to render a specific set of graphics and text, then extracting the resulting pixel data. The final image is not identical across different machines due to variations in the graphics rendering pipeline.

This creates a high-entropy fingerprint derived from the user's specific hardware and software configuration.

**Key Sources of Entropy:**

1.  **GPU & Graphics Driver:** The primary source of variation. Different GPUs (NVIDIA, AMD, Intel, Apple Silicon) and, more importantly, their specific driver versions, implement rendering operations like anti-aliasing and sub-pixel rendering differently. This results in subtle but consistent pixel-level differences in the final image.
2.  **Operating System:** The OS-level graphics libraries (e.g., DirectX on Windows, Metal/Core Graphics on macOS, OpenGL/Vulkan on Linux) influence how rendering commands are processed before being sent to the driver, adding another layer of variation.
3.  **Font Rendering:** The `fillText()` method is a critical component. The specific set of fonts installed on the OS, combined with the OS's font rasterization and smoothing engine (e.g., ClearType on Windows), produces unique text renderings. Emojis are particularly effective as their rendering is highly platform-dependent.
4.  **Browser Engine:** Minor differences in the implementation or version of the browser's graphics engine (e.g., Google's Skia, used in Chrome) can contribute to the fingerprint.

**Bot Detection Anomalies:**

*   **Inconsistency:** A bot may spoof its User-Agent to claim it's "Chrome on Windows" but produce a canvas fingerprint that is known to come from a headless Chromium instance running on a Linux server (e.g., in an AWS data center). This mismatch is a strong signal of deception.
*   **Known Headless Values:** Headless browsers like Puppeteer or Playwright, when run on common cloud infrastructure without a dedicated GPU, often produce a small set of well-known, default canvas hashes. Anti-bot systems maintain blocklists of these fingerprints.
*   **Failed Rendering:** Some minimal bot environments may lack a proper canvas implementation, causing errors or producing a blank image, which is an immediate red flag.
*   **Randomization Noise:** Advanced bots may attempt to defeat canvas fingerprinting by adding random noise to the pixel data. This can be detected because the noise pattern is statistically different from the subtle, deterministic variations seen in real user hardware.

## Sample Code
This snippet demonstrates the core logic. It draws text and shapes to an off-screen canvas and extracts the image data, which would then be hashed to create a fingerprint.

```javascript
/**
 * Generates a canvas fingerprint by rendering specific elements and extracting the data.
 * @returns {string} The Base64 encoded data URL of the rendered canvas.
 */
function getCanvasFingerprint() {
  try {
    // Create an off-screen canvas to avoid visual disruption
    const canvas = document.createElement('canvas');
    canvas.width = 200;
    canvas.height = 50;
    const ctx = canvas.getContext('2d');

    // A string with mixed characters and an emoji to maximize rendering differences
    const text = "FingerprintJS 🦄 #1.234";

    // Draw various elements to trigger different parts of the rendering engine
    ctx.textBaseline = "top";
    ctx.font = "14px 'Arial'";
    ctx.fillStyle = "#f60";
    ctx.fillRect(125, 1, 62, 20);
    ctx.fillStyle = "#069";
    ctx.fillText(text, 2, 15);
    ctx.fillStyle = "rgba(102, 204, 0, 0.7)";
    ctx.fillText(text, 4, 17);

    // Extract the pixel data as a Base64 encoded string.
    // This string is highly unique to the user's hardware/software stack.
    const dataUrl = canvas.toDataURL();

    // In a real-world scenario, this data URL would be hashed (e.g., with SHA-256)
    // to create a compact and stable fingerprint.
    return dataUrl;
  } catch (e) {
    // If canvas is blocked or fails, return an error indicator.
    return 'canvas-error';
  }
}

// const fingerprintData = getCanvasFingerprint();
// console.log(fingerprintData.substring(0, 80) + '...'); 
// Output: "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAMgAAAAyCAYAAACgq+lFAAA..."
```