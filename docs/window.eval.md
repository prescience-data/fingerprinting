---
title: window.eval
layout: default
---
# window.eval
## Purpose
The `eval()` function is a global property of the `window` object that parses and executes a string of JavaScript code within the current execution scope.

## Fingerprinting Usage
While `eval()` itself does not provide high-entropy information, its primary relevance to bot detection lies in **integrity checks**. Automation frameworks like Puppeteer and Playwright often need to override or "hook" native browser functions to hide their presence or alter behavior. `eval()` is a prime target for such modifications.

Anti-bot scripts detect these modifications using several methods:

1.  **`toString()` Inconsistency:** The most common check. In a standard Chrome browser, `window.eval.toString()` returns the string `"function eval() { [native code] }"`. If a bot framework has wrapped `eval()` in a custom JavaScript function, the `toString()` result will be the source code of that wrapper function, which is a definitive sign of tampering.

2.  **Property Mismatches:** Native functions have specific properties.
    *   `eval.length`: The arity (number of expected arguments) of the native `eval` function is `1`. A custom wrapper, especially one using rest parameters like `(...args)`, might have a length of `0`.
    *   `eval.name`: The name property will be `"eval"`. A wrapper function might be anonymous or have a different name.

3.  **Stack Trace Analysis:** Executing code via `eval()` that intentionally throws an error can reveal the nature of the execution environment.
    *   `try { eval('throw new Error()') } catch (e) { console.log(e.stack) }`
    *   In a bot environment (e.g., Node.js running Puppeteer), the stack trace format, file paths (`file:///...`), or line numbers may differ significantly from a genuine browser environment. The presence of Node.js-specific internal module paths is a strong giveaway.

## Sample Code
This snippet demonstrates a typical integrity check used by anti-bot systems to verify if `eval()` has been modified from its native state.

```javascript
function isEvalTampered() {
  // 1. Check the string representation of the function.
  // Native functions will report "[native code]".
  const isNativeCode = eval.toString() === 'function eval() { [native code] }';

  // 2. Check the function's arity (expected number of arguments).
  // Native eval expects exactly one argument.
  const hasCorrectLength = eval.length === 1;

  // If either check fails, the function has likely been tampered with.
  if (!isNativeCode || !hasCorrectLength) {
    return true; // Detected tampering
  }

  return false; // Appears to be native
}

if (isEvalTampered()) {
  console.log('Bot-like behavior detected: window.eval has been modified.');
  // Trigger anti-bot countermeasures
}
```