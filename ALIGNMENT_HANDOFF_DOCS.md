# ALIGNMENT — Code Handoff Documentation
## Version 0.22 (as of April 2026)

This document describes the current state of `alignment.html`, a single-file HTML5 combat game. If you're an AI or developer picking up this project, read this before making changes. The code has organically evolved over many iterations and has several non-obvious quirks that will break things if touched carelessly.

---

## 1. TOP-LEVEL ARCHITECTURE

**Single HTML file**, ~2800 lines, contains:
1. CSS (lines ~10-580) — all styles inline
2. HTML body (~580-635) — all game elements
3. JavaScript (~640-end) — all game logic

All code runs on page load. No build step. No external dependencies except **Tone.js** (loaded from CDN) for procedural music.

**Target platforms:** iOS Safari (primary), Chrome desktop, any modern browser.

**Game container:** `#game` div, max 480×854px, centered in viewport.

---

## 2. GAME FLOW (Top-Level State Machine)

```
TITLE (ENGAGE tap) → BOOT SEQUENCE → DISC FIGHT → [victory] → NINJA FIGHT → [victory] → KNIGHT FIGHT → WIN/LOSE
```

Each fight has its own reveal cinematic, HP bars, exchange loop, and defeat sequence.

---

## 3. STATE VARIABLES (All Declared as Module-Level `let`)

### Core game state

| Variable | Type | Values | Purpose |
|---|---|---|---|
| `state` | enum string | `'idle'`, `'telegraph'`, `'committed'`, `'stunned'`, `'closing'`, `'resetting'`, `'transition'`, `'won'`, `'lost'` | Current phase of active exchange |
| `currentEnemy` | string | `'ball'` (unused/disc), `'ninja'`, `'knight'` | Who you're fighting. Disc fight uses `'ball'` for legacy reasons. |
| `playerHealth` | number | 0-100 | Player HP |
| `enemyHealth` | number | 0-100 | Enemy HP |

**State transitions** (critical to understand):
- `IDLE` → enemy picks an attack → `TELEGRAPH` (parry window open)
- `TELEGRAPH` → player swipes correctly → parry lands → **combo check** (see below) → `STUNNED`
- `TELEGRAPH` → player swipes wrong / doesn't swipe → `COMMITTED` → apply damage → back to `IDLE`
- `STUNNED` → player taps/swipes to counter → damage enemy → `IDLE`
- `STUNNED` → window closes → `STUNNED_CLOSING` → fully `IDLE`

### Attack-direction state (knight specific)

| Variable | Type | Values | Purpose |
|---|---|---|---|
| `attackDirection` | string | `'downRight'`, `'downLeft'` | Which direction the knight's slash is going — randomized per attack |

### Parry state

| Variable | Type | Values | Purpose |
|---|---|---|---|
| `requiredParry` | string | `'counterDiagonal'`, `'right'`, `'left'`, `'any'` | What parry swipe is expected. Set by enemy logic before opening telegraph window. |
| `requiredCounter` | string | `'stab'`, `'hslash'`, `'vslash'`, `'dslash'` | Type of counter-attack gesture expected after parry. **Currently unused for matching — any tap/swipe lands the hit.** Still shown visually as an indicator. |

**IMPORTANT:** `'upLeft'`, `'up'`, and `'downRightFromAbove'` are legacy values from the removed aerial attack. They are NOT used anywhere in the current code. Do not add logic referencing them — the aerial attack has been cut.

### Ninja combo state

| Variable | Type | Values | Purpose |
|---|---|---|---|
| `comboTotal` | number | 1 or 2 | How many parries this exchange requires (always 2 for ninja, 1 for others) |
| `comboIndex` | number | 0 or 1 | Which parry we're currently on (0 = first) |
| `comboPattern` | string | `''`, `'ground'`, `'groundMirrored'` | Which ninja attack variant is currently running. Used by `onParrySuccess` to decide which continuation function to call. |

**The ninja attack is now:**
- `comboPattern === 'ground'`: first thrust LEFT, then second thrust RIGHT
- `comboPattern === 'groundMirrored'`: sprite flipped via CSS `scaleX(-1)`, first thrust RIGHT, then second thrust LEFT
- No aerial attack — that was removed. The `ninjaAerialExchange`, `ninjaAerialContinue` functions are GONE. Do not reintroduce references to them.

