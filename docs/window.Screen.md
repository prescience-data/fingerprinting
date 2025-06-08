---
title: window.screen
layout: default
---
# window.Screen
## Purpose
The `window.Screen` object provides properties detailing the user's display screen, including its total dimensions, available rendering space (excluding OS interface elements like taskbars), color depth, and orientation.

## Fingerprinting Usage
The `screen` object is a cornerstone of browser fingerprinting due to the high entropy provided by its properties, especially in combination. Anti-bot systems use it extensively to detect inconsistencies that expose automation or virtualization.

*   **Screen Resolution (`width`, `height`):** The combination of screen width and height is a primary fingerprinting vector. While common resolutions like 1920x1080 are prevalent, the full distribution of resolutions across the web is vast, providing significant entropy. Headless browsers often use default resolutions (e.g., 800x600 for headless Chrome, 1024x768 for Puppeteer's default) unless explicitly configured, making them easy to identify.

*   **Available Screen Space (`availWidth`, `availHeight`):** This is more subtle and powerful than total resolution. The difference between `height` and `availHeight` can reveal the size and position of the OS taskbar or menu bar (e.g., a ~40px difference at the bottom suggests a Windows taskbar; a ~25px difference at the top suggests a macOS menu bar). Bots or virtualized environments often fail to mimic this realistically, reporting `height` equal to `availHeight`.

*   **Color and Pixel Depth (`colorDepth`, `pixelDepth`):** On modern systems, these values have low entropy, typically being `24` or `32`. However, they serve as a strong consistency check. An unusual value (e.g., `16`) can indicate an older system, a remote desktop session, or a misconfigured virtual machine. If a bot spoofs these values, they might be inconsistent with other system properties.

*   **Consistency Checks (Anti-Bot Signal):** The most critical use in bot detection is checking for logical inconsistencies between `screen` properties and other browser APIs.
    *   **`screen` vs. `window`:** The `window.innerWidth`/`innerHeight` must be less than or equal to `screen.availWidth`/`availHeight`. A browser window cannot be larger than the available screen space. Automation frameworks can easily get this wrong when spoofing values.
    *   **`screen` vs. CSS Media Queries:** A sophisticated check involves comparing the JavaScript-reported `screen.width` with the result of a CSS media query (e.g., `@media (max-width: ...)`). A bot might hook the `screen.width` property in JavaScript but fail to alter the underlying value used by the browser's rendering engine for CSS evaluation, creating a detectable mismatch.

## Sample Code
```javascript
function getScreenFingerprint() {
    const s = window.screen;
    const fingerprint = {
        // High entropy resolution data
        resolution: `${s.width}x${s.height}`,
        availableResolution: `${s.availWidth}x${s.availHeight}`,
        
        // Color information, useful for detecting anomalies
        colorDepth: s.colorDepth,
        pixelDepth: s.pixelDepth,
        
        // Consistency check: A key anti-bot signal.
        // A real browser window cannot be larger than the available screen.
        isConsistent: window.innerWidth <= s.availWidth && window.innerHeight <= s.availHeight,

        // Taskbar/UI geometry inference
        // A non-zero value suggests a taskbar or menu bar.
        uiHeight: s.height - s.availHeight,
    };

    // The collected object can be stringified and hashed
    // to contribute to a user's overall fingerprint.
    return JSON.stringify(fingerprint);
}

// Example usage:
// const screenFP = getScreenFingerprint();
// console.log(screenFP);
// Result might be:
// {"resolution":"1920x1080","availableResolution":"1920x1040","colorDepth":24,"pixelDepth":24,"isConsistent":true,"uiHeight":40}
```