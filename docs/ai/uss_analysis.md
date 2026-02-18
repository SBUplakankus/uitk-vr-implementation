# UITK Visual Styling — Modern Dark Glass Aesthetic

> **Verified against**: Unity 6.3 LTS docs (fetched February 2026)  
> **Aesthetic target**: Vercel / Linear / Raycast — professional dark glass, clean hierarchy  
> **Scope**: USS styling, gradients, transitions, Shader Graph per-element materials  
> **Platform**: Quest 2 render texture → world space quad pipeline  
> **Note**: The default theme is intentionally restrained. The token system makes
> it fully customisable — glows, gradients, and colour pops are a theme swap away.

---

## What the Vercel / Linear Aesthetic Actually Is

Before techniques, it's worth being precise about what makes these interfaces look the way they do. It is not primarily about effects — it is about restraint and surface hierarchy.

**The defining characteristics:**

- Near-black base surfaces, not pure black. `#0a0a0f` to `#0d0d12` — a very slight cool blue-black rather than flat black. Pure black in UI reads as a void; this reads as a surface.
- Colour appears almost nowhere on the base layer. The surfaces, cards, and panels are essentially monochrome. Colour exists only in interactive elements, status indicators, and deliberate accent moments.
- Borders are the primary way surfaces are separated. Not shadows, not strong background colour contrast — a single pixel border at `rgba(255,255,255, 0.06–0.10)` is what defines a card edge.
- Typography carries the hierarchy. Large, slightly tracked titles. Muted body text. The weight contrast between heading and body is doing most of the work.
- Gradients are used extremely sparingly and precisely. One gradient highlight on a hero element, or a very subtle radial glow behind a primary CTA. Not on every card.
- Interactive feedback is immediate and physical. Hover states are instant or near-instant (80–100ms). Press states give a perceptible scale response. There is no ambiguity about whether you clicked something.

**The customisable layer sits on top of this.** The token system means a designer can push any element toward a more vibrant, glow-heavy, gradient-rich look by editing theme values. The default theme ships restrained; custom themes can be anything.

---

## The Honest Capability Map

What UITK supports vs what web developers take for granted in 2026. Every gap confirmed against Unity 6.3 docs.

| Feature | CSS | USS (Unity 6.3) | Workaround |
|---|---|---|---|
| `linear-gradient()` | ✅ | ❌ | Baked texture, SVG, or Shader Graph |
| `radial-gradient()` | ✅ | ❌ | Same |
| `box-shadow` | ✅ | ❌ | Border trick or Shader Graph glow |
| `backdrop-filter: blur()` | ✅ | ⚠️ Partial | `filter:` on a sibling element |
| `@keyframes` | ✅ | ❌ | C# `schedule` or DOTween |
| `transition` | ✅ | ✅ | — |
| Transform (translate/rotate/scale) | ✅ | ✅ layout-free | — |
| `filter: blur/grayscale/invert` | ✅ | ✅ | — |
| Custom shader per element | ❌ | ✅ `-unity-material` | — |
| CSS custom properties `var()` | ✅ | ✅ | — |
| `opacity` | ✅ | ✅ | — |
| `border-radius` | ✅ | ✅ | — |
| Spring physics animations | ✅ via libs | ❌ | DOTween spring ease |
| `mix-blend-mode` | ✅ | ❌ | Shader Graph blend node |

The critical gap is `linear-gradient()` — absent from USS since UI Toolkit launched. Everything that defines modern web surfaces relies on gradients. In UITK every gradient needs a workaround.

---

## Default Dark Theme — The Colour Language

These are the base token values for the default professional theme. Every value has a reason.