### Animation/timing state

| Variable | Type | Purpose |
|---|---|---|
| `activeTimers` | array of timer IDs | All pending `setTimeout` calls — cleared on exchange abort |
| `activeAnimations` | array of `{cancel: false}` objects | All active `playFramesAdvanced` animations — set `.cancel = true` to stop |
| `currentExchangeToken` | incrementing number | Incremented at start of each exchange. Async functions check `if (token !== currentExchangeToken) throw new Error('aborted')` to cancel cleanly when a new exchange starts. |
| `bootInProgress` | boolean | Whether the boot sequence is currently playing |
| `bootSkipped` | boolean | Set true when player taps to skip boot |

### Pointer/input state

| Variable | Type | Purpose |
|---|---|---|
| `pointerDown` | boolean | Is touch/mouse currently pressed |
| `pointerStart` | `{x, y, time}` | Where the current gesture began |
| `pointerPath` | array of `{x, y, time}` | All positions during current gesture |

### Damage constants (tunable)

| Constant | Value | Purpose |
|---|---|---|
| `DAMAGE_PLAYER` | 20 | How much HP player loses per hit taken |
| `DAMAGE_ENEMY` | 25 | How much HP enemy loses per counter landed |

---

## 4. THE ASSETS OBJECT — Critical Naming Conventions

**All filenames are CASE-SENSITIVE** because GitHub Pages serves files case-sensitive. Do not "fix" apparent inconsistencies in capitalization — they match the actual filenames.

```javascript
const ASSETS = {
  knightAttack:  // frame_000.png through frame_049.png (50 frames, lowercase, 3-digit pad)
  knightBlock:   // block_1.png through block_8.png (8 frames, 1-digit — NOTE: NO PADDING)
  knightRecover: // recover_frame_000.png through recover_frame_003.png (4 frames, 3-digit pad)
  discRotate:    // disc_rotate_frame_002.png through 008.png (frames 002-008 ONLY, lowercase d)
                 // Note: NO frames 000 or 001! Array has 7 entries.
  discBlock:     // Disc_block_frame_000.png through 015.png (16 frames, CAPITAL D)
  ninjaLeap:     // ninja_leap_frame_000.png through 024.png (25 frames, lowercase n)
                 // NOTE: Still loaded for the disc/ninja reveal animation; NOT used as attack.
  ninjaSideAttack: // Ninja_side_attack_frame_000.png through 024.png (25 frames, CAPITAL N)
  ninjaBlocked:  // Ninja_blocked_frame_000.png, then 008-024 (CAPITAL N)
                 // IMPORTANT: Array has 18 entries: index 0 = file 000, index 1 = file 008,
                 //            index 2 = file 009, ... index 17 = file 024
                 // Files 001-007 DO NOT EXIST. Do not try to reference them.
  ninjaRecover:  // ninja_recover_frame_000.png through 008.png (9 frames, lowercase n)
};
```

**Files NO LONGER referenced (removed from ASSETS object):**
- `ninja_downslash_frame_*.png` — aerial attack was cut, these are dead weight
- `attack_impact_frame_*.png` — old ball attack sprites, replaced by disc
- `block_return_frame_*.png` — old ball block sprites, replaced by disc

### Background images (not in ASSETS, set directly in HTML):

- `background_1.png` — Archive hallway (used for disc fight). Tall image, 896×1372.
- `cyber_hallway_9x16_no_text.png` — Sector 7 (used for ninja fight). **LANDSCAPE despite name**: 952×840. Position adjusted via CSS `background-position: center 35%`.
- `power_nexus_9x16_no_text.png` — Power Nexus (used for knight fight). **LANDSCAPE**: 952×840. Position `center 40%`.

### Audio files

```javascript
const SFX_FILES = {
  swings:  [...], // 8 files: 78678__joe93barlow__swing0.mp3 through swing7.mp3
  strikes: [...]  // 3 files: 78675__joe93barlow__strike0.mp3 through strike2.mp3
};
```

The naming (`78675-78685`) is from freesound.org — do not rename these.

---

## 5. CORE FUNCTIONS AND WHAT THEY DO

### Animation helpers

