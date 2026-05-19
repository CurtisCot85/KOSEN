# Fulcrum — Claude Code Instructions

## What this project is

Fulcrum is a browser-based app that measures a user's body proportions via camera (MediaPipe Pose Landmarker), computes their personalized ideal squat mechanics using a physics-based kinematic solver (CoG balance over mid-foot), records their actual squat, and compares actual vs ideal with coaching cues.

The entire app is a single HTML file (`index.html`). No framework, no build step, no server. One file, opened in a browser.

## How to write code

Follow this style guide exactly. Every piece of code must match these conventions. Do not fall back on standard Python (PEP 8) or JavaScript conventions — this style guide overrides them.

### Indentation

Two spaces. Not four. Not tabs.

### Spacing inside parentheses

Spaces inside every set of parentheses, around every argument. Function calls, definitions, conditionals, and list/dict indexing.

```javascript
// yes
let model = solve_at_depth( l_foot , l_shank , l_thigh , l_torso , body_mass , bar_mass , sex );
let result = compute_ratio( segment_a , segment_b );
if ( ratio > 1.0 ) {

// no
let model = solve_at_depth(l_foot, l_shank, l_thigh, l_torso, body_mass, bar_mass, sex);
if (ratio > 1.0) {
```

### Spaces around operators

Spaces around `=`, `+`, `-`, `*`, `/`, `==`, `>`, `<`, and all other operators. Spaces around `=` in keyword arguments.

```javascript
// yes
let ankle_x = -( midfoot_from_heel - ankle_from_heel );

// no
let ankle_x = -(midfoot_from_heel - ankle_from_heel);
```

### Naming conventions

snake_case for everything — variables, functions, files, parameters. No camelCase, no PascalCase. This applies to JavaScript even though JS convention is camelCase.

```javascript
// yes
let torso_angle = compute_torso_angle( segment_ratios , depth_deg );

// no
let torsoAngle = computeTorsoAngle( segmentRatios , depthDeg );
```

### Capitalization

No capitalization in variable names, function names, or comments. The only exceptions are proper nouns (MediaPipe, Fulcrum), acronyms (CoG, FPS, HTML), and cases where a framework requires it.

```javascript
// yes
let cog_margin_mm = 40.0;                                   // acceptable CoG deviation

// no
let COG_Margin_MM = 40.0;
// Compute the torso angle
```

Exception: true constants at the top of a file may use ALL_CAPS.

```javascript
let ANIM_FPS       = 30;
let COG_MARGIN_MM  = 40.0;
```

### No one-liners

Every operation gets its own line. No ternary expressions. No chained operations on a single line. No inline conditionals. No list comprehensions. No lambda functions. Always use explicit loops and if/else blocks.

```javascript
// yes
if ( frame.valid ) {
  torso_angle = frame.torso_angle_deg;
} else {
  torso_angle = 0.0;
}

// no
torso_angle = frame.valid ? frame.torso_angle_deg : 0.0;
```

### Comments — section headers

Every logical section of code must be preceded by a 1-2 paragraph comment block explaining what the section does and why. These explain purpose and context before the implementation.

```javascript
// compute the torso angle that keeps the system center of gravity directly
// over mid-foot. this is the core constraint of the model — if CoG drifts
// forward of mid-foot, the lifter falls on their face. if it drifts back,
// they sit down. the torso angle is the one free variable that satisfies
// this balance equation given all other joint positions.
```

### Comments — inline

Important individual lines get short comments explaining what they do. Align them to the right when there are multiple in a row.

```javascript
let ankle_from_heel   = 0.38 * l_foot;                      // ankle position along foot
let midfoot_from_heel = 0.50 * l_foot;                      // mid-foot reference point
let ankle_x = -( midfoot_from_heel - ankle_from_heel );     // negative = behind mid-foot
```

### Blank lines

Use blank lines generously between logical blocks. A blank line before and after each section-header comment. Code should have visible breathing room.

### Alignment

When multiple related assignments appear together, align them for visual clarity.

```javascript
let l_foot    = height_m * user_fracs.foot_frac;
let l_shank   = height_m * user_fracs.shank_frac;
let l_thigh   = height_m * user_fracs.thigh_frac;
let l_torso   = height_m * user_fracs.torso_frac;
```

## Architecture overview

### Screen flow

1. **start** — welcome screen with color selection (3 colorways)
2. **inputs** — sex, height, body weight
3. **calibrate** — front-facing camera, T-pose, MediaPipe Pose Landmarker (full model), multi-capture with screenshots, nose-to-heel height measurement, segment ratio extraction
4. **results** — animated stick figure of ideal squat, squat type selector (bodyweight/high/low bar), bar weight input, solver re-runs on change
5. **record** — rear camera, auto-detection state machine (waiting → ready → squatting → between_reps → captured), up to 4 reps, hands-free
6. **compare** — animated depth-synced overlay of ideal (blue) + best rep (green or red) + worst rep (red), ±5° margin band, mass-weighted CoG dots, coaching cues, video replay of best rep

### Key systems

- **Kinematic solver** (`solve_at_depth`) — computes joint positions for a given squat depth using CoG balance constraint. Inputs: segment lengths, body mass, bar mass, bar position, sex (for de Leva mass fractions), ankle dorsiflexion.
- **Calibration pipeline** — MediaPipe landmarks → EMA smoothing → confidence-weighted side selection → segment ratios + direct height fractions (nose-to-heel / 0.965) → multi-capture history.
- **Recording state machine** — auto-detects person, establishes baseline hip height, tracks descent/ascent, segments reps, scores each against ideal.
- **Comparison scoring** — deviation score per rep = |torso_diff| × 2 + |thigh_diff| + |shank_diff| × 0.5. Within margin = all three angles within ±5°. Display: all within → best green; all outside → best red; mixed → best green + worst red.
- **Plate occlusion handling** — camera is on the plate side. Shoulder/head always occluded. Side selection by hip/knee/ankle visibility only. Facing direction from knee position. Shoulder used at any visibility for torso angle (MediaPipe infers from visible back). Segmentation-based approach under development.
- **Video replay** — MediaRecorder captures canvas stream (video + skeleton composite) during recording. Replays best rep on comparison screen.

### CSS theme system

Three colorways via CSS custom properties set on `:root`. The `COLORWAYS` object maps variable names to values. `apply_colorway()` iterates and sets them.

### Functions exposed to window

All onclick handlers must be exported: `window.function_name = function_name`. The app runs inside a `<script type="module">` block, so nothing is global by default.

## What NOT to do

- Do not use camelCase anywhere
- Do not use ternary operators
- Do not use list comprehensions or arrow functions for logic (arrow functions in `.then()` callbacks are acceptable)
- Do not capitalize comments or variable names
- Do not add npm dependencies, build steps, or frameworks
- Do not split into multiple files — everything stays in index.html
- Do not use four-space indentation
- Do not omit spaces inside parentheses
- Do not create new features not requested — this is an implementation project, not a design project