```css
/* base-variables.uss */
:root {
    /* ── Surfaces ──────────────────────────────────────────────────────── */
    /* Not pure black. The slight blue shift reads as a lit surface, not a void */
    --color-background:       rgb(10, 10, 15);
    --color-surface:          rgb(16, 16, 24);
    --color-surface-raised:   rgb(22, 22, 32);
    --color-surface-overlay:  rgb(28, 28, 40);

    /* ── Borders ──────────────────────────────────────────────────────── */
    /* Low opacity white. This is doing the work of defining edges */
    --color-border:           rgba(255, 255, 255, 0.07);
    --color-border-subtle:    rgba(255, 255, 255, 0.04);
    --color-border-emphasis:  rgba(255, 255, 255, 0.14);

    /* ── Text ─────────────────────────────────────────────────────────── */
    --color-text-primary:     rgb(237, 237, 240);   /* near white, not harsh */
    --color-text-secondary:   rgb(160, 160, 170);   /* muted — for labels, captions */
    --color-text-tertiary:    rgb(100, 100, 110);   /* very muted — timestamps, hints */

    /* ── Accent (the only real colour on screen by default) ───────────── */
    --color-accent:           rgb(99, 102, 241);    /* indigo — restrained, professional */
    --color-accent-hover:     rgb(118, 121, 255);
    --color-accent-muted:     rgba(99, 102, 241, 0.12);  /* tinted backgrounds */
    --color-on-accent:        rgb(255, 255, 255);

    /* ── Semantic ─────────────────────────────────────────────────────── */
    --color-success:          rgb(52, 211, 153);
    --color-warning:          rgb(251, 191, 36);
    --color-error:            rgb(248, 113, 113);

    /* ── Typography ───────────────────────────────────────────────────── */
    --size-text-xs:           11px;
    --size-text-sm:           13px;
    --size-text-base:         15px;
    --size-text-lg:           18px;
    --size-text-xl:           24px;
    --size-text-2xl:          32px;
    --size-text-3xl:          48px;

    /* ── Spacing ──────────────────────────────────────────────────────── */
    --space-1:                4px;
    --space-2:                8px;
    --space-3:                12px;
    --space-4:                16px;
    --space-6:                24px;
    --space-8:                32px;
    --space-10:               40px;
    --space-12:               48px;

    /* ── Radius ───────────────────────────────────────────────────────── */
    --radius-sm:              4px;
    --radius-md:              8px;
    --radius-lg:              12px;
    --radius-xl:              16px;
    --radius-full:            999px;

    /* ── Animation ────────────────────────────────────────────────────── */
    --duration-instant:       80ms;
    --duration-fast:          150ms;
    --duration-normal:        250ms;
    --duration-slow:          400ms;
    --ease-out:               cubic-bezier(0.0, 0.0, 0.2, 1);
    --ease-in-out:            cubic-bezier(0.4, 0.0, 0.2, 1);
    --ease-spring:            cubic-bezier(0.34, 1.56, 0.64, 1);
}
```

The indigo accent (`#6366f1`) is the Linear default. For a more Vercel-adjacent feel, swap it to white/near-white — Vercel's UI is almost monochrome with white as the interactive colour. Both are in the realm of "professional dark", one warmer, one colder.

---

## Surface and Panel Styling

### The Panel Stack

Every panel uses the same base pattern. Surfaces stack by raising the background colour value slightly — not by using shadows.

```css
/* panels.uss */

.panel-root {
    flex-grow: 1;
    background-color: var(--color-background);
    padding: var(--space-6);
}

/* A contained panel — cards, dialogs, settings panes */
.panel-surface {
    background-color: var(--color-surface);
    border-width: 1px;
    border-color: var(--color-border);
    border-radius: var(--radius-lg);
    padding: var(--space-6);
}

/* Elevated above panel-surface — dropdowns, tooltips, modals */
.panel-raised {
    background-color: var(--color-surface-raised);
    border-width: 1px;
    border-color: var(--color-border-emphasis);
    border-radius: var(--radius-lg);
    padding: var(--space-4);
}

/* Header within a panel */
.panel-header {
    flex-direction: row;
    justify-content: space-between;
    align-items: center;
    padding-bottom: var(--space-4);
    border-bottom-width: 1px;
    border-color: var(--color-border-subtle);
    margin-bottom: var(--space-6);
}

.panel-footer {
    padding-top: var(--space-4);
    border-top-width: 1px;
    border-color: var(--color-border-subtle);
    margin-top: var(--space-6);
    flex-direction: row;
    justify-content: flex-end;
}

.divider {
    height: 1px;
    background-color: var(--color-border-subtle);
    margin: var(--space-4) 0;
}
```