- **`playFramesAdvanced(el, frames, schedule)`**: Plays a sequence of frames with per-frame durations. `schedule` is an array of `{frame: index, duration: ms}`. Returns a Promise that resolves when the animation completes or is canceled. **Always await this, always check `token !== currentExchangeToken` after** to handle cancellation.
- **`wait(ms)`**: Promise wrapper around setTimeout. Same cancellation rules.
- **`clearAllTimers()`**: Kills all pending timers and animations. Called when exchange aborts.

### Input handlers

- **`onPointerDown/Move/Up(e)`**: Touch and mouse handlers. **Critical detail**: `preventDefault()` is ONLY called if the event target is NOT a button/input/link. This is essential for iOS Safari — calling `preventDefault` on `touchstart` cancels the synthesized click event that fires on buttons. Do NOT remove this check.
- **`handleTap()` / `handleSwipe(dx, dy, dist)`**: Called by onPointerUp based on distance threshold (12px during parry window, 18px otherwise).
- **`classifySwipe(dx, dy)` → returns angle in degrees**: 0°=right, 90°=down, 180°=left, -90°=up.

### Parry classifiers

- **`isCounterDiagonal(angle, strikeDirection)`** — for knight. Returns true if swipe is within 55° of target (-45° for ↘, -135° for ↙).
- **`isNinjaParry(angle, dist)`** — for ninja. Requires distance ≥ 25. Checks angle against current `requiredParry` value. Only accepts `'left'` (180°) or `'right'` (0°).
- **`isBallParry(dx, dy, dist)`** — for disc. Any swipe ≥ 60px passes. "Disc" uses `currentEnemy === 'ball'` for legacy reasons.

### Circular distance formula (used by both parry classifiers)

```javascript
const raw = Math.abs(angle - target) % 360;
const diff = raw > 180 ? 360 - raw : raw;
return diff <= 55; // within tolerance
```

**WARNING**: An earlier version had inverted this logic (`return diff > 180 - 55`). If parries stop working, check this formula — the correct behavior is `diff <= 55`, where `diff` is 0 when angles match exactly.

### Enemy exchange functions

Each follows the pattern: set telegraph → play animation → check state → apply outcome.

- **`ballExchange(token)`**: disc attack. Pick lane, drift, zoom. Any ≥60px swipe parries.
- **`knightExchange(token)`**: pick direction, play windup, hold, commit. One parry required.
- **`ninjaGroundExchange(token, mirrored)`**: 2-parry combo. First parry direction depends on `mirrored` param. When mirrored, the ninja sprite gets `.mirrored` CSS class which applies `scaleX(-1)`.
- **`ninjaComboContinue(token)`**: called by `onParrySuccess` after first parry lands. Reads `comboPattern` to decide direction.

### Parry success handler (`onParrySuccess`)

Big function. Flow:
1. Early return if state isn't TELEGRAPH
2. Play strike sound, commit flash
3. Emit particle burst
4. Hit-stop (80ms wait)
5. **Combo check**: if `comboTotal > 1 && comboIndex < comboTotal - 1`, increment index and call `ninjaComboContinue`. Return.
6. Otherwise, enter stun flow: play block animation, show counter indicator, wait for player input, play recover animation.

### Parry success branches per enemy:
- **knight**: plays `knightBlock` (8 frames) → counter window → `knightRecover` (reverse play of frames)
- **disc** (`currentEnemy === 'ball'`): plays `discBlock` (16 frames with long hold on frame 8) → receding frames 11-15
- **ninja**: plays `ninjaBlocked` (18 sparse frames) → `ninjaRecover` (9 frames)

### Counter-attack handler (`onCounterAttack`)

Plays strike sound, flashes white, applies `DAMAGE_ENEMY`, plays recovery animation, schedules next exchange OR checks defeat.

**CURRENT STATE: Counter-attack is fully forgiving.** Any tap or swipe during STUNNED state calls `onCounterAttack` regardless of `requiredCounter`. The `onCounterWrong` function still exists but is unreachable from current code.

---

## 6. CRITICAL SAFARI/MOBILE QUIRKS (DO NOT BREAK THESE)

### 1. SVG `r` attribute does NOT animate via CSS keyframes in Safari
Use CSS `transform: scale()` instead. The counter-attack expanding ring (`.ghost-stab`) uses this — **don't switch it back to animating the `r` attribute**, you'll get a static tiny circle on iPhone.

