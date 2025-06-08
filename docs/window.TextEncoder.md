---
title: window.TextEncoder
layout: default
---
# window.TextEncoder
## Purpose
The `TextEncoder` interface, part of the Encoding Standard, converts a DOMString into a stream of bytes using the UTF-8 encoding scheme, returning a `Uint8Array`. It is the modern, standardized counterpart to older, non-standard string encoding methods.

## Fingerprinting Usage
The direct output of `TextEncoder` for a given string is strictly specified by the UTF-8 standard and is therefore not a source of entropy across compliant browsers. However, its implementation and performance characteristics can be used for fingerprinting and bot detection.

*   **Performance Timing:** The execution speed of the `encode()` method can be a significant signal. The underlying implementation is native code within the browser's engine (e.g., V8, SpiderMonkey, JavaScriptCore). Timing how long it takes to encode a large and complex string can help differentiate between browser engines, browser versions, and underlying hardware. Bots running in emulated or less optimized environments (like some versions of Node.js or other server-side JS runtimes) may exhibit different performance profiles.

*   **Implementation Quirks & Error Handling:** While the standard is strict, non-compliant environments may differ. A key test is feeding the encoder invalid sequences, such as lone surrogate pairs (e.g., `\uD800`). A compliant browser will replace this with the replacement character (`U+FFFD`), which encodes to `0xEF 0xBF 0xBD`. A non-compliant or buggy implementation in a bot environment might throw an error or produce a different byte sequence, revealing itself.

*   **Property and Prototype Analysis:** The properties of the `TextEncoder` constructor and its prototype (`TextEncoder.prototype`) can be enumerated. The presence, absence, or order of properties like `encode`, `encodeInto`, and `encoding` can differ between JavaScript engines or indicate modification by an automation framework's instrumentation. For example, `TextEncoder.prototype.constructor.name` should be `"TextEncoder"`.

*   **Existence Check:** In very basic bot environments that are not full browser emulations (e.g., simple HTTP clients with a JS engine), the `window.TextEncoder` API may be missing entirely. Its absence is a strong signal of a non-browser environment.

## Sample Code
This snippet demonstrates performance timing and error handling for a lone surrogate, which are common signals for fingerprinting scripts.

```javascript
async function getEncoderFingerprint() {
  const signals = {};

  if (typeof window.TextEncoder !== 'function') {
    return { error: 'TextEncoder_not_found' };
  }

  try {
    // 1. Performance Timing
    const largeString = ('\u{1F600} \u{1F4A9} text-to-encode ').repeat(10000);
    const encoder = new TextEncoder();
    const startTime = performance.now();
    encoder.encode(largeString);
    const endTime = performance.now();
    signals.encodeTime = endTime - startTime; // This duration is a fingerprinting signal

    // 2. Lone Surrogate Handling
    // A compliant browser will replace the lone surrogate with U+FFFD (ef bf bd)
    const loneSurrogate = '\uD83D'; // An unpaired high surrogate
    const encoded = encoder.encode(loneSurrogate);
    signals.loneSurrogateOutput = Array.from(encoded).join(','); // Expected: "239,191,189"

  } catch (e) {
    // An error here is a strong anomaly signal
    signals.error = e.name + ': ' + e.message;
  }

  // The `signals` object would be sent to a server for analysis.
  // e.g., { encodeTime: 12.345, loneSurrogateOutput: "239,191,189" }
  return signals;
}

getEncoderFingerprint().then(fp => console.log(fp));
```