---
title: window.Error
layout: default
---
# window.Error
## Purpose
The `Error` constructor creates error objects. These objects are thrown when runtime errors occur and can also be used for user-defined exceptions. Their primary feature for analysis is the `stack` property, which provides a trace of the function calls that led to the error's creation.

## Fingerprinting Usage
The `Error` object is a high-entropy source for fingerprinting and a critical tool for detecting bots and automation frameworks. Its utility comes almost entirely from the non-standardized and implementation-specific `error.stack` property.

1.  **Engine-Specific Stack Trace Format**: The format of the stack trace string is highly specific to the JavaScript engine. V8 (Chrome), SpiderMonkey (Firefox), and JavaScriptCore (Safari) all produce distinctly formatted strings. Analyzing the structure (e.g., the presence of "at", the formatting of anonymous functions, the parentheses around file paths) is a primary method for identifying the underlying browser engine. A bot claiming to be Chrome but providing a Firefox-style stack trace is an immediate red flag.

2.  **Source Code Integrity Check**: This is a powerful anti-bot technique. A script can intentionally create an error and inspect its own stack trace. Since the script knows the exact line and column number where the `new Error()` call occurs, it can parse the stack trace and verify these coordinates. If a bot or researcher has modified the script (e.g., by running it through a beautifier or deobfuscator), the line and column numbers will change, revealing the tampering.

3.  **Automation Framework Artifacts**: Headless browsers and automation frameworks like Puppeteer and Playwright often inject their own scripts or execute user code in a specific context. This can leave artifacts in the call stack. For example, file paths or function names like `__puppeteer_evaluation_script__` or `playwright` may appear in the stack trace, directly revealing the presence of automation.

4.  **V8-Specific APIs**: V8 provides a non-standard API, `Error.prepareStackTrace`. This function, if defined, is called by the engine when the `.stack` property of an error is accessed. Fingerprinting scripts can check for the existence and behavior of this API to confirm they are running on V8. Bots attempting to emulate a V8 environment may fail to correctly implement this proprietary feature.

5.  **Property Descriptors**: In V8, the `stack` property is a non-enumerable, lazy-loaded accessor property on the `Error` instance. Scripts can check this using `Object.getOwnPropertyDescriptor(new Error(), 'stack')`. If a bot uses a naive proxy or shim to fake the `Error` object, it may incorrectly define `stack` as an enumerable or a simple data property, which is a detectable anomaly.

## Sample Code
This snippet demonstrates the source code integrity check. It creates an error and uses a regular expression to parse its own line number from the stack trace, comparing it to a hardcoded expected value.

```javascript
function checkErrorStackIntegrity() {
  // This line number must be known beforehand. Let's assume this function
  // starts on line 100 of the script. This call is on line 102.
  const EXPECTED_LINE = 102;

  try {
    const err = new Error();
    const stack = err.stack;

    // V8 stack trace line format: " at functionName (source:line:column)"
    // We grab the second line of the stack (the first is the error message).
    const stackLine = stack.split('\n')[1];

    // Extract the line number.
    const match = stackLine.match(/:(\d+):\d+\)?$/);
    
    if (!match) {
      console.log('Stack format is unexpected. Likely not V8 or is spoofed.');
      return false;
    }

    const lineNumber = parseInt(match[1], 10);

    if (lineNumber === EXPECTED_LINE) {
      console.log('Stack integrity check passed.');
      return true;
    } else {
      console.log(`Stack integrity check failed! Expected line ${EXPECTED_LINE}, but got ${lineNumber}. Code has been modified.`);
      return false;
    }
  } catch (e) {
    // If .stack doesn't exist or fails, it's a red flag.
    console.log('Error accessing stack property. Environment is suspicious.');
    return false;
  }
}

// In a real scenario, the EXPECTED_LINE would be calculated by a build tool
// and injected into the script. For this example, you'd need to adjust
// the number based on where you paste this code.
checkErrorStackIntegrity();
```