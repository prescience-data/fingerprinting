---
title: window.RegExp
layout: default
---
# window.RegExp
## Purpose
The `window.RegExp` constructor creates regular expression objects for matching text with a pattern. It is a fundamental, standardized part of JavaScript's text processing capabilities.

## Fingerprinting Usage
While `RegExp` is standardized by ECMA-262, its implementation details can vary significantly between JavaScript engines (V8, SpiderMonkey, JavaScriptCore) and even between different versions of the same engine. This makes it a high-entropy source for fingerprinting.

1.  **Error Message Fingerprinting**: The exact text of error messages thrown for invalid regular expressions is not strictly defined by the ECMAScript specification. V8, for instance, has a very specific format for its `SyntaxError` messages which can be used to identify the engine and often its version.
    *   **Signal**: `new RegExp('(')` might throw `"Invalid regular expression: /(: Unmatched '('"` in V8, while other engines produce different strings. Changes to this string across V8 versions provide additional entropy.

2.  **`toString()` Serialization**: The `toString()` method of a RegExp object serializes it back to a string literal. While modern engines have converged on a standard format (e.g., `/pattern/flags`), historical or non-compliant engines might have subtle differences in spacing or flag order, which can be a tell-tale sign.

3.  **Feature Support**: The set of supported RegExp features is a powerful fingerprinting vector. As new features are added to the ECMAScript standard, their presence or absence can pinpoint a browser's engine version with high accuracy.
    *   **Key Features for Detection**:
        *   **Lookbehind assertions**: `(?<=...)` and `(?<!...)`
        *   **Named capture groups**: `(?<name>...)`
        *   **Unicode property escapes**: `\p{Script=Greek}`
        *   **RegExp `d` flag (indices)**: Provides start and end indices for matched substrings.
    *   **Bot Detection**: This is a primary method for detecting inconsistencies. A bot may spoof its User-Agent string to claim it is the latest version of Chrome, but a simple test for a recent RegExp feature (like the `v` flag with set notation) can instantly reveal that its underlying engine is older (e.g., a specific version of Puppeteer/Headless Chrome). This mismatch is a high-confidence bot signal.

4.  **Performance Characteristics**: *[Speculative but used in advanced systems]* The execution time of highly complex or specially crafted regular expressions can differ based on the engine's internal optimization strategies (e.g., DFA vs. NFA, JIT compilation). Measuring the execution time of a "canary" regex can provide a behavioral signal, though it is susceptible to noise from system load.

## Sample Code
This snippet demonstrates fingerprinting through error messages and feature support, which are common techniques used by anti-bot systems.

```javascript
function getRegExpFingerprint() {
  const results = {};

  // 1. Capture the specific error message for an invalid pattern
  try {
    new RegExp('(?!)');
  } catch (e) {
    // V8 (Chrome): "Invalid regular expression: /(?!)/: Invalid group"
    // SpiderMonkey (Firefox): "SyntaxError: invalid identity escape in regular expression"
    results.errorMessage = e.message;
  }

  // 2. Test for modern feature support (e.g., RegExp 'd' flag)
  try {
    new RegExp('a', 'd');
    results.hasIndicesFlag = true;
  } catch (e) {
    results.hasIndicesFlag = false;
  }
  
  // 3. Test for named capture group support
  try {
    new RegExp('(?<test>a)').exec('a');
    results.hasNamedGroups = true;
  } catch (e) {
    results.hasNamedGroups = false;
  }

  // The combination of these results provides a highly specific fingerprint.
  // A bot claiming to be Chrome 120 but returning `hasIndicesFlag: false` is exposed.
  return JSON.stringify(results);
}

// Example Output (Modern Chrome):
// '{"errorMessage":"Invalid regular expression: /(?!)/: Invalid group","hasIndicesFlag":true,"hasNamedGroups":true}'
```