---
title: window.Keyboard
layout: default
---
# window.Keyboard
## Purpose
The `Keyboard` API, accessed via `navigator.keyboard`, allows a web application to retrieve the user's physical keyboard layout. Its primary method, `getLayoutMap()`, returns a promise that resolves with a map of physical key codes to the characters they generate.

## Fingerprinting Usage
The `Keyboard` API is a high-entropy source for fingerprinting and a powerful tool for bot detection due to the strong correlation between keyboard layout, language, and geographic region.

1.  **High-Entropy Locale Signal:** The keyboard layout is a strong indicator of the user's language and region. While many users have a standard US QWERTY layout, the presence of layouts like QWERTZ (Germanic/Central European), AZERTY (French/Belgian), or Cyrillic (Russian) provides significant identifying information. This data can be combined with other locale-based signals (`navigator.language`, timezone) to create a more robust fingerprint.

2.  **Inconsistency Detection:** This is its most critical use in anti-bot systems. Bots and fraudulent users often spoof location-related properties like `navigator.language`, the `Accept-Language` header, or their timezone to appear as if they are from a specific country. However, they frequently fail to spoof the keyboard layout, which reflects the underlying OS configuration of the server running the bot. A mismatch—for example, a browser claiming `navigator.language: 'fr-FR'` but exposing a standard US QWERTY layout instead of AZERTY—is a major red flag for bot detection systems.

3.  **Headless Browser Artifact:** Headless browsers like Puppeteer or Playwright, when run in default configurations on cloud servers (typically Linux), will almost always report a standard `en-US` QWERTY keyboard layout. A large volume of traffic claiming diverse user profiles but consistently presenting this single keyboard layout is indicative of automation.

4.  **API Availability:** The `Keyboard` API is not universally supported. Its presence is a strong signal that the user is on a Chromium-based browser (Chrome, Edge, Opera), while its absence points towards Firefox or Safari. This binary check contributes to the overall browser fingerprint.

## Sample Code
This snippet demonstrates how an anti-bot script might check for inconsistencies between the claimed browser language and the detected keyboard layout. It specifically checks if a browser claiming to be German (`de`) has the expected QWERTZ layout.

```javascript
async function checkKeyboardLayoutConsistency() {
  // 1. Check if the API is even supported (fingerprints as Chromium-based)
  if (!navigator.keyboard || !navigator.keyboard.getLayoutMap) {
    console.log('Keyboard API not supported. (Likely Firefox/Safari)');
    return;
  }

  try {
    const layoutMap = await navigator.keyboard.getLayoutMap();
    const userLanguage = navigator.language.toLowerCase();

    // 2. Check for a common inconsistency: a "German" user without a German keyboard
    // On a US (QWERTY) keyboard, the key physically located where 'Z' is on a German
    // keyboard is 'Y'. The physical key code is `KeyY`.
    // A real German (QWERTZ) keyboard maps `KeyY` to the character 'z'.
    const keyYMapping = layoutMap.get('KeyY');

    if (userLanguage.startsWith('de') && keyYMapping !== 'z') {
      console.warn(`[BOT DETECTION] High Confidence: User claims German language ('${navigator.language}') but has a non-QWERTZ keyboard layout. KeyY maps to '${keyYMapping}'.`);
      // This is a strong signal of a bot or spoofed environment.
      return true; // Inconsistency detected
    }

    console.log('Keyboard layout appears consistent with browser language.');
    return false; // No inconsistency found

  } catch (error) {
    console.error('Could not get keyboard layout map:', error);
    return null; // Error during detection
  }
}

// Example usage:
checkKeyboardLayoutConsistency();
```