### Glass Panel Variant

The glass variant is for panels that float visually over other content — a settings overlay, a tooltip, a confirmation dialog. In the Vercel/Linear aesthetic this effect is very subtle — mostly just the border change and a near-transparent fill.

```css
/* The "glass" in this aesthetic is 90% about the border, 10% about the fill */
.glass-panel {
    background-color: rgba(16, 16, 24, 0.72);
    border-width: 1px;
    border-color: rgba(255, 255, 255, 0.10);
    border-radius: var(--radius-xl);
    padding: var(--space-6);
}

/* Optional: use USS filter for a blur-behind effect on a sibling element.
   Performance cost: renders subtree to intermediate texture. Use sparingly. */
.glass-blur-layer {
    position: absolute;
    top: 0; left: 0; right: 0; bottom: 0;
    filter: blur(20px);
}
```

For the Shader Graph sheen that completes the glass effect — a subtle bright gradient along the top edge — see the Shader Graph section below.

---

## Typography

Clean typography carries most of the hierarchy in this aesthetic. The key principle: contrast comes from weight and opacity, not from using many font sizes.

```css
/* typography.uss */

.text-title {
    font-size: var(--size-text-2xl);
    color: var(--color-text-primary);
    -unity-font-style: bold;
    letter-spacing: -0.5px;  /* slight tracking-tight — characteristic of Linear */
}

.text-heading {
    font-size: var(--size-text-xl);
    color: var(--color-text-primary);
    -unity-font-style: bold;
    letter-spacing: -0.3px;
}

.text-subheading {
    font-size: var(--size-text-lg);
    color: var(--color-text-secondary);
    -unity-font-style: normal;
}

.text-body {
    font-size: var(--size-text-base);
    color: var(--color-text-primary);
    white-space: normal;
}

.text-label {
    font-size: var(--size-text-sm);
    color: var(--color-text-secondary);
    -unity-font-style: bold;
    letter-spacing: 0.3px;
}

.text-caption {
    font-size: var(--size-text-xs);
    color: var(--color-text-tertiary);
}

/* Accent text — used for active states, links, CTA labels */
.text-accent {
    color: var(--color-accent);
    font-size: var(--size-text-sm);
    -unity-font-style: bold;
}

/* Monospace — for module step codes, identifiers */
.text-mono {
    font-size: var(--size-text-sm);
    color: var(--color-text-secondary);
}
```

---

## Buttons

```css
/* buttons.uss */

/* Primary — accent fill. Used once per panel maximum */
.btn-primary {
    background-color: var(--color-accent);
    color: var(--color-on-accent);
    border-radius: var(--radius-md);
    border-width: 0;
    padding: var(--space-2) var(--space-4);
    font-size: var(--size-text-sm);
    -unity-font-style: bold;

    transition-property: background-color, scale;
    transition-duration: var(--duration-instant), var(--duration-instant);
    transition-timing-function: var(--ease-out);
}
.btn-primary:hover  { background-color: var(--color-accent-hover); scale: 1.02 1.02; }
.btn-primary:active { scale: 0.97 0.97; }

/* Secondary — bordered, no fill. The workhorse button */
.btn-secondary {
    background-color: transparent;
    color: var(--color-text-primary);
    border-width: 1px;
    border-color: var(--color-border-emphasis);
    border-radius: var(--radius-md);
    padding: var(--space-2) var(--space-4);
    font-size: var(--size-text-sm);

    transition-property: border-color, background-color, scale;
    transition-duration: var(--duration-instant);
    transition-timing-function: var(--ease-out);
}
.btn-secondary:hover  {
    border-color: rgba(255, 255, 255, 0.24);
    background-color: rgba(255, 255, 255, 0.04);
    scale: 1.01 1.01;
}
.btn-secondary:active { scale: 0.97 0.97; }

/* Ghost — invisible until hover. Used for icon buttons and secondary actions */
.btn-ghost {
    background-color: transparent;
    color: var(--color-text-tertiary);
    border-width: 0;
    padding: var(--space-2);
    border-radius: var(--radius-sm);

    transition-property: color, background-color;
    transition-duration: var(--duration-instant);
}
.btn-ghost:hover {
    color: var(--color-text-primary);
    background-color: rgba(255, 255, 255, 0.06);
}

/* Tab button */
.btn-tab {
    background-color: transparent;
    color: var(--color-text-secondary);
    border-width: 0 0 1px 0;
    border-color: transparent;
    border-radius: 0;
    padding: var(--space-2) var(--space-4);
    font-size: var(--size-text-sm);

    transition-property: color, border-color;
    transition-duration: var(--duration-fast);
}
.btn-tab--active {
    color: var(--color-text-primary);
    border-color: var(--color-accent);
}
.btn-tab:hover:not(.btn-tab--active) {
    color: var(--color-text-primary);
}
```

