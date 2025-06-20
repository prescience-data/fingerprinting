---
title: window.WheelEvent
layout: default
---
# window.WheelEvent
## Purpose
Represents events that occur due to the user moving a mouse wheel or a similar input device like a trackpad. It provides data about the direction and magnitude of the scroll action.

## Fingerprinting Usage
The `WheelEvent` is a rich source of behavioral and hardware-specific data, making it highly valuable for bot detection and fingerprinting. Analysis focuses on the sequence and properties of events, not just a single occurrence.

*   **Hardware & OS Fingerprinting:** The `deltaY` values generated by scrolling differ significantly across hardware and operating systems.
    *   **High-Precision Trackpads (e.g., macOS):** Produce a rapid stream of events with small, often fractional `deltaY` values (e.g., -1.25, -2.5, -3.75).
    *   **Standard Mice (e.g., Windows):** Typically produce events with larger, discrete `deltaY` values that are often multiples of a base number (e.g., 100 or 120), corresponding to physical "clicks" of the wheel.
    *   **`deltaMode` Property:** This property (`0` for pixels, `1` for lines, `2` for pages) provides additional entropy, as its default and behavior can vary with the device and browser settings. A User-Agent claiming to be macOS but sending Windows-style wheel events is a strong anomaly.

*   **Behavioral Analysis:** Human scrolling is non-uniform and complex, whereas bot-generated scrolling is often simplistic and robotic.
    *   **Timing and Velocity:** The `timeStamp` difference between consecutive wheel events reveals the user's scrolling speed, acceleration, and deceleration. Humans rarely scroll at a perfectly constant rate.
    *   **Inertial Scrolling (Coasting):** Many trackpads and touch devices exhibit inertia. A user's flick generates an initial burst of events, followed by a series of events with decreasing `deltaY` values as the scroll coasts to a stop. Bots often fail to replicate this natural decay pattern.
    *   **Event Frequency:** A bot might try to scroll a page by dispatching a single event with a very large delta value, or a series of events with perfectly regular timing, both of which are highly unnatural.

*   **`isTrusted` Flag:** This is a fundamental anti-bot check.
    *   A genuine `WheelEvent` initiated by the user has `event.isTrusted` set to `true`.
    *   An event created and dispatched programmatically via JavaScript (e.g., `element.dispatchEvent(new WheelEvent(...))`) will have `event.isTrusted` set to `false`. While sophisticated bots can bypass this using lower-level automation (like Chrome DevTools Protocol), it defeats simple script-based automation.

## Sample Code
This snippet collects a series of wheel events to analyze the pattern, which is more effective than inspecting a single event.

```javascript
const scrollEvents = [];
let scrollTimeout;

// Listen for wheel events on the document
document.addEventListener('wheel', (event) => {
  // Check 1: Is the event triggered by a real user?
  if (!event.isTrusted) {
    console.log('Bot detected: Untrusted wheel event.');
    return;
  }

  // Collect relevant data points for behavioral analysis
  scrollEvents.push({
    deltaY: event.deltaY,
    deltaMode: event.deltaMode,
    timeStamp: event.timeStamp,
  });

  // After scrolling stops, analyze the collected gesture
  clearTimeout(scrollTimeout);
  scrollTimeout = setTimeout(() => {
    analyzeScrollBehavior(scrollEvents);
    scrollEvents.length = 0; // Clear for the next gesture
  }, 500);
}, { passive: true });

function analyzeScrollBehavior(events) {
  if (events.length < 3) return; // Not enough data for a gesture

  console.log(`Analyzing scroll gesture with ${events.length} events.`);

  // Check 2: Look for robotic, uniform timing
  const timeDiffs = [];
  for (let i = 1; i < events.length; i++) {
    timeDiffs.push(events[i].timeStamp - events[i-1].timeStamp);
  }
  const isTimingRobotic = timeDiffs.every(diff => diff === timeDiffs[0]);
  if (isTimingRobotic) {
    console.log('Anomaly detected: Scroll timing is perfectly uniform.');
  }

  // Check 3: Differentiate hardware based on delta values
  const averageDelta = Math.abs(events.reduce((acc, e) => acc + e.deltaY, 0) / events.length);
  if (events[0].deltaMode === 0 && averageDelta < 10) {
    console.log('Fingerprint: Likely a high-precision trackpad (macOS-style).');
  } else if (events[0].deltaMode === 0 && averageDelta > 50) {
    console.log('Fingerprint: Likely a standard mouse wheel (Windows-style).');
  }
}
```