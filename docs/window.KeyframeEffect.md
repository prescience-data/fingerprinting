---
title: window.KeyframeEffect
layout: default
---
# window.KeyframeEffect
## Purpose
The `KeyframeEffect` constructor, part of the Web Animations API (WAAPI), creates a new object that describes an animation as a set of executable keyframes and their timing options. This object is then typically applied to an element via an `Animation` object to be rendered by the browser's compositing engine.

## Fingerprinting Usage
`KeyframeEffect` is a high-entropy source for fingerprinting, primarily by measuring minute inconsistencies in rendering output. Anti-bot systems use it to create a "graphics fingerprint" that is difficult to spoof, as it depends on the entire graphics stack (OS, GPU, driver, browser engine).

1.  **Rendering Inconsistencies:** This is the most powerful vector. An animation is defined with complex transformations (e.g., `transform`, `filter`). The animation is run for a single frame or paused at a specific, non-trivial timestamp. `getComputedStyle` is then used to read the resulting property value. The exact floating-point values in the computed `transform` matrix (e.g., `matrix3d(...)`) will vary subtly across different:
    *   **GPUs:** NVIDIA, AMD, Intel, and Apple Silicon chips produce slightly different results.
    *   **Graphics Drivers:** Different driver versions can alter floating-point calculations.
    *   **Operating Systems:** The underlying graphics libraries (e.g., DirectX, Metal, OpenGL) influence the final output.
    *   **Browser Engine (Blink/V8):** Minor changes in Chrome's Skia rendering engine between versions can be detected.
    *   Bots using software rendering (like Mesa in headless Linux) or running without GPU acceleration produce starkly different results from a typical user's hardware-accelerated browser.

2.  **Performance Benchmarking:** The time required to construct and apply a large number of complex `KeyframeEffect` objects can be measured.
    *   **Signal:** Bots running in resource-constrained virtualized environments or headless browsers without proper GPU hardware support will exhibit significantly different performance profiles compared to real user devices. This can be used as a behavioral signal to flag non-human traffic.

3.  **Error Handling:** The exact error messages and error object types thrown for malformed keyframes or invalid timing options can differ between browser versions and engines. While lower in entropy, this can help distinguish V8/Blink from other engines like Gecko or WebKit.

## Sample Code
This snippet demonstrates the core rendering inconsistency technique. It creates an off-screen element, applies a complex animation, pauses it, and reads the computed transform matrix, which serves as a potent fingerprint.

```javascript
async function getAnimationFingerprint() {
  return new Promise(resolve => {
    // Use an off-screen element to avoid visual disruption.
    // 'display: none' would prevent rendering, so we position it out of view.
    const element = document.createElement('div');
    element.style.position = 'absolute';
    element.style.left = '-9999px';
    document.body.appendChild(element);

    // Define a complex animation with floating-point values
    const keyframes = [
      { transform: 'rotate(0.123rad) scale(1.001)' },
      { transform: 'rotate(0.456rad) scale(0.999)' }
    ];

    const options = {
      duration: 1000, // Duration doesn't matter much here
      fill: 'forwards'
    };

    // Create and play the animation
    const animation = element.animate(keyframes, options);
    
    // Pause the animation at a specific, non-trivial point in time
    animation.pause();
    animation.currentTime = 333.33; // A fractional time to force interpolation

    // We need to wait for the next frame for the style to be computed and applied
    requestAnimationFrame(() => {
      const computedStyle = window.getComputedStyle(element);
      const transformValue = computedStyle.getPropertyValue('transform');
      
      // Clean up the DOM
      document.body.removeChild(element);
      
      // The resulting string, e.g., "matrix(0.90..., 0.42..., -0.42..., ...)",
      // is a high-entropy fingerprint.
      resolve(transformValue);
    });
  });
}

// Usage:
getAnimationFingerprint().then(fingerprint => {
  console.log('Animation Fingerprint:', fingerprint);
  // Example output on one machine: "matrix(0.90998, 0.41466, -0.41466, 0.90998, 0, 0)"
  // This value will differ slightly on other machines.
});
```