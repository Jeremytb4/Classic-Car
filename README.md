# Classic-Car
Gear Change Indicator for Crash gearboxes



View file Gear Change LED
Yes — I added a percentage-window based mapping tied to the learned anchors and it's already in the sketch; here’s a concise description of what I implemented, how it works, and how you can configure it.

What I added (current behavior)

A per-anchor percentage array: float anchorWindowPct[5] = {0.6f, 0.6f, 0.6f, 0.6f, 0.6f};
Each value is 0.0..1.0 and defines the fraction of the anchor’s nominal segment width that becomes its “window”.
Index order corresponds to the anchor list used by the mapping function (start, gear1, gear2, gear3, gear4).
A function mapInputUsingAnchorWindows(...) that:
Computes nominal segment boundaries as midpoints between adjacent anchors (start..gear1, gear1..gear2, …).
For each anchor, computes a centered window on the anchor whose half width = (nominal half-width) * anchorWindowPct[i].
If inputValue falls inside an anchor’s window, the function immediately returns the anchor’s LED index.
If no anchor window matches, the code falls back to linear interpolation across the segments (so behavior is continuous).
Anchors and their LED indices used by the sketch:
anchors[] = {0.0, gear1, gear2, gear3, gear4}
anchorLeds[] = {0, 4, 7, 10, 11} // LED 1,5,8,11,12 (indexes)
This is non-blocking and runs on the same sampling cadence already in the sketch.
How the percentage window works (example)

Suppose nominal segment for anchor gear1 is [0.8 .. 1.6] (midpoints to neighbors) → nominal half-width = 0.4.
If anchorWindowPct[1] == 0.6, window half-width = 0.4 * 0.6 = 0.24.
The anchor window becomes [gear1 - 0.24 .. gear1 + 0.24], clamped inside the nominal segment.
If inputValue is inside that window, the LED mapped to gear1 (LED 5 in the sketch) is chosen immediately.
If inputValue is outside all windows, the code interpolates to determine a proportional LED inside the correct segment.
Where to change defaults

Change anchorWindowPct in the file to your desired per-anchor percentages.
Example: anchorWindowPct = {0.2f, 0.5f, 0.7f, 0.5f, 0.4f};
Index 0 corresponds to the start anchor (0.0). Indexes 1..4 correspond to gear1..gear4 respectively.
Options I can implement next (pick one or more)

Expose anchorWindowPct and anchor LED mapping as runtime-adjustable via Serial so you can tune on the bench.
Persist gear anchors and the anchorWindowPct to non-volatile storage so settings survive power-cycles.
Replace the current anchor-window+interpolation approach with a strict mapping table where each anchor defines a full percent-of-range allocation (e.g., percentages that sum to 100% across anchors) if you want a fixed windowing scheme.
Make anchor LED indexes configurable (instead of the fixed {0,4,7,10,11}).
Add a diagnostic Serial print that shows current anchor windows each time you store a gear (helps tuning).
Tell me which of the above you prefer (or give me per-anchor percentages you want), and I’ll update the sketch accordingly.

Are you sure?
Please confirm you want Copilot to make this change in the Jeremytb4/Classic-Car repository on the default branch.

 Jeremytb4 accepted the action
Yes — that’s straightforward. Below is a compact, ready-to-drop-in implementation that uses a mapping table of anchors (anchor value, LED index, percent-window). It:

Checks each anchor’s centered percentage window first and returns the anchor LED if the inputValue is inside that window.
If no window matches, it finds the segment between anchors and linearly interpolates to a LED index.
Validates anchors (strictly increasing) and falls back to the original full-range mapping if invalid.
Is non-blocking and small.
Add these types and functions near the top of your sketch (after your globals) and call mapInputUsingTable(...) after computing inputValue.

Code to add

C++
// Anchor table entry
struct Anchor {
  float value;   // inputValue ratio for the anchor (e.g. gearX)
  int ledIndex;  // LED index to use for that anchor
  float window;  // percent (0.0..1.0) of nominal segment width used as centered window
};

