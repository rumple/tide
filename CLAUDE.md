# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**Tide** ("Breath — a quiet pacer") is a single-file, dependency-free breathing pacer web app. The entire application lives in `breath.html` — no build step, no package manager, no framework. To run it, open `breath.html` in any modern browser.

## Development workflow

There is no build, lint, or test toolchain. Changes are made directly to `breath.html` and verified by opening the file in a browser.

To test on mobile: serve the file over a local network (e.g. `python3 -m http.server`) and open the IP on a phone. Haptics and audio require a real device gesture to unlock, so can't be verified headlessly.

## Architecture

All code is in a single IIFE in `breath.html`:

- **CSS** (~lines 7–191): Custom properties on `:root` drive all animation. JS writes `--s` (scale), `--glow`, `--orb`, and `--orb-edge` directly via `root.style.setProperty(...)` every animation frame.
- **HTML** (~lines 192–265): Semantic markup with ARIA throughout. The settings panel is a slide-in drawer (`role="dialog"`, `aria-modal`), revealed by toggling `.open` + removing `hidden`.
- **JS** (~lines 266–533): One IIFE, no modules.

### Breathing engine (the core loop)

`buildSeq()` converts the 4-element `pattern` array `[inhale, holdIn, exhale, holdOut]` into a sequence of phase objects `{type, label, dur}`. Phase types are `'inhale'`, `'hold'`, `'exhale'`, `'holdOut'`.

`enter(i)` starts a phase: records `phaseStart`, updates label text, fires audio/haptic cue.

`loop(now)` is the `requestAnimationFrame` callback. It computes `p = elapsed / dur` (0→1), applies easing to get orb scale, updates the SVG progress ring via `stroke-dashoffset`, and advances to the next phase when `p >= 1`. Breaths are counted when an `'exhale'` phase completes.

Easing: inhale uses `easeOut` (`1 - (1-p)^2.2`), exhale uses `easeIn` (`p^2.2`). Hold phases lock the orb at max/min scale.

### Audio

Web Audio API. `tone(freqs, dur, gainPeak)` creates oscillator(s) with a linear amplitude envelope (attack ~120ms, release to 0 at `dur`). When two frequencies are passed, they cross-fade: first freq fades out while second fades in over 60% of `dur` — creating pitch sweeps (e.g. inhale: 392→523 Hz, exhale: 440→294 Hz).

`AudioContext` is lazy-initialized on first `start()` to satisfy browser autoplay policy.

### Haptics

- **Android**: Vibration API with patterns (`navigator.vibrate`)
- **iOS**: Clicks a hidden `<input type="checkbox" switch>` to trigger UIKit haptics — a workaround Apple patched in iOS 26.5. The `isiOS` detection covers iPhone/iPad/iPod and Mac+touch.

### State

Global variables within the IIFE: `pattern` (active timing array), `activePreset`, `cues` (`{sound, vibe, visual}`), `running`, `seq`, `idx`, `phaseStart`, `breaths`, `raf` (animation frame handle), `startTime`.

### Drawer

Hidden with `hidden` attribute + `translateX(100%)`. Opening: remove `hidden`, force a reflow (`offsetHeight`), add `.open`. Closing: remove `.open`, then set `hidden` in a `transitionend` listener. Backdrop click, Escape key, and swipe-right all close it.

## Key conventions

- **CSS custom properties** control all dynamic visual state; never set `.style.transform` or `.style.opacity` directly on the orb/halo — set `--s` and `--glow` and let CSS compute the rest.
- **ARIA** is maintained throughout: `aria-pressed` on chips, `aria-expanded` on the settings button, `aria-live="polite"` on the phase display. Keep these in sync when adding interactive elements.
- **No zero-duration phases**: `buildSeq()` skips `holdIn`/`holdOut` when their duration is 0. Any new phase type should follow this pattern.
- **Pattern is always `.slice()`d** before use to avoid mutating the preset array.
- The `$` helper is `document.querySelector`; there is no `$$` — use `document.querySelectorAll` directly.
- **Bump `VERSION`** (the `const VERSION` near the top of the IIFE) with every change. It's shown as `v<VERSION>` at the bottom of the settings drawer. Use semver: patch for fixes/tweaks, minor for new features.
