# IOS-NAVIGATION-BAR
prompt
 first one 
 ROLE
You are a senior UI engineer. Implement a "liquid glass" bottom navigation bar in my project. It must feel like a physical drop of glass that I can touch, not a sliding highlight.

PROJECT CONTEXT (fill in)
- Platform / stack: [Website (vanilla / React / Next / Vue) | Android (Jetpack Compose / XML) | Flutter | React Native | SwiftUI]
- App topic and screens: [e.g. music, shop, notes, 4 screens]
- Tabs (3 to 5): [Label + icon each]
- Brand accent color: [e.g. #e5435f]
- Theme: [light / dark / follows system], pastel glassmorphism background

VISUAL SPEC
1. The bar is a floating capsule, fully rounded, inset 16px from the sides and about 22px above the bottom (plus safe-area inset). It is about 76px tall with 6px inner padding. Tabs are an equal-width grid. Each tab has an icon (26px) above a 12.5px semi-bold label.
2. The bar itself is frosted glass and very transparent: about 20% white fill, a 14px backdrop blur, saturation 1.7, and a 1px white border at about 85% opacity. It has a soft inset top highlight and a soft pink-tinted drop shadow. Page content must stay visible through it.
3. Idle tab color is a dark neutral (#5d4a54). The active tab is the accent color. Only the tab under the bubble shows the accent color.
4. The bubble ("lens") is a clear glass capsule, about 1.04x one tab wide. It is slightly taller than the bar's inner area, so it overflows the bar by about 9px above and below. It has almost no fill (about 2% white) and these layers:
   - Refraction: it bends the real pixels behind it, meaning the bar, the icons and the page content. This is the key effect.
   - Chromatic aberration: the R, G and B channels refract at 100%, 90% and 80% strength, which gives a faint rainbow rim.
   - A 1.6px specular rim: a gradient border, white at the top-left fading to transparent in the middle, then pink at the bottom-right.
   - A soft inner glow with low opacity (about 0.16), plus a faint pink inner shadow at the bottom edge.
   - A separate drop shadow on its own sibling layer behind it. Do not put the outer shadow on the same element that has the backdrop filter, because it breaks the filter region on Chromium.
   - A mild post filter: saturate(1.4) brightness(1.02).
   The bubble must never look like an opaque white pill.

REFRACTION IMPLEMENTATION BY PLATFORM
- Web (Chromium): use backdrop-filter: url(#lg) saturate(1.4) brightness(1.02) on the lens.
  - Define an SVG filter with filterUnits and primitiveUnits = objectBoundingBox, and region x=0 y=0 width=1 height=1.
  - Use color-interpolation-filters="sRGB".
  - Add an feImage that holds the displacement map, with preserveAspectRatio="none" so it stretches to the bubble.
  - Add three feDisplacementMap passes with in="SourceGraphic", xChannelSelector="R" and yChannelSelector="G".
  - After each pass, isolate one channel with feColorMatrix. Then combine the three with feBlend mode="screen".
  - Set the displacement scale per frame to bend/width. The base is bend = height*(0.5 + (pressScale-1)*1.5). Multiply by 1.0, 0.9 and 0.8 for R, G and B.
  - Generate the map once in JS on a 256x128 canvas, as a capsule signed-distance field:
    - Radius r = 64.
    - bezel = 0.5*r.
    - For each pixel, depth = r - distanceToCapsuleAxis.
    - Magnitude is m = (1 - depth/bezel)^1.7 for depth < bezel, otherwise 0.
    - Encode R = 128 - nx*m*127 and G = 128 - ny*m*127, where (nx,ny) is the outward normal.
    - Set B = 128 and A = 255. 128 means neutral, so the center stays undistorted and the rim bends inward.
  - Make the bar background a separate sibling element behind the lens. Nesting the lens inside an element that already has its own backdrop-filter makes it sample only that element.
  - Fallback for Safari and Firefox, which do not support SVG filters in backdrop-filter: on those browsers use backdrop-filter: blur(5px) saturate(1.6) brightness(1.1). Keep the same shape and motion.
- Android Compose (API 33+): use RenderEffect.createRuntimeShaderEffect with an AGSL shader.
  - The shader samples the layer behind the bubble.
  - It offsets the sample inward near the capsule rim, using the same SDF profile and exponent 1.7.
  - It samples R, G and B with slightly different offsets.
  - Below API 33, fall back to Modifier.blur plus a translucent white capsule with a rim stroke.
- Flutter: use ImageFilter / BackdropFilter with a FragmentShader (GLSL) that implements the same SDF refraction. Fall back to a blurred BackdropFilter where shaders are unavailable.
- SwiftUI (iOS 26+): use native .glassEffect(.regular.interactive()) inside a GlassEffectContainer. On older iOS, use ultraThinMaterial with a rim stroke. The physics below still applies.
- React Native: use @react-native-community/blur or a Skia shader (@shopify/react-native-skia) for refraction, and Reanimated for the springs.

PHYSICS (the most important part)
Do not use fixed-duration or eased transitions for the bubble. Run a real spring simulation driven by requestAnimationFrame, Choreographer, withSpring, or equivalent. It must support per-frame interruption and retargeting. Sub-step the integration 4 times per frame, and clamp dt to 1/30 s.
Spring step: a = -k*(x - target) - c*v; v += a*dt; x += v*dt.

The bubble is not one rigid box. Simulate these separately:
1. Left edge L and right edge R, each with its own spring. Targets are centerX ± (bubbleWidth/2 * pressScale).
   - While moving, the leading edge (the side in the direction of travel) is stiff and the trailing edge is soft.
   - After release or tap, use leading k=330, c=17 and trailing k=105, c=8.5. The trailing edge lags and snaps back, so the blob stretches across several tabs and then wobbles into place.
   - While dragging, use leading k=520, c=30 and trailing k=230, c=17.
   - At rest, use k=220, c=12 for both.
2. Height H, with its own spring (k=230, c=10). The target is restHeight*pressScale*(1 - clamp(stretch, -0.5, 1.2)*0.30), where stretch = width/(baseWidth*pressScale) - 1. A long blob gets thinner and a squeezed one gets taller, so it jiggles when it lands.
3. Press scale S, with k=300 and c=11. On touch-down the target is 1.2, and on release it is 1.0. A tap also adds a velocity kick of +3.2, and the end of a drag adds +1.2.
4. The whole bar squishes toward the finger while pressed: scaleX = 1 + press*0.03, scaleY = 1 - press*0.012. The transform origin is the bubble's current center. Add up to +0.012 more scaleX based on bubble speed.

Interaction:
- Pointer down: swell the bubble, buzz the haptic (navigator.vibrate(10) or the platform haptic), and start following the finger.
- Dragging: the bubble center follows the finger through the springs, so it lags slightly. Past either end, apply rubber-banding: overshoot * 0.28.
- The active tab and screen switch live as the bubble passes over each tab, with a light haptic tick (6ms).
- Release: pick the destination tab from the bubble position plus velocity * 0.12 (a flick carries it further). Springs carry it there.
- Tap on a tab: the bubble swells, then the springs carry it across.
- Keep the animation loop running only while anything is still moving, and stop it when everything has settled.
- Respect prefers-reduced-motion and the platform's reduce-motion setting. Reduce stiffness and wobble, but keep it functional.
- Accessibility: use real buttons or tab roles, aria-current/selected state, focus-visible rings, 44px minimum touch targets, and keyboard arrow-key navigation. The text contrast on the bar must meet WCAG AA.

CONTENT AND RESPONSIVENESS
- Page content scrolls beneath the bar. Add bottom padding to scroll areas (about 130px) so nothing is trapped behind it.
- The bar must resize cleanly: recompute tab width on resize or rotation, and snap the springs to the new target without animating.
- On wide desktop viewports, either constrain the bar to a centered max width (about 480px) or keep it at the bottom of the phone-sized container.

DELIVERABLES
1. The complete working code, structured for my stack as one reusable NavBar / GlassTabBar component with props: items, activeIndex, onChange, accentColor.
2. A short list of the tunable constants in one place: spring stiffness and damping, bend strength, bezel width, swell amount, and chroma split.
3. Keep the rest of my UI and topic untouched. Only add the navigation bar, and wire it to my existing screens.
4. Test it. Verify the bubble overshoots and wobbles after a tap, stretches during a multi-tab jump, magnifies and bends the content under it, and switches screens correctly. Confirm there are no console errors.
5. Tell me about any platform limits (for example, which browsers or API levels get true refraction and which get the fallback).



second one 


# MASTER PROMPT: "Liquid Glass" physical navigation bar (any platform, any app topic)

## 0. How to use this
Paste everything from "ROLE" to the end of section 14 into your AI coding tool. Fill in the `[...]` fields in section 2. The appendix is a working web reference implementation. If your stack is the web, port it. For any other stack, reproduce its behavior and numbers exactly using the platform mapping in section 10.

---

## ROLE
You are a senior UI/graphics engineer. Add a **liquid-glass bottom navigation bar** to my project. The bar is a floating, frosted capsule. Inside it is a **clear glass bubble** that bends the pixels behind it. The bubble behaves like a drop of liquid under a spring simulation: it swells when pressed, stretches while travelling, overshoots, wobbles and settles. Do not substitute a CSS-transition highlight, a plain blurred pill or an opaque white indicator. If you cannot reach the full effect on a platform, implement the documented fallback and tell me.

## 1. Perceived-quality goals (what "correct" feels like)
1. **Transparent.** The bubble has almost no fill. I can see icons, text and page colours through it, and they are visibly bent near its rim.
2. **Physical.** Tapping pokes the bubble. Dragging drags it with lag. Releasing flings it. It never moves on a fixed-duration curve.
3. **Liquid.** The two ends of the bubble move independently. The leading end pulls ahead and the trailing end lags, so the bubble stretches across tabs and thins out, then snaps back and wobbles.
4. **Alive under the finger.** Pressing swells the bubble. The whole bar squishes slightly toward the finger. Haptic ticks fire on press and on tab change.
5. **Quiet when idle.** When nothing is moving, no animation loop runs.

## 2. Project inputs (fill in)
- Platform / stack: `[Web: vanilla | React | Next.js | Vue | Svelte]  [Android: Jetpack Compose | XML Views]  [Flutter]  [React Native]  [SwiftUI]`
- Min OS / browser support: `[e.g. Android 8+, Chrome 110+, iOS 16+]`
- App topic and screens: `[...]`
- Tabs, 3 to 5 (label + icon each): `[...]`
- Accent color: `[e.g. #e5435f]`   Neutral icon color: `[default #5d4a54]`
- Theme: `[light | dark | follow system]`. If dark, scale the white tints in section 5 accordingly. Do not just invert.
- Background behind the bar: `[gradient / images / lists that scroll under it]`. The effect needs visible content behind the bar.

## 3. Layer stack (bottom to top) and element contract
Everything is positioned inside one container, `NavBar`, fixed to the bottom of the screen. Keep the layers as **siblings**, not nested. Section 11 explains why.

| z | Layer | Purpose |
|---|---|---|
| 0 | `bar-bg` | The frosted capsule: fill, backdrop blur, border, shadow. A separate element behind everything, never the parent of the bubble. |
| 1 | `lens-shadow` | The bubble's drop shadow only. It has the same geometry as the bubble and is updated every frame together with it. |
| 1 | `tab` buttons | Icon and label for each tab. Their DOM order comes after `lens-shadow`. |
| 2 | `lens` (bubble) | The clear glass: refraction, rim light, inner glow. It has `pointer-events:none`, and the container handles all input. |

The container is the pointer target for drag and tap. `touch-action:none` is required on the tab row so horizontal drags do not scroll or swipe-navigate.

## 4. Geometry (CSS px / dp; scale proportionally if the bar size changes)
- Bar: left/right inset **16**, bottom **22** plus the safe-area inset, outer height **76**, inner padding **6**, fully rounded (radius = height/2).
- Tab row = bar inner box: height **64** (76 − 2×6), `N` equal columns. Tab width `w = rowWidth / N`.
- Tab content: icon **26×26** above a **12.5px / weight 600** label, with a gap of **3**. Minimum touch target 44×44.
- Bubble rest width `bw = 1.04 × w`.
- Bubble rest height `restH = rowHeight + 18 = 82`. It is centered vertically on the row and overflows the bar's inner box by 9 above and below.
- Bubble center for tab `i`: `i·w + w/2`.
- Bubble shape is always a capsule (radius = its own height/2), even while stretched.
- Scroll content gets about **130** bottom padding so nothing is trapped behind the bar. An optional mini-player sits at bottom **112**.

## 5. Visual tokens (light theme, exact)
**bar-bg**
- fill `rgba(255,255,255,.20)`
- backdrop blur **14px**, saturate **1.7**
- border `1px rgba(255,255,255,.85)`
- shadows: `inset 0 1px 0 #fff`, `inset 0 -10px 20px rgba(255,255,255,.30)`, `0 18px 40px -12px rgba(190,100,140,.45)`

**Tab idle color** `#5d4a54`. **Tab active color** = accent, with a `.25s` color/fill transition. Only the tab at the bubble's current index is active, and it switches live as the bubble passes over tabs.

**Lens (bubble)**
- fill `rgba(255,255,255,.02)`, which is nearly invisible. This is intentional.
- backdrop filter: `[refraction, section 6] saturate(1.4) brightness(1.02)`
- inset shadows only: `inset 0 1.5px 1px rgba(255,255,255,.95)`, `inset 0 -2px 5px rgba(255,255,255,.55)`, `inset 0 0 14px rgba(255,255,255,.16)`, `inset 0 -10px 18px -8px rgba(255,140,185,.28)`
- specular rim: a **1.6px** ring (mask a gradient to the border area) with `linear-gradient(140deg, rgba(255,255,255,1), rgba(255,255,255,0) 32%, rgba(255,255,255,0) 62%, rgba(255,196,224,.95))`
- **no outer drop shadow on this element** (see section 11, pitfall A)

**Lens-shadow layer:** `0 16px 30px -10px rgba(140,60,110,.32), 0 2px 6px rgba(60,20,40,.08)`

## 6. Refraction: exact math
The refraction is a displacement map applied to whatever is painted behind the bubble.

**Map (generate once, 256×128 RGBA canvas / bitmap, fully opaque).** The shape is a capsule signed-distance field. Let `r = 64` (half the height) and `bezel = 0.5·r = 32`. For each pixel `(x, y)`:
```
cx    = clamp(x + 0.5, r, 256 - r)        // nearest point on the capsule axis, x
dx    = x + 0.5 - cx ;  dy = y + 0.5 - r
dist  = max(hypot(dx, dy), 1e-6)
depth = r - dist                          // 0 at the rim, grows inward
m     = depth < bezel ? (1 - max(depth,0)/bezel)^1.7 : 0
nx, ny = dx/dist, dy/dist                 // outward unit normal
R = 128 - nx*m*127 ;  G = 128 - ny*m*127 ;  B = 128 ;  A = 255
```
`128` is neutral (no shift). The interior (`m = 0`) is undistorted. The rim pushes samples **inward**. Never sample outward: outside the bubble there are no source pixels, so the rim would go transparent.

**Applying it.** The displacement is `shift = scale · (channel/255 − 0.5)`. For this map, the maximum shift is about `0.5·scale·m` in the direction `−n`.

**Per-frame strength** (in pixels, then converted):
```
bend = bubbleHeight * (0.5 + (pressScale - 1) * 1.5)     // rest: 0.5·H; fully pressed (S=1.2): 0.8·H
```
**Chromatic aberration:** three displacement passes. The red channel uses `1.0·bend`, green uses `0.9·bend`, and blue uses `0.8·bend`. Each pass keeps only its own channel. The three results are recombined with `screen` blending.

**Web-specific unit note.** The SVG filter uses `primitiveUnits="objectBoundingBox"`, so `scale` is a fraction of the bubble's box. The horizontal shift is `scale·width` and the vertical shift is `scale·height`. Set `scaleAttr = bend / bubbleWidth` per frame. Consequently the horizontal rim shift is about `0.5·bend` px and the vertical rim shift is about `0.5·bend·(height/width)` px. On other platforms, reproduce these pixel shifts explicitly (section 10).

Do not exceed `bend ≈ 0.8·H`. Higher values duplicate or mirror content (for example, a label visible twice).

## 7. Physics model (exact)
**Integrator.** For each spring object `o = {x, v}`:
```
a = -k·(o.x − target) − c·o.v
o.v += a·dt ;  o.x += o.v·dt          // semi-implicit Euler
```
Per frame: `dt = min(frameDelta, 1/30)`, and **4 sub-steps** of `dt/4`. Run it from `requestAnimationFrame` / Choreographer / a ticker, and make it fully interruptible (retarget at any time while keeping velocity).

**State:** `L` (left edge x), `R` (right edge x), `H` (height), `S` (press scale, target `St`, default 1). Initial: `L = w/2 − 0.52w`, `R = w/2 + 0.52w`, `H = restH`, `S = 1`.

**Targets each frame.**
- Center target `cT`:
  - not dragging: `idx·w + w/2`.
  - dragging: the finger x in row coordinates, **rubber-banded** past the ends. With `lo = w/2` and `hi = w·(N − 0.5)`: if `x < lo`, use `lo + (x − lo)·0.28`; if `x > hi`, use `hi + (x − hi)·0.28`.
- `half = (bw/2)·S`. Left target `Lt = cT − half`, right target `Rt = cT + half`.
- Height target: `Ht = restH · S · (1 − clamp(stretch, −0.5, 1.2) · 0.30)`, where `stretch = (R − L)/(bw·S) − 1`. A long blob thins and a squeezed one grows taller, so volume is roughly conserved and it jiggles when it lands.

**Spring constants (k = stiffness, c = damping).**

| Situation | Leading edge (k, c) | Trailing edge (k, c) |
|---|---|---|
| Moving after tap or release (`|cT − center| > 1.5`) | 330, 17 | 105, 8.5 |
| Moving while dragging | 520, 30 | 230, 17 |
| Stationary while dragging | 420, 26 (both) | 420, 26 (both) |
| At rest / settling | 220, 12 (both) | 220, 12 (both) |

"Leading" is the edge in the direction of travel: `R` if `cT − center ≥ 0`, otherwise `L`. Height spring: `k = 230, c = 10`. Press-scale spring: `k = 300, c = 11`.

**Impulses (velocity kicks added to `S.v`).** A tap adds **+3.2**. Releasing after a real drag adds **+1.2**. Releasing without a drag adds **+2.2**.

**Bar squish (the whole nav container).** `press = (S − 1)/0.2`. Apply `scaleX = 1 + press·0.03 + clamp(|R.v + L.v|/(w·60), 0, 0.012)` and `scaleY = 1 − press·0.012`. The transform origin is the bubble's current center x (plus the bar's left inset) and 50% of the height.

**Rest condition (stop the loop only when all hold):** not dragging, `|cT − center| ≤ 0.05`, `|L.v| + |R.v| ≤ 0.6`, `|H.v| ≤ 0.6`, `|S − St| ≤ 0.001`, `|S.v| ≤ 0.02`.

**Render each frame:**
- Bubble `left = L`, `width = R − L`, `height = H`, `top = rowHeight/2 − H/2`.
- Copy the same four values to the lens-shadow layer.
- Update the refraction strength (section 6).

## 8. Interaction state machine
- **pointer down** (anywhere on the tab row): `dragging = true`, `moved = false`, record the start x, capture the pointer, set `St = 1.2`, fire a 10 ms haptic, start the loop.
- **pointer move:** if `|dx| > 4px`, set `moved = true`. Update the finger x. If `moved`, set the active index to `clamp(floor(x / w), 0, N−1)` live. Changing the active tab fires a 6 ms haptic, changes the active color and switches the visible screen.
- **pointer up / cancel:** `dragging = false`, `St = 1`. If `moved`: `idx = clamp(floor((center + avgEdgeVelocity·0.12)/w), 0, N−1)`, so a fast flick carries further. If not moved: `idx = clamp(floor(x/w), 0, N−1)`. Add the impulse (section 7) and run the loop.
- **click on a tab button:** ignore it if a drag just ended. Otherwise set the active index and add a +3.2 impulse. Keyboard activation also goes through this path.
- **resize / rotation / insets change:** recompute `w`, snap `L`, `R`, `H` to rest for the current `idx` with zero velocity, and do not animate.
- Switching screens uses a short fade/translate (about 0.45 s). The bubble and the screen change are decoupled. The screen switches as soon as the index changes.

## 9. Frame loop (reference pseudo-code)
```
frame(now):
  dt = min((now-last)/1000, 1/30)
  compute cT, Lt, Rt, spring constants (section 7)
  repeat 4x: spring(R,Rt,..); spring(L,Lt,..); update H target from stretch; spring(H,Ht,230,10); spring(S,St,300,11)
  apply bubble geometry + lens-shadow geometry
  update refraction scale = (height*(0.5+(S-1)*1.5)) / width   // per channel x1.0, x0.9, x0.8
  apply bar squish transform
  if not at rest: schedule next frame
```

## 10. Platform mapping (implement the one for my stack)
- **Web (Chromium, the full effect):** the appendix. Use `backdrop-filter: url(#lg) saturate(1.4) brightness(1.02)` with an SVG filter. Feature-detect with a UA check for Chromium, because `CSS.supports` is unreliable here. **Fallback for Safari and Firefox:** `backdrop-filter: blur(5px) saturate(1.6) brightness(1.1)`. All geometry and physics stay identical. React/Vue/Svelte: keep the springs in a ref or non-reactive store and write styles directly. Never run the physics through component state per frame.
- **Android, Jetpack Compose (API 33+, full effect):** render the content behind the bar into a `GraphicsLayer`, then apply `RenderEffect.createRuntimeShaderEffect` with an AGSL shader on the bubble layer. Use the same math. Per-axis pixel offsets are `off.x = −n.x·m·0.5·bend` and `off.y = −n.y·m·0.5·bend·(H/W)`:
  ```
  uniform shader content; uniform float2 size; uniform float bend;
  half4 main(float2 p){
    float r = size.y*0.5;
    float2 d = p - float2(clamp(p.x, r, size.x - r), r);
    float dist = max(length(d), 1e-4);
    float depth = r - dist, bz = r*0.5;
    float m = depth < bz ? pow(1.0 - clamp(depth,0.0,bz)/bz, 1.7) : 0.0;
    float2 n = d/dist;
    float2 off = float2(-n.x*m*0.5*bend, -n.y*m*0.5*bend*(size.y/size.x));
    half4 c; c.r = content.eval(p+off*1.0).r; c.g = content.eval(p+off*0.9).g; c.b = content.eval(p+off*0.8).b; c.a = 1.0;
    return c;
  }
  ```
  Use `Animatable` / `spring()` or a manual integrator for L, R, H and S. A manual integrator is required to reproduce the asymmetric edge stiffness exactly. **Below API 33:** blur the bar backdrop, draw a translucent capsule plus the rim stroke, keep the physics.
- **Android XML Views:** same idea with a `RuntimeShader` on API 33+, and `Choreographer.FrameCallback` for the loop. Below 33 use a blur plus a gradient rim drawable.
- **Flutter:** `BackdropFilter` with `ImageFilter.shader` (FragmentProgram, GLSL port of the AGSL above) on Impeller. Use a `Ticker` for the springs. Fallback: `BackdropFilter(ImageFilter.blur)` plus the rim `CustomPainter`.
- **React Native:** `@shopify/react-native-skia` runtime shader (same SKSL math), with Reanimated worklets for the springs on the UI thread. Fallback: a blur view plus a rim.
- **SwiftUI (iOS 26+):** prefer the native `.glassEffect(.regular.interactive())` inside a `GlassEffectContainer`, and still implement the two-edge asymmetric spring if you want my exact liquid stretch. Below iOS 26: `ultraThinMaterial` capsule plus the rim stroke and `TimelineView`-driven springs.

## 11. Known pitfalls (already hit, avoid them)
A. **Outer `box-shadow` on the element with the SVG backdrop-filter breaks the filter region in Chromium.** The filtered area shifts or is clipped. Keep only inset shadows on the lens and put the drop shadow on the separate `lens-shadow` sibling.
B. **A parent with its own `backdrop-filter` becomes a backdrop root.** A nested bubble then only sees that parent's contents, not the page. Keep `bar-bg` as a sibling behind the bubble, not its ancestor.
C. Use `filterUnits` and `primitiveUnits = objectBoundingBox` with `feImage` at `0 0 1 1` and `preserveAspectRatio="none"`. Pixel-unit regions drifted out of alignment when the bubble was moved by transform and resized per frame.
D. Too-large `bend` mirrors or duplicates content. Keep it at or below `0.8·H`.
E. Sampling outward from the rim returns transparency. Always push inward.
F. Never drive the springs through framework state or layout passes. Write transforms and sizes directly, and avoid animating `top`/`height` through slow layout paths where a transform or layer property is available (the bubble's changing height is the one required exception).
G. `will-change` on the bubble and the bar is fine. Do not apply SVG-filter backdrops to many elements. Use one bubble only.

## 12. Accessibility, performance, edge cases
- Semantics: a `nav` with real `button`s (or the platform tab role), the active one marked with `aria-current="page"` / `selected`, a visible focus ring, and arrow-key / Home / End navigation.
- Contrast: label text on the bar must meet WCAG AA against the worst-case page behind it. Raise the `bar-bg` fill if it does not.
- Honour reduce-motion: lower stiffness and remove overshoot (use a critically damped `c ≈ 2√k`), disable the bar squish, and keep the haptics optional.
- Haptics only where the platform supports them. Guard every call.
- Performance budget: 60 fps on a mid-range phone. If it drops below that, reduce the `bend` filter cost first (shrink the map, drop to 2 channels), not the physics.
- RTL: mirror tab order and the drag direction. Keep the physics symmetric.
- Dark theme: rescale the white fills and highlights so the bubble stays clear, not milky.

## 13. Acceptance tests (verify and report each)
1. At rest the bubble sits centered on the active tab at 1.04× tab width and 82 high, and it is visibly more transparent than the bar.
2. A tap on a tab 2+ steps away shows the bubble stretching wider while thinning, overshooting the target and settling in about 0.5–0.8 s.
3. Content under the bubble (text, icons, gradients) is visibly bent near its rim, with a faint colour fringe, and undistorted in the center.
4. Press-and-hold swells the bubble and slightly squishes the bar. Dragging follows the finger with lag. Pulling past either end rubber-bands. A fast flick lands one tab beyond the release point.
5. The active color follows the bubble live while dragging.
6. When everything has settled, the animation loop is idle.
7. No console errors or shader-compile errors. Rotating the device or resizing the window re-snaps the bubble with no animation glitch.
8. Fallback path works: force the non-refraction branch and confirm that the shape and physics are unchanged.

## 14. Deliverables
1. One reusable component: `GlassTabBar(items, activeIndex, onChange, accent)`, wired to my existing screens. Do not redesign my other UI.
2. A single constants block with every tunable from sections 4–7 (sizes, spring k/c, bend factor, bezel, chroma factors, swell, impulses, rubber-band factor).
3. The verification results for section 13, and the list of platform limits and fallbacks actually in effect.
4. A short note on how to retune the feel. Higher `k` makes it snappier and lower `c` makes it bouncier. A smaller trailing `k` gives a longer liquid tail.

---

# APPENDIX: working web reference (verified in Chromium)
Port the numbers exactly. The tab icons are omitted for brevity. Use any 26px icons.

### A1. CSS (nav only; `--accent` = your accent color)
```css
/* ========= liquid glass tab bar ========= */
.tabbar{will-change:transform;position:absolute;left:16px;right:16px;bottom:22px;height:76px;z-index:25;padding:6px}
.bar-bg{position:absolute;inset:0;border-radius:999px;
  background:rgba(255,255,255,.2);backdrop-filter:blur(14px) saturate(1.7);-webkit-backdrop-filter:blur(14px) saturate(1.7);
  border:1px solid rgba(255,255,255,.85);
  box-shadow:inset 0 1px 0 #fff,inset 0 -10px 20px rgba(255,255,255,.3),0 18px 40px -12px rgba(190,100,140,.45)}
.tabs{position:relative;display:grid;grid-template-columns:repeat(4,1fr);height:100%;touch-action:none}
.tabs{overflow:visible}
/* clear glass bubble: refracts the real pixels behind it (SVG displacement map) */
.lens{position:absolute;top:0;height:100%;left:0;width:25%;z-index:2;border-radius:999px;pointer-events:none;
  background:rgba(255,255,255,.02);
  -webkit-backdrop-filter:url(#lg) saturate(1.4) brightness(1.02);backdrop-filter:url(#lg) saturate(1.4) brightness(1.02);
  box-shadow:inset 0 1.5px 1px rgba(255,255,255,.95),inset 0 -2px 5px rgba(255,255,255,.55),inset 0 0 14px rgba(255,255,255,.16),inset 0 -10px 18px -8px rgba(255,140,185,.28);
  will-change:transform,width,height}
.lens::after{content:"";position:absolute;inset:0;border-radius:inherit;padding:1.6px;
  background:linear-gradient(140deg,rgba(255,255,255,1),rgba(255,255,255,0) 32%,rgba(255,255,255,0) 62%,rgba(255,196,224,.95));
  -webkit-mask:linear-gradient(#000 0 0) content-box,linear-gradient(#000 0 0);-webkit-mask-composite:xor;mask-composite:exclude}
.lens-shadow{position:absolute;top:0;height:100%;left:0;width:25%;z-index:1;border-radius:999px;pointer-events:none;box-shadow:0 16px 30px -10px rgba(140,60,110,.32),0 2px 6px rgba(60,20,40,.08);will-change:transform}
.no-svg-refract .lens{-webkit-backdrop-filter:blur(5px) saturate(1.6) brightness(1.1);backdrop-filter:blur(5px) saturate(1.6) brightness(1.1)}
.tab{position:relative;z-index:1;display:flex;flex-direction:column;align-items:center;justify-content:center;gap:3px;border:0;background:none;cursor:pointer;
  font:600 12.5px/1 inherit;font-family:inherit;color:#5d4a54;transition:color .25s}
.tab svg{width:26px;height:26px;fill:#5d4a54;transition:fill .25s}
.tab.on{color:var(--accent)}.tab.on svg{fill:var(--accent)}
```

### A2. SVG refraction filter (place once in the page body)
```html
<svg width="0" height="0" style="position:absolute" aria-hidden="true">
  <filter id="lg" filterUnits="objectBoundingBox" primitiveUnits="objectBoundingBox" x="0" y="0" width="1" height="1" color-interpolation-filters="sRGB">
    <feImage id="lgMap" x="0" y="0" width="1" height="1" preserveAspectRatio="none" result="map"/>
    <feDisplacementMap id="dR" in="SourceGraphic" in2="map" scale="40" xChannelSelector="R" yChannelSelector="G" result="r0"/>
    <feColorMatrix in="r0" type="matrix" values="1 0 0 0 0  0 0 0 0 0  0 0 0 0 0  0 0 0 1 0" result="r"/>
    <feDisplacementMap id="dG" in="SourceGraphic" in2="map" scale="36" xChannelSelector="R" yChannelSelector="G" result="g0"/>
    <feColorMatrix in="g0" type="matrix" values="0 0 0 0 0  0 1 0 0 0  0 0 0 0 0  0 0 0 1 0" result="g"/>
    <feDisplacementMap id="dB" in="SourceGraphic" in2="map" scale="32" xChannelSelector="R" yChannelSelector="G" result="b0"/>
    <feColorMatrix in="b0" type="matrix" values="0 0 0 0 0  0 0 0 0 0  0 0 1 0 0  0 0 0 1 0" result="b"/>
    <feBlend in="r" in2="g" mode="screen" result="rg"/>
    <feBlend in="rg" in2="b" mode="screen"/>
  </filter>
</svg>
```

### A3. Markup
```html
<nav class="tabbar" aria-label="Main"><div class="bar-bg"></div>
    <div class="tabs" id="tabs">
      <div class="lens-shadow" id="lensSh"></div><div class="lens" id="lens"></div>
      <button class="tab on" data-i="0"><svg viewBox="0 0 24 24"><!-- 26px icon, fill:currentColor-style --></svg>Home</button>
      <button class="tab" data-i="1"><svg viewBox="0 0 24 24"><!-- 26px icon, fill:currentColor-style --></svg>New</button>
      <button class="tab" data-i="2"><svg viewBox="0 0 24 24"><!-- 26px icon, fill:currentColor-style --></svg>Radio</button>
      <button class="tab" data-i="3"><svg viewBox="0 0 24 24"><!-- 26px icon, fill:currentColor-style --></svg>Library</button>
    </div>
  </nav>
```

### A4. Script (springs, drag, refraction map)
```html
<script>
(() => {
  const tabs = document.getElementById('tabs'), bar = document.querySelector('.tabbar'), lens = document.getElementById('lens'), lensSh = document.getElementById('lensSh');
  const btns = [...tabs.querySelectorAll('.tab')], screens = [...document.querySelectorAll('.screen')]; // screens: your app's pages, one per tab
  // refraction map: capsule signed-distance field -> inward push strongest at the rim (R=x, G=y, 128 = neutral)
  const isChromium = /Chrome|Chromium|Edg\//.test(navigator.userAgent) && !/Firefox/.test(navigator.userAgent);
  if(!isChromium) document.documentElement.classList.add('no-svg-refract');
  const fMap = document.getElementById('lgMap'), fil = document.getElementById('lg'), dR = document.getElementById('dR'), dG = document.getElementById('dG'), dB = document.getElementById('dB');
  (function buildMap(){
    const W = 256, Hh = 128, cv = document.createElement('canvas'); cv.width = W; cv.height = Hh;
    const g = cv.getContext('2d'), im = g.createImageData(W, Hh), r = Hh/2, bz = r*.5;
    for(let y=0;y<Hh;y++) for(let x=0;x<W;x++){
      const cx = Math.min(Math.max(x+.5, r), W-r), dx = x+.5-cx, dy = y+.5-r, dist = Math.hypot(dx,dy) || 1e-6;
      const depth = r - dist; let m = 0;
      if(depth < bz){ const t = Math.max(depth,0)/bz; m = Math.pow(1-t, 1.7); }
      const nx = dx/dist, ny = dy/dist, o = (y*W+x)*4;
      im.data[o]   = 128 - nx*m*127;   // push toward centre -> magnified, bent rim
      im.data[o+1] = 128 - ny*m*127;
      im.data[o+2] = 128; im.data[o+3] = 255;
    }
    g.putImageData(im,0,0);
    fMap.setAttribute('href', cv.toDataURL());
  })();

  // liquid body = two independent edges + a height spring + a press (swell) spring
  const L = {x:0,v:0}, R = {x:0,v:0}, H = {x:0,v:0}, S = {x:1,v:0,t:1};
  let idx = 0, dragging = false, moved = false, pointerX = 0, startX = 0, raf = 0, last = 0, tx = 0;
  const tw = () => tabs.clientWidth / 4;
  const spring = (o, t, k, c, dt) => { o.v += (-k*(o.x - t) - c*o.v)*dt; o.x += o.v*dt; };
  const clamp = (v,a,b) => Math.min(Math.max(v,a),b);
  const buzz = ms => { try{ navigator.vibrate && navigator.vibrate(ms); }catch(e){} };

  function frame(now){
    const dt = Math.min((now-last)/1000, 1/30) || 1/60; last = now;
    const w = tw(), h = tabs.clientHeight, bw = w*1.04, restH = h + 18;

    // where the blob wants its centre (finger is rubber-banded past the ends)
    let cT = idx*w + w/2;
    if(dragging){
      const lo = w/2, hi = w*3.5;
      cT = pointerX < lo ? lo + (pointerX-lo)*.28 : pointerX > hi ? hi + (pointerX-hi)*.28 : pointerX;
    }
    const half = bw/2 * S.x;
    const Lt = cT - half, Rt = cT + half;
    const cur = (L.x + R.x)/2, d = cT - cur, moving = Math.abs(d) > 1.5;
    // leading edge is stiff (pulled), trailing edge is soft (drags & snaps back) => liquid stretch
    let kl = 220, cl = 12, kt = 220, ct = 12;
    if(moving){
      if(dragging){ kl = 520; cl = 30; kt = 230; ct = 17; }
      else        { kl = 330; cl = 17; kt = 105; ct = 8.5; }
    } else if(dragging){ kl = kt = 420; cl = ct = 26; }
    const rightLeads = d >= 0;
    const sub = 4, h_ = dt/sub;
    for(let i=0;i<sub;i++){
      spring(R, Rt, rightLeads?kl:kt, rightLeads?cl:ct, h_);
      spring(L, Lt, rightLeads?kt:kl, rightLeads?ct:cl, h_);
      const stretch = (R.x - L.x)/(bw*S.x) - 1;                  // >0 elongated, <0 squeezed
      spring(H, restH*S.x*(1 - clamp(stretch,-.5,1.2)*.30), 230, 10, h_);   // volume-ish: long => thin, jiggles
      spring(S, S.t, 300, 11, h_);
    }
    const width = R.x - L.x, height = H.x, top = h/2 - height/2;
    lens.style.transform = `translateX(${L.x}px)`;
    lens.style.width = width + 'px';
    lens.style.top = top + 'px';
    lens.style.height = height + 'px';
    lensSh.style.transform = lens.style.transform; lensSh.style.width = lens.style.width; lensSh.style.top = lens.style.top; lensSh.style.height = lens.style.height;
    // keep the refraction map glued to the bubble's live size; pressing deepens the bend
    const bend = height*(.5 + (S.x-1)*1.5);
    const k = bend/width;
    dR.setAttribute('scale', k); dG.setAttribute('scale', k*.9); dB.setAttribute('scale', k*.8);
    // the glass bar itself leans & squishes toward the finger
    const press = (S.x - 1)/.2;
    bar.style.transformOrigin = `${(L.x+R.x)/2 + 16}px 50%`;
    bar.style.transform = `scale(${1 + press*.03 + clamp(Math.abs(R.v+L.v)/(w*60),0,.012)}, ${1 - press*.012})`;

    const alive = dragging || Math.abs(d) > .05 || Math.abs(L.v)+Math.abs(R.v) > .6 || Math.abs(H.v) > .6 || Math.abs(S.x-S.t) > .001 || Math.abs(S.v) > .02;
    if(alive) raf = requestAnimationFrame(frame); else raf = 0;
  }
  const kick = () => { if(!raf){ last = performance.now(); raf = requestAnimationFrame(frame); } };

  function setIdx(i){
    if(i === idx) return; idx = i; buzz(6);
    btns.forEach((b,k) => b.classList.toggle('on', k === i));
    screens.forEach((s,k) => s.classList.toggle('on', k === i));
  }
  // tap = a squishy poke: swell, then spring across
  btns.forEach((b,i) => b.addEventListener('click', () => { if(moved) return; setIdx(i); S.v += 3.2; kick(); }));

  tabs.addEventListener('pointerdown', e => {
    dragging = true; moved = false; startX = e.clientX; tabs.setPointerCapture(e.pointerId);
    pointerX = e.clientX - tabs.getBoundingClientRect().left;
    S.t = 1.2; buzz(10); kick();
  });
  tabs.addEventListener('pointermove', e => {
    if(!dragging) return;
    if(Math.abs(e.clientX - startX) > 4) moved = true;
    pointerX = e.clientX - tabs.getBoundingClientRect().left;
    if(moved) setIdx(clamp(Math.floor(pointerX / tw()), 0, 3));
    kick();
  });
  const end = () => {
    if(!dragging) return; dragging = false; S.t = 1;
    const w = tw(), vel = (L.v + R.v)/2, cur = (L.x + R.x)/2;
    let i = moved ? clamp(Math.floor((cur + vel*.12)/w), 0, 3) : clamp(Math.floor(pointerX/w), 0, 3);  // flick carries it further
    setIdx(i); S.v += moved ? 1.2 : 2.2; kick();
    setTimeout(() => moved = false, 0);
  };
  tabs.addEventListener('pointerup', end); tabs.addEventListener('pointercancel', end);

  addEventListener('resize', () => { const w = tw(); L.x = idx*w + w/2 - w*.52; R.x = idx*w + w/2 + w*.52; L.v = R.v = 0; kick(); });
  { const w = tw(); L.x = w/2 - w*.52; R.x = w/2 + w*.52; H.x = tabs.clientHeight + 18; }
  kick();
})();
</script>
```