```css
/* GOOD — works on Safari */
.ghost-stab { transform-origin: center; transform-box: fill-box; }
@keyframes stab-ripple {
  0%   { transform: scale(0.15); }
  80%  { transform: scale(18); }
}
```

### 2. `preventDefault()` on touchstart cancels synthesized click events
This means `<button onclick="...">` silently fails on iOS if `preventDefault` is called before the click fires. Fix: skip `preventDefault` when target is a button/link/input. Also wire buttons with explicit `touchend` handlers as backup.

### 3. `cloneNode()` on Audio elements loses decoded buffer on mobile
Early versions cloned audio for rapid-fire plays — this caused missing sounds on iOS. Current code uses a **pool** of pre-loaded Audio elements (4 copies of each sound file). Don't switch back to cloneNode.

### 4. Tone.js requires user gesture before audio unlocks
Call `Tone.start()` inside a tap/click handler. Current code does this on the ENGAGE tap. Don't try to autoplay Tone.js music before user interaction.

### 5. CSS `touch-action: none` on `#game` is required
Without it, iOS interprets vertical swipes as page scroll / pull-to-refresh. Don't remove this.

---

## 7. AUDIO SYSTEM

Two subsystems, can be independently muted via the ♪ button top-right.

### SFX (MP3s via HTMLAudioElement)
- Pool of 4 copies per file to handle rapid-fire plays
- `Audio.playSwing()` — called on every attack telegraph start
- `Audio.playStrike()` — called on: successful parry, successful counter, player taking damage

### Music (Tone.js)
- Four "moods" defined in `MOODS` object: `title`, `archive`, `sector7`, `nexus`
- Each mood has: bpm, keyRoot, scale, bassPattern, leadPattern, kickPattern, snarePattern, hatPattern, stabPattern, filterBase, distortionWet
- `Audio.setMood(MOODS.x)` — transitions to new mood
- `Audio.setIntensity(0-1)` — ramps up music with enemy HP loss
- `nexus` mood has heavy distortion (`distortionWet: 0.55`) for industrial NIN/RATM feel

**Don't edit the MOODS object casually** — the patterns are hand-tuned musical phrases. Small changes to bassPattern or kickPattern can make a mood sound terrible.

---

## 8. KNOWN BUGS & QUIRKS (DO NOT "FIX" THESE — they're intentional)

1. **`currentEnemy === 'ball'` for the disc fight**: legacy naming. The disc replaced the ball mid-development. Renaming this requires touching ~15 code sites and isn't worth the risk.

2. **`ninjaBlocked` array is sparse**: array index and filename number don't match (index 1 = filename 008). The array construction accounts for this. If you add new schedule entries, use array INDICES 0-17, not filename numbers.

3. **`ninjaDownslash` is gone from ASSETS but files may still exist in the repo**: harmless. Don't re-add to ASSETS unless you're reintroducing the aerial attack.

4. **Knight `block` files use `block_1.png`-`block_8.png`** (no zero padding, 1-indexed). All other sprite sets use 3-digit zero-padded 0-indexed naming.

5. **`disc_rotate_frame_*.png` starts at 002**: files 000 and 001 don't exist. The ASSETS array compensates with `i+2`.

6. **The victory sequence has a hardcoded delay** between fights — don't shorten without testing the reveal animation.

---

## 9. RECOMMENDED EDIT WORKFLOW

### When modifying gameplay:
1. **Never touch the `playFramesAdvanced` function**. Its cancellation logic is subtle.
2. **Always pass `token`** through async exchange functions. Check `if (token !== currentExchangeToken) throw new Error('aborted')` after every `await`.
3. **Use `state === State.TELEGRAPH` to gate parry logic**, not time-based checks.
4. **Reset `comboTotal`, `comboIndex`, `comboPattern`** whenever you abort an exchange.

### When modifying CSS:
1. **Don't remove `touch-action: none`** from `#game`
2. **Don't animate SVG geometry attributes** (r, cx, cy) — use transforms
3. **Don't remove `pointer-events: none`** from overlays unless you mean to

### When modifying audio:
1. **Don't add `cloneNode` to Audio element playback**
2. **Don't try to init Tone.js before user gesture**
3. **Test music transitions** — the crossfade between moods is finicky