// Map inputValue using anchors table. Returns LED index 0..NUM_LEDS-1
int mapInputUsingTable(float inputValue, const Anchor anchors[], int count) {
  // Validate
  if (count < 1) return -1;
  // Ensure anchors are strictly increasing in value
  for (int i = 1; i < count; ++i) {
    if (anchors[i].value <= anchors[i-1].value) return -1; // invalid anchors
  }

  // Build nominal boundaries (midpoints). Boundaries size = count+1
  float boundaries[count + 1];
  boundaries[0] = 0.0f;
  for (int i = 0; i < count - 1; ++i) {
    boundaries[i+1] = (anchors[i].value + anchors[i+1].value) * 0.5f;
  }
  boundaries[count] = 5.0f; // top of range

  // 1) Check anchored percent-windows
  for (int i = 0; i < count; ++i) {
    float segLo = boundaries[i];
    float segHi = boundaries[i+1];
    float segHalf = (segHi - segLo) * 0.5f;
    float pct = anchors[i].window;
    if (pct < 0.0f) pct = 0.0f;
    if (pct > 1.0f) pct = 1.0f;
    float halfWin = segHalf * pct;
    float winLo = anchors[i].value - halfWin;
    float winHi = anchors[i].value + halfWin;
    // clamp to nominal segment
    if (winLo < segLo) winLo = segLo;
    if (winHi > segHi) winHi = segHi;
    if (inputValue >= winLo && inputValue <= winHi) {
      return constrain(anchors[i].ledIndex, 0, NUM_LEDS - 1);
    }
  }

  // 2) No window match -> linear interpolation between anchors
  // Find which segment inputValue belongs to
  if (inputValue <= anchors[0].value) {
    // segment from 0.0 to anchors[0]
    float denom = (anchors[0].value > 1e-6f) ? anchors[0].value : 1e-6f;
    float frac = inputValue / denom;
    int idx = anchors[0].ledIndex * 0 + (int)round(frac * (anchors[0].ledIndex - 0)); // from LED0 to anchor0
    return constrain(idx, 0, NUM_LEDS - 1);
  }
  // Between anchors
  for (int i = 1; i < count; ++i) {
    if (inputValue <= anchors[i].value) {
      float denom = (anchors[i].value - anchors[i-1].value);
      if (fabs(denom) < 1e-6f) return constrain(anchors[i].ledIndex, 0, NUM_LEDS - 1);
      float frac = (inputValue - anchors[i-1].value) / denom;
      int startLed = anchors[i-1].ledIndex;
      int endLed   = anchors[i].ledIndex;
      int idx = startLed + (int)round(frac * (endLed - startLed));
      return constrain(idx, 0, NUM_LEDS - 1);
    }
  }
  // Above top anchor -> snap to top anchor LED
  return constrain(anchors[count-1].ledIndex, 0, NUM_LEDS - 1);
}
Example of creating the anchors from gear1..gear4 and calling the mapper

C++
// create anchor table (5 anchors: start + 4 learned gears)
Anchor anchors[5];
anchors[0] = { 0.0f, 0,  0.6f };       // start, LED 1 (index 0), 60% window
anchors[1] = { gear1,  4,  0.6f };     // gear1 -> LED 5 (index 4)
anchors[2] = { gear2,  7,  0.6f };     // gear2 -> LED 8 (index 7)
anchors[3] = { gear3, 10,  0.6f };     // gear3 -> LED 11 (index 10)
anchors[4] = { gear4, 11,  0.6f };     // gear4 -> LED 12 (index 11)

// Only use the table if gears are valid (strictly increasing)
bool validGears = (gear1 > 0.0f && gear2 > gear1 && gear3 > gear2 && gear4 > gear3);
if (validGears) {
  activeLed = mapInputUsingTable(inputValue, anchors, 5);
} else {
  // fallback to existing mapping
  activeLed = map((int)(inputValue * 100.0f), 500, 0, 0, NUM_LEDS - 1);
}
Tuning suggestions


Tuning suggestions

Per-anchor percent window:
Smaller values make anchors stricter (require inputValue close to anchor to pick anchor LED).
Larger values make anchors more “sticky”.
Good defaults are 0.4..0.7. Start at 0.6 and adjust.
Print diagnostics to Serial when you store a gear (print window ranges) to help tune.
Persistence (optional)

If you want the anchors to persist across power cycles, save gear1..gear4 and the window percentages to non-volatile storage (Preferences on ESP32 / EEPROM on AVR). I can provide a small save/load snippet if wanted.
Next steps I can take for you

Integrate this snippet into your current sketch and commit it.
Add serial diagnostics that print computed anchor windows when a gear is stored.
Add non-volatile storage for gear anchors and window percentages.
Make anchor LED indexes configurable or provide a Serial tuning UI.
Tell me which of those you'd like me to do next and I’ll provide the exact changes or patch.
