---
title: window.queryLocalFonts
layout: default
---
# window.queryLocalFonts
## Purpose
Part of the Local Font Access API, `window.queryLocalFonts()` allows a web application to request an enumeration of the fonts installed on the user's local system. It requires explicit user permission via a browser prompt before providing any data.

## Fingerprinting Usage
The list of installed fonts is a classic and extremely high-entropy fingerprinting vector. `queryLocalFonts` provides a direct, reliable method to access this information, superseding older, less reliable techniques like canvas or element measurement.

*   **High-Entropy Data Source:** The specific combination of fonts installed on a device is highly unique. It is determined by the operating system, its version, installed software (e.g., Microsoft Office, Adobe Creative Suite), and fonts installed manually by the user. The resulting list can uniquely identify a user across different websites.

*   **Bot vs. Human Differentiation:**
    *   **Bots:** Headless browsers running in minimalist environments (like Docker containers on Linux) typically have a very small, predictable set of default fonts (e.g., DejaVu, Liberation, Noto). A short, standard font list is a strong indicator of a non-human, automated environment.
    *   **Real Users:** Have a much larger and more diverse set of fonts corresponding to their OS (e.g., Calibri, Segoe UI on Windows; Helvetica, San Francisco on macOS) and applications.

*   **Consistency Checks:** The font list can be cross-referenced with other signals like the `User-Agent` string. For example, a browser reporting a Windows `User-Agent` but providing a font list characteristic of a default Linux server is a clear anomaly, indicating spoofing.

*   **Behavioral Signal (Permission Prompt):** Unlike passive fingerprinting methods, this API requires user interaction. This interaction itself is a signal:
    *   A bot may be unable to handle the permission prompt, causing the script to time out or fail.
    *   The speed and method of interacting with the prompt (e.g., an instantaneous, programmatic click on "Allow") can be detected as non-human behavior.
    *   A user denying the request is also a data point.

## Sample Code
This snippet demonstrates how a script would request the font list and process it. The resulting array of font names is the fingerprint.

```javascript
async function getFontFingerprint() {
  try {
    // This call triggers a user permission prompt.
    const availableFonts = await window.queryLocalFonts();

    const fontNames = [];
    for (const fontData of availableFonts) {
      // The PostScript name is a stable and unique identifier for a font.
      fontNames.push(fontData.postscriptName);
    }

    // Sort the list to ensure a consistent hash regardless of enumeration order.
    fontNames.sort();
    
    // The resulting array of font names is a high-entropy fingerprint.
    // It can be hashed and sent to a server for analysis.
    console.log(`Found ${fontNames.length} fonts.`);
    return fontNames;

  } catch (err) {
    // The promise rejects if the user denies permission or the feature is disabled.
    // This rejection is itself a signal (user denied, or bot failed to handle the prompt).
    console.error('Font access denied or failed:', err.name, err.message);
    return 'permission_denied';
  }
}

getFontFingerprint();
```