---

## Cards and Module List Items

```css
/* components.uss */

.card {
    background-color: var(--color-surface);
    border-width: 1px;
    border-color: var(--color-border);
    border-radius: var(--radius-lg);
    padding: var(--space-4);
    margin-bottom: var(--space-2);

    transition-property: border-color, translate, background-color;
    transition-duration: var(--duration-instant);
    transition-timing-function: var(--ease-out);
}

/* Hover: border brightens + slight lift. No glow by default */
.card:hover {
    border-color: var(--color-border-emphasis);
    translate: 0 -2px;
    background-color: var(--color-surface-raised);
}

/* Active/selected state: accent-tinted border */
.card--active {
    border-color: var(--color-accent);
    background-color: var(--color-accent-muted);
}

.card--locked {
    opacity: 0.4;
}

/* Progress bar — used inside cards and on the dashboard */
.progress-track {
    height: 3px;  /* Intentionally thin — Vercel/Linear use very thin bars */
    background-color: var(--color-surface-overlay);
    border-radius: var(--radius-full);
    overflow: hidden;
}

.progress-fill {
    height: 100%;
    background-color: var(--color-accent);
    border-radius: var(--radius-full);
    transition-property: width;
    transition-duration: var(--duration-slow);
    transition-timing-function: var(--ease-out);
}

/* Badge — status pill */
.badge {
    border-radius: var(--radius-full);
    padding: 2px var(--space-2);
    font-size: var(--size-text-xs);
    -unity-font-style: bold;
}
.badge--success { background-color: rgba(52, 211, 153, 0.12); color: rgb(52, 211, 153); }
.badge--warning { background-color: rgba(251, 191, 36, 0.12);  color: rgb(251, 191, 36); }
.badge--error   { background-color: rgba(248, 113, 113, 0.12); color: rgb(248, 113, 113); }
.badge--default { background-color: var(--color-surface-raised); color: var(--color-text-secondary); }
```

---

## Inputs

```css
/* inputs.uss */

.input-group { margin-bottom: var(--space-4); }

.input-label {
    font-size: var(--size-text-xs);
    color: var(--color-text-secondary);
    -unity-font-style: bold;
    letter-spacing: 0.4px;
    margin-bottom: var(--space-1);
}

.input-field {
    background-color: var(--color-surface-raised);
    border-width: 1px;
    border-color: var(--color-border);
    border-radius: var(--radius-md);
    color: var(--color-text-primary);
    padding: var(--space-2) var(--space-3);
    font-size: var(--size-text-base);

    transition-property: border-color;
    transition-duration: var(--duration-fast);
}
.input-field:focus       { border-color: var(--color-accent); }
.input-field--error      { border-color: var(--color-error); }

/* Slider track */
.input-field .unity-base-slider__tracker {
    background-color: var(--color-surface-overlay);
    border-radius: var(--radius-full);
    height: 3px;
    border-width: 0;
}

/* Slider thumb */
.input-field .unity-base-slider__dragger {
    background-color: var(--color-text-primary);
    border-radius: var(--radius-full);
    width: 14px;
    height: 14px;
    margin-top: -5px;
    border-width: 0;
    transition-property: scale, background-color;
    transition-duration: var(--duration-instant);
}
.input-field .unity-base-slider__dragger:hover {
    background-color: var(--color-accent);
    scale: 1.2 1.2;
}
```

