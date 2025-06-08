---
title: window.TextDecoder
layout: default
---
# window.TextDecoder
## Purpose
The `TextDecoder` interface, part of the Encoding API, decodes a stream of bytes (e.g., from a `Uint8Array`) into a string, using a specified character encoding like 'utf-8'. It is the programmatic counterpart to `TextEncoder`.

## Fingerprinting Usage
`TextDecoder` is a potent source of entropy for fingerprinting due to implementation variations across browsers, versions, and underlying operating systems.

1.  **Supported Encodings:** The set of character encodings supported by `new TextDecoder(encoding)` is not standardized and is a primary fingerprinting vector. The availability of specific encodings (e.g., `gbk`, `shift_jis`, `ibm866`) depends on the browser's build configuration and the International Components for Unicode (ICU) library it was compiled with. This creates a detailed fingerprint of the browser's internationalization capabilities, which often differs between Chrome on Windows, macOS, and Linux.

2.  **Canonicalization of Labels:** The way a browser canonicalizes encoding labels can be unique. For example, passing `'UTF8'` (uppercase) to the constructor might result in the `encoding` property being `'utf-8'`. Testing a list of non-canonical labels and observing the resulting canonical name adds entropy.

3.  **Malformed Input Handling:** While the spec defines behavior for malformed byte sequences (often inserting a `U+FFFD` replacement character `�`), subtle differences can exist between browser engines (V8, SpiderMonkey, JavaScriptCore) in how they handle specific edge cases for less common encodings. This can be used to distinguish not just browser families but sometimes specific versions.

4.  **Bot Detection (Integrity Checks):** Anti-bot systems heavily rely on checking the integrity of native browser APIs.
    *   **`toString()` Spoofing:** Automation frameworks like Puppeteer and Playwright often modify native APIs to hide their presence. A common check is `TextDecoder.toString()`. In a real Chrome browser, this returns `"function TextDecoder() { [native code] }"`. Any other result, such as a plain function definition, indicates that the constructor has been tampered with.
    *   **Constructor Behavior:** Calling the constructor with an invalid label must throw a `RangeError`. Bots might incorrectly patch the constructor, causing it to fail differently or not at all.

## Sample Code
This snippet demonstrates checking for a list of supported encodings and their canonical names, a common fingerprinting technique.

```javascript
/**
 * Checks for supported encodings and their canonical names to generate a fingerprint.
 * @returns {Promise<string>} A hash representing the encoding support profile.
 */
async function getEncodingFingerprint() {
  const encodingsToTest = [
    'utf-8', 'ibm866', 'iso-8859-2', 'koi8-r', 'macroman',
    'windows-1251', 'x-mac-cyrillic', 'gbk', 'gb18030',
    'hz-gb-2312', 'big5', 'euc-jp', 'iso-2022-jp', 'shift_jis',
    'euc-kr', 'utf-16be', 'utf-16le', 'x-user-defined'
  ];

  const supportProfile = {};

  for (const encoding of encodingsToTest) {
    try {
      const decoder = new TextDecoder(encoding);
      // Check if the encoding is supported and what its canonical name is.
      if (decoder.encoding) {
        supportProfile[encoding] = decoder.encoding;
      } else {
        supportProfile[encoding] = null; // Supported but no canonical name reported
      }
    } catch (e) {
      // A RangeError indicates the encoding is not supported.
      supportProfile[encoding] = false;
    }
  }

  // The resulting JSON string can be hashed to form a stable fingerprint component.
  const profileString = JSON.stringify(supportProfile);
  
  // Using SubtleCrypto for a robust hash (in a real scenario)
  // For demonstration, we just return the string.
  console.log(profileString);
  return profileString; 
}

getEncodingFingerprint();
// Example output on a specific Chrome version:
// {"utf-8":"utf-8","ibm866":"ibm866", ... ,"gbk":"gbk", ... ,"x-user-defined":"x-user-defined"}
```