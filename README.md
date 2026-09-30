# Breathing Light Animation — Bézier Easing Curves

Claude'd into CSS

## Background

The MacBook "breathing light" sleep indicator's brightness pattern was measured and analyzed by Avital Pekker ([full write-up](https://avital.ca/notes/a-closer-look-at-apples-breathing-light)) using a photodiode and a BeagleBone Black ADC. The smoothed data was fit to a Gaussian curve via `scipy.curve_fit`, yielding:

- **mean** = 18.045
- **sigma** = 48.033

(in raw sample-index units from the original measurement — not directly usable as animation timing).

The final LED-driving software didn't use that raw fit directly. Instead it used a **piecewise function of two different Gaussian halves** over a 5-second breath cycle:

- A **narrower, faster** Gaussian half for the brighten (inhale) phase
- A **wider, slower** Gaussian half for the dim (exhale) phase

split according to the human inspiration:expiration (I:E) ratio, which for a healthy adult at rest ranges roughly **1:1.5 to 1:2** (exhale takes longer than inhale).

## Converting to Bézier Curves

Each half-Gaussian (from a tail cutoff up to the peak, or peak down to a tail cutoff) is itself an S-curve — slow, then fast, then slow again — the same shape a cubic Bézier easing curve produces. Each half-Gaussian was normalized to a `[0,1] × [0,1]` box and least-squares fit to a standard cubic Bézier (`P0 = (0,0)`, `P3 = (1,1)`, solving for control points `P1`, `P2`).

Shape parameters used: `0.42` (inhale — sharper) and `0.60` (exhale — more gradual), chosen to match the article's "narrower vs. wider" description since the original post doesn't publish exact final sigma values for its implementation.

### Results

| Phase | P1 | P2 | Max fit error |
| --- | --- | --- | --- |
| **Inhale (brighten)** | `(0.597, 0.011)` | `(0.686, 0.961)` | \~0.4% |
| **Exhale (dim)** | `(0.314, 0.039)` | `(0.403, 0.989)` | \~0.4% |

As `cubic-bezier()` timing functions:

```
Inhale: cubic-bezier(0.597, 0.011, 0.686, 0.961)
Exhale: cubic-bezier(0.314, 0.039, 0.403, 0.989)
```

**Note on direction:** both curves run progress `0 → 1`. For the exhale phase, the *value* is interpolated from high → low across that same 0→1 progress — the curve shape doesn't reverse, only the value mapping does.

## Suggested Timing

Using an I:E ratio of \~1:1.75 within a 5-second total breath cycle:

| Segment | Duration | % of cycle |
| --- | --- | --- |
| Inhale (brighten) | \~1.6s | 32% |
| Exhale (dim) | \~2.8s | 56% |
| Pause (held at baseline) | \~0.6s | 12% |

## CSS Implementation

CSS keyframes only support one `animation-timing-function` per animation by default, but you can override it **per-keyframe** so each segment eases independently:

```css
.breathing-light {
  animation: breathe 5s infinite;
}

@keyframes breathe {
  0% {
    opacity: 0.04; /* baseline */
    animation-timing-function: cubic-bezier(0.597, 0.011, 0.686, 0.961); /* inhale easing */
  }
  32% {
    opacity: 1; /* peak brightness */
    animation-timing-function: cubic-bezier(0.314, 0.039, 0.403, 0.989); /* exhale easing */
  }
  88% {
    opacity: 0.04; /* back to baseline, dim phase complete */
    animation-timing-function: linear; /* hold flat through the pause */
  }
  100% {
    opacity: 0.04; /* pause ends, cycle restarts */
  }
}
```

### Notes

- `0%` sets the timing function that governs the segment *starting* there (i.e. the inhale), matching how CSS applies `animation-timing-function` to the interval following each keyframe.
- The `88%–100%` segment is a flat hold representing the pause between breaths; `linear` (or omitting a timing function) keeps it flat since there's no value change to ease.
- Adjust the `32%` / `88%` breakpoints if you change the I:E ratio or total cycle duration.

## Tuning

The `0.42` / `0.60` shape parameters controlling curve sharpness are adjustable — smaller values produce a snappier transition, larger values a more gradual one. Re-run the fitting script (shared earlier in this conversation) with different `shape_sigma` arguments to regenerate control points for a different feel.