---

## Transitions and Animations

### The Safe Properties

Only animate these on Quest 2 without hesitation — they bypass layout recalculation:

```
translate    rotate    scale    opacity    background-color
border-color    color    filter (careful — see below)
```

Never animate `width`, `height`, `padding`, `margin`, `left`, `top` for visual effects. Use `scale` or `translate` instead.

### Panel Entry — Fade and Slide Up

```css
/* animations.uss */

.panel-root {
    opacity: 0;
    translate: 0 8px;
    transition-property: opacity, translate;
    transition-duration: var(--duration-normal), var(--duration-normal);
    transition-timing-function: var(--ease-out), var(--ease-out);
}

.panel-root.visible {
    opacity: 1;
    translate: 0 0;
}

.hidden { display: none; }
.invisible { opacity: 0; }

/* Reduced motion — applied to root when user preference is set */
.reduced-motion * {
    transition-duration: 0s !important;
}
```

```csharp
// Host.Show() — one-frame delay is required for transition to fire
public override void Show()
{
    Root.RemoveFromClassList(UIStyles.Hidden);
    Root.schedule.Execute(() =>
        Root.AddToClassList(UIStyles.Visible)
    ).StartingIn(16);
}

public override void Hide()
{
    Root.RemoveFromClassList(UIStyles.Visible);
    Root.schedule.Execute(() =>
        Root.AddToClassList(UIStyles.Hidden)
    ).StartingIn((long)(250)); // match transition duration
}
```

### Staggered List Entry

No native USS equivalent. C# schedule loop:

```csharp
private void AnimateListEntries(VisualElement list)
{
    int i = 0;
    foreach (var child in list.Children())
    {
        child.AddToClassList("entry-hidden"); // opacity:0, translate: 0 6px
        int index = i++;
        child.schedule.Execute(() => {
            child.RemoveFromClassList("entry-hidden");
            child.AddToClassList("entry-visible"); // opacity:1, translate: 0 0
        }).StartingIn(index * 50L); // 50ms stagger
    }
}
```

### Number Counter (Progress / Stats)

Animating a numeric value display — common in dashboards:

```csharp
private void AnimateCounter(Label label, float from, float to, float duration = 0.6f)
{
    float elapsed = 0f;
    label.schedule.Execute(timer => {
        elapsed += timer.deltaTime / 1000f;
        float t = Mathf.Clamp01(elapsed / duration);
        // Ease out cubic
        float eased = 1f - Mathf.Pow(1f - t, 3f);
        label.text = Mathf.RoundToInt(Mathf.Lerp(from, to, eased)).ToString();
    }).Until(() => elapsed >= duration);
}
```

---

## Gradients — The Workaround Playbook

`linear-gradient()` is not in USS. The options in priority order for this aesthetic:

### 1. Baked textures for static decoration

For the very subtle gradient that appears in hero panels or behind primary CTAs. A near-transparent PNG tinted by USS.

```css
.hero-gradient-bg {
    background-image: url("project://Assets/UI/Textures/gradient-corner.png");
    -unity-background-scale-mode: stretch-to-fill;
    /* Tint with accent colour at very low opacity */
    -unity-background-image-tint-color: rgba(99, 102, 241, 0.08);
}
```

When generating gradient textures, raise the alpha channel to a power of 2 or 3 before export. A linear ramp looks flat and dated. Power 2 gives a fast-then-slow falloff — the texture looks like light catching a surface edge, not a colour fill.

### 2. SVG assets for crisp scalable gradients

SVGs natively support `linearGradient` and `radialGradient`. A designer creates them in Figma, exports SVG, Unity renders them at any resolution without texture memory overhead.

```css
.card-accent-bg {
    background-image: url("project://Assets/UI/Vectors/accent-radial.svg");
    -unity-background-scale-mode: stretch-to-fill;
}
```

