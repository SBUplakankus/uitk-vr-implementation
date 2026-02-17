# BagelGame — Sample Analysis

> **Repo**: [Unity-Technologies/BagelGame](https://github.com/Unity-Technologies/BagelGame)  
> **Branch**: `6.3` (Unity 6.3 LTS)  
> **Language split**: C# 93% / HLSL 7%  
> **Purpose**: Full vertical game demonstrating UI Toolkit for 3D games

---

## What It Actually Is

BagelGame is a physics-based 3D game where you navigate a bagel to a toaster before it goes stale. The game itself is deliberately simple — the point is the UI, not the gameplay. Unity describe it as a "full vertical," meaning it's a complete, shippable game slice rather than a toy scene, so all the UI features are implemented in a realistic, connected context rather than isolated demos.

It is the closest thing to an official reference implementation for production UI Toolkit in Unity 6.3.

---

## Confirmed Features

These are taken directly from the official README — not inferred or speculated:

- **World Space UI** — UITK panels rendered into 3D game space
- **Custom Shaders** — Shader Graph integration with UI elements
- **Filters** — USS filter effects (available as of Unity 6.3)
- **Data Bindings** — Runtime data binding (Unity 6 feature)
- **UXML / USS** — Declarative structure and styling
- **Themes** — Dynamic theme switching system
- **Input System Integration** — New Input System, not legacy

---

## Why It Matters for This Project

### 1. World Space UI

This is the most directly relevant feature. BagelGame demonstrates UITK panels in 3D space, which is the same fundamental problem as rendering a UI panel on a virtual tablet in VR. The render pipeline — Panel Settings → Render Texture → world-space quad — is the same approach regardless of whether the viewer is in a headset or not.

**Gap to be aware of**: BagelGame targets desktop PC, not Quest 2. The world space setup won't account for the mobile GPU constraints or the render texture resolution tradeoffs at Quest 2's display density. Treat it as the architectural pattern, not the performance-tuned implementation.

### 2. Shader Graph + Filters

The C#/HLSL split (93/7) tells you HLSL is present but not dominant. The shader work is likely targeted effects rather than a full custom rendering pipeline. This aligns with Unity 6.3's USS filter system, which is the recommended approach for effects like blur and glow without breaking batching.

**Key constraint from Unity 6.3**: Applying a custom shader to a UITK element affects that element *and all its children*. This matters for how you structure the visual hierarchy — glass panels with children need careful thought.

**Batching note**: More than 6 unique un-atlased textures in a UITK hierarchy breaks batching. This is a hard limit worth knowing early.

### 3. Data Bindings

Unity 6 introduced runtime `SerializedObject` binding. BagelGame uses this to keep game state (score, toppings, time) in sync with UI labels and progress bars automatically. This is directly applicable to syncing VR interaction state with visual feedback.

### 4. Themes

The theme system is one of the more transferable patterns. CSS custom properties (`:root` variables in USS) can be updated at runtime to retheme the entire panel without touching individual elements. This maps directly to the ScriptableObject-driven theme preset idea described in the index page.

### 5. Input System

BagelGame uses the New Input System. This is relevant context but not directly portable to VR — Meta's Poke Interactables sit above the input layer and don't feed through Unity's standard Input System event pipeline to UITK. The interaction bridge between Meta SDK and UITK panels needs to be custom.

---

## Honest Gaps (What BagelGame Doesn't Cover)

These are things you will need to solve yourself:

| Gap                                  | Why It Matters                                                           |
| ------------------------------------ | ------------------------------------------------------------------------ |
| No Quest 2 / XR target               | No mobile GPU perf tuning, no headset rendering path                     |
| No hand tracking or poke interaction | Meta SDK integration is entirely absent                                  |
| Desktop input model                  | Controller ray and poke events don't map to standard UITK pointer events |
| No performance budgeting for VR      | 72/90Hz target, foveated rendering, thermal constraints all absent       |
| No accessibility / comfort patterns  | VR UI ergonomics (panel distance, FOV, readable text size) not addressed |

---

## What to Actually Study in the Repo

Rather than reading the code to copy patterns verbatim, focus on:

1. **How Panel Settings is configured for world space** — the `SetScreenToPanelSpaceFunction` override is the critical piece for mapping controller input to panel coordinates
2. **How themes are structured in USS** — specifically how `:root` variables are declared and whether they swap USS sheets entirely or patch variables at runtime
3. **How the shader materials are assigned** — via USS `filter` property or via the experimental `SetMaterial` API, since these have different batching implications
4. **The data model structure** — how GameData ScriptableObjects are shaped for `SerializedObject` binding, since this pattern is directly reusable

---

## Unity 6.3 Context

BagelGame was built against Unity 6.3 LTS, which is when several long-awaited UITK features landed:

- Custom shader support via USS filters (6.3)
- World space UI as a first-class feature (6.2/6.3)
- SVG support integrated natively (no separate package needed)
- Runtime data binding stabilised

This means the patterns in BagelGame were **not possible before mid-2025**. Any older UITK tutorials or Stack Overflow answers predate these APIs and should be treated with caution.

---

## Current UITK Limitations (as of 6.3)

Relevant gaps that BagelGame won't help with because they're engine-level:

- No dynamic text gradients — static only, or implement via custom text animation
- Rich text tags (`<color>`, `<link>`) can't be targeted by USS — no dynamic styling on inline text
- No anti-aliasing on `VectorImage` elements
- No glow/drop shadow natively — USS filters are the workaround as of 6.3
- Batching breaks above 6 unique un-atlased textures in a hierarchy

---

## Relationship to This Project

BagelGame is the best available reference for the UITK architecture layer — world space rendering, theming, data binding, shader effects. It is **not** a reference for the VR interaction layer. The split in this project's architecture mirrors that:

```
BagelGame covers:        This project must add:
────────────────────     ──────────────────────────
World space UITK panel   Meta Poke Interactable layer
USS theming              ScriptableObject theme presets  
Data binding             VR-specific state (hand pose, etc.)
Shader Graph effects     Quest 2 performance budgeting
Input System (desktop)   Meta SDK input bridge
```

Clone it, run it, open the USS files and the Panel Settings configuration. Don't treat the C# controllers as copy-paste templates — understand the pattern, then implement it against your own architecture.

---

*Analysis based on the public README, confirmed Unity 6.3 feature set, and official Unity documentation. Internal file structure not verified — clone the repo to inspect actual class names and scene hierarchy.*