### When adding sprite sheets:
1. **Match the exact filename format** used in project (3-digit zero-padded, case-sensitive)
2. **Verify files exist in the project folder AND in GitHub repo** before referencing
3. **Add to `preloadAll`** via the ASSETS object — it handles error tracking
4. **Use the 404 diagnostic banner** (it auto-appears at bottom of game if files fail to load)

---

## 10. FILE LIST (as of v0.22)

### Code files:
- `alignment.html` — the entire game

### Currently used sprite files:
- `frame_000.png` through `frame_049.png` (50 files — knight attack)
- `block_1.png` through `block_8.png` (8 files — knight parried)
- `recover_frame_000.png` through `003.png` (4 files — knight recover)
- `disc_rotate_frame_002.png` through `008.png` (7 files — disc idle spin)
- `Disc_block_frame_000.png` through `015.png` (16 files — disc parried, capital D)
- `ninja_leap_frame_000.png` through `024.png` (25 files — ninja leap, still used for reveal)
- `Ninja_side_attack_frame_000.png` through `024.png` (25 files — ninja attack, capital N)
- `Ninja_blocked_frame_000.png` + `008.png` through `024.png` (18 files — ninja parried, capital N)
- `ninja_recover_frame_000.png` through `008.png` (9 files — ninja recover)
- `background_1.png` (archive bg)
- `cyber_hallway_9x16_no_text.png` (sector 7 bg)
- `power_nexus_9x16_no_text.png` (power nexus bg)

### Currently used audio files:
- `78678__joe93barlow__swing0.mp3` through `swing7.mp3` (8 files)
- `78675__joe93barlow__strike0.mp3` through `strike2.mp3` (3 files)

### Files in project that are NO LONGER used:
- `ninja_downslash_frame_*.png` — aerial attack cut
- `attack_impact_frame_*.png` — replaced by disc
- `block_return_frame_*.png` — replaced by disc

Safe to delete from GitHub repo but not urgent.

---

## 11. DEPLOYMENT TO GITHUB PAGES

1. Upload new `alignment.html` (drag-drop in GitHub web UI to replace existing)
2. Ensure all sprite files referenced in ASSETS exist in repo root
3. Settings → Pages → "Deploy from branch" → `main` / `/` → Save
4. Wait 30-60s for Actions to complete
5. Visit `https://brabbittdc.github.io/Alignment-game/alignment.html`

Hard-refresh on phone (close tab entirely, reopen) after updates — Safari caches HTML aggressively.

---

## 12. COMMON TASKS CHEAT SHEET

### Make ninja attack faster / slower
Edit the `duration` values in `thrust1Windup` and `thrust2Windup` arrays inside `ninjaGroundExchange` and `ninjaComboContinue`.

### Change parry tolerance
Edit the `55` in `isNinjaParry` and `isCounterDiagonal` — larger = more forgiving.

### Change minimum swipe distance
Edit the `dist < 25` check in `isNinjaParry` (for ninja only) or `swipeThreshold = 12` in `onPointerUp` (global).

### Change music character
Edit the pattern arrays in the MOODS object. Bass pattern is the most impactful; a 16-step array where 1 = note-on, 0 = silent.

### Add a new enemy
Follow the pattern: add reveal function, add exchange function, add state machine entry (TRANSITION → new fight), add background, add assets. Reference how `startNinjaFight` works.

### Add a new audio SFX
Add file URL to `SFX_FILES.swings` or `SFX_FILES.strikes` array, pool will auto-build. Or create a new pool by following the pattern in `Audio.init()`.

---

## 13. FINAL NOTE FOR AI ASSISTANTS

This codebase has been iteratively built with claude.ai across dozens of conversations. The user (brabbittDC on GitHub) is a product-minded non-programmer who tests on an iPhone via GitHub Pages. They cannot install dev tools locally.

**Golden rules:**
- Always make minimal, surgical edits
- Always verify filenames match the actual files in the project
- Always test syntax with `node --check` before delivering
- Always deploy to `/mnt/user-data/outputs/alignment.html`
- Always tell the user what changed in plain English
- Never reintroduce removed features without explicit request
- Never "refactor for cleanliness" unless asked — aesthetic changes cause regressions

The user understands game design well but not code. When they describe a bug, believe them; don't argue that "the code should work." Debug systematically: check asset loading first, then state transitions, then input handling, then animation timing.