**Best for:** Card accent backgrounds, decorative section highlights. Colours are baked into the SVG — not theme-reactive. Either provide multiple SVG variants per theme or use Option 3 for theme-reactive gradients.

### 3. Shader Graph material for dynamic and animated gradients

The correct approach for anything theme-reactive or animated. Assigned via USS `-unity-material`. Runs per-element on the GPU.

```css
.card-gradient {
    -unity-material: url("project://Assets/UI/Materials/CardGradient.mat");
}
```

The Power curve in the shader is the difference between a 2004 gradient and a 2026 one. Linear UV (power 1.0) = flat, boring. Power 2.0 = quadratic ease, smooth. Power 3.0 = fast highlight, slow fade — reads as a light source hitting a surface from one angle.

```
/* Conceptual shader graph — CardGradient */
Element Layout UV
    → Y component (0=top, 1=bottom)
    → 1 - Y  (flip: top=1, bottom=0)
    → Power(node: 2.5)
    → Multiply(intensity: 0.06)  ← very subtle by default
    → Add to Default Solid output
    → Render Type Branch (Solid port) → Fragment Base Color
```

For a more vibrant theme with a visible gradient, raise the intensity parameter. The material exposes it as a Blackboard property so it can be set via `material.SetFloat("_Intensity", 0.18f)` from the `ThemeManager`.

**Performance note:** One simple gradient shader is 5–10 GPU instructions — effectively free. Every distinct material breaks UITK batching. Keep distinct materials to a small set. Prefer controlling variation via shader properties, not separate materials.

---

## Shader Graph Per-Element Effects

All of the following use the `-unity-material` USS property and stay within the UITK pipeline. No separate camera passes or uGUI overlays required.

### Glass Panel Sheen

The bright highlight along the top edge of a glass panel. This is what sells the glass surface — more important than the blur.

```
Element Layout UV
    → Y component
    → 1 - Y  (top=1)
    → Power(2.5)                    ← fast falloff
    → Multiply(sheenIntensity: 0.08)
    → Add to base glass colour
    → Render Type Branch (Solid) → Fragment
```

At `sheenIntensity: 0.08` this is barely perceptible — just enough that the surface reads differently at the top vs bottom. For the Linear-style feel, keep it very subtle. For a more sci-fi aesthetic, push to `0.20–0.30`.

### Border Glow (Replaces CSS `box-shadow`)

CSS `box-shadow` doesn't exist in USS. This shader adds a soft inner glow near the element's border, which reads similarly to an outer glow on the panel.

```
Element Layout UV → remap to -1..1 (Multiply 2, Subtract 1)
    → Abs
    → Max(X, Y)                    ← outermost edge of the element
    → SmoothStep(0.80, 1.0)        ← only the outer ~20%
    → Multiply(glowColour)
    → Add to Default Solid output
    → Render Type Branch (Solid) → Fragment
```

At low intensity with an accent colour this reads as a subtle selection glow. Higher intensity gives you the neon border look. Controlled by a Blackboard `_GlowColour` and `_GlowIntensity` property — set to near-zero by default, overridden by the glow theme.

### Card Hover Sheen

The bright light sweep across a card on hover — a technique used heavily in Linear's interface.

```
_HoverProgress (float 0→1, animated from C#)

Element Layout UV → X component
    → abs(X - _HoverProgress)     ← distance from sweep position
    → 1 - SmoothStep(0.0, 0.25)  ← soft bright band ~25% wide
    → Multiply(0.08)              ← subtle
    → Add to Default Solid output
```

```csharp
// Animate from View.Wire() — card registers pointer events
card.RegisterCallback<PointerEnterEvent>(_ => {
    var mat = card.resolvedStyle.unityMaterial;
    if (mat == null) return;

    float elapsed = 0f;
    const float duration = 0.5f;
    card.schedule.Execute(timer => {
        elapsed += timer.deltaTime / 1000f;
        mat.SetFloat("_HoverProgress", Mathf.Clamp01(elapsed / duration));
    }).Until(() => elapsed >= duration);
});
```

### Vignette

Applied to the outermost panel container. Darkens the edges of the screen, reinforcing the "VR display" quality and drawing focus to the centre.

```
Element Layout UV → remap to -1..1
    → Length (distance from centre)
    → Power(1.8)
    → Multiply(vignetteStrength: 0.20)
    → Clamp(0, 1)
    → 1 - result
    → Multiply with Default Solid output
    → Render Type Branch (Solid) → Fragment
```

Default `vignetteStrength: 0.20` is tasteful — edges are 20% darker. The full-screen vignette Shader Graph for the render texture overlay is covered in the bonus doc.

### Scanlines

A very subtle CRT texture effect. In a VR context this reads as "this is a screen" rather than "this is a flat plane".

```
Element Layout UV → Y component
    → Multiply(lineFrequency: 100.0)
    → Fraction
    → Step(0.5)
    → 1 - result
    → Multiply(lineIntensity: 0.025)   ← very subtle on Quest 2
    → 1 - result
    → Multiply with Default Solid output
```

Keep `lineIntensity` below `0.04` for Quest 2. Strong scanlines will strobe at 72fps. The goal is texture, not distraction.

---

## The Customisable Layer — Glow and Colour Themes

The default theme is intentionally minimal. The Shader Graph materials all expose Blackboard properties. The `UITheme` ScriptableObject maps to those properties. A designer wanting a "neon glow" theme sets:

```
GlowIntensity:    0.30      (was 0.02)
SheenIntensity:   0.25      (was 0.08)
GradientIntensity: 0.18     (was 0.05)
ColorAccent:      #00ff88   (was indigo)
```

And sets those on the material at theme-switch time from `ThemeManager`:

```csharp
private void ApplyShaderProperties(UITheme theme)
{
    if (theme.GlowMaterial != null)
    {
        theme.GlowMaterial.SetColor("_GlowColour",    theme.ColorAccent);
        theme.GlowMaterial.SetFloat("_GlowIntensity", theme.GlowIntensity);
    }

    if (theme.GradientMaterial != null)
    {
        theme.GradientMaterial.SetColor("_GradientColour",    theme.ColorAccent);
        theme.GradientMaterial.SetFloat("_GradientIntensity", theme.GradientIntensity);
    }
}
```

The professional default ships quiet. The vibrant variant is a preset the designer creates in the inspector.

---

## What Cannot Be Done Within the UITK Pipeline

These effects require going outside the `-unity-material` per-element approach. They are documented in the bonus doc (Global Render Texture Overlay) and are entirely optional — the UITK pipeline delivers a complete and good-looking UI without them.

- **Full-screen scanlines/vignette as a post-process** over the complete render texture
- **True depth-of-field or bloom** affecting the VR panel as a whole
- **Chromatic aberration** on the panel edges
- **Blurring the 3D world visible through a glass panel** (requires a scene render texture read)

---

## File Structure

```
Assets/UI/
├── Styles/
│   ├── base-variables.uss
│   ├── components/
│   │   ├── typography.uss
│   │   ├── panels.uss
│   │   ├── buttons.uss
│   │   ├── inputs.uss
│   │   ├── components.uss     (cards, badges, progress)
│   │   └── animations.uss
│   └── themes/
│       ├── theme-default.uss
│       ├── theme-vibrant.uss  (accent colours, higher glow)
│       └── theme-light.uss
│
├── Materials/
│   ├── UI_GlassPanel.mat      (sheen + subtle gradient)
│   ├── UI_CardGradient.mat    (card bg gradient, very subtle default)
│   ├── UI_BorderGlow.mat      (selection / hover glow)
│   ├── UI_CardSheen.mat       (hover sweep)
│   └── UI_ScreenBase.mat      (vignette + scanlines on root panel)
│
├── Shaders/
│   ├── GlassPanel.shadergraph
│   ├── CardGradient.shadergraph
│   ├── BorderGlow.shadergraph
│   ├── CardSheen.shadergraph
│   └── ScreenBase.shadergraph
│
├── Textures/
│   ├── gradient-corner-tl.png
│   └── gradient-radial-centre.png
│
└── Vectors/
    ├── accent-radial.svg
    └── accent-linear-horizontal.svg
```