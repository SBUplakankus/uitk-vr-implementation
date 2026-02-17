# Industry UI Toolkit Workflow

> **Basis**: Unite 2024 session recordings, Unity 6.3 official roadmap posts, confirmed shipped titles  
> **Scope**: What studios are actually doing with UITK in production, as of early 2026

---

## The Honest Landscape

UITK in production is still rare. Most shipped Unity games using it are either Unity's own samples or early adopters who partnered directly with Unity engineering. The best documented real-world case is **Football Manager 2025**, which is the only publicly discussed large-scale UITK shipping title as of writing. Everything else is either small indie projects, internal Unity samples, or studios still migrating.

This matters for calibrating expectations. There is no ten-year body of UITK production patterns the way there is for UGUI. The "industry standard" is still being established.

---

## Football Manager 2025 — The Closest Thing to a Production Reference

Sports Interactive migrated FM25 to Unity from a proprietary engine, making it one of the first major commercial titles to ship UITK at scale. They presented the technical detail at Unite 2024 in the session *"Tackling UI Challenges in Football Manager 2025"* (recording publicly available).

### What They Actually Did

FM25's UI/UX is driven by what they called a "tile and card" system. A tile is a panel of information with multiple states, from small amounts of information to larger cards which contain more material. This is a data-driven component model — the same tile type renders differently depending on what data is bound to it.

Their workflow had designers and developers working together to create prototypes using ScriptableObjects as data sources. Data sources are established at a root container and propagate through the hierarchy so all child elements inherit the necessary data. Elements bind to data sources using a path-based system, with binding modes determining how the UI and data source interact — allowing designers to make dynamic, data-driven changes without touching code.

For styling, they emphasised reusability: inline styles allowed immediate visual tweaks, while USS selectors enabled styles to be shared across elements — following the same logic as CSS class reuse on the web.

Sports Interactive worked closely with Unity's engineers directly to adapt the engine to their needs. This is not a workflow most teams can replicate, but it tells you where the pain points were significant enough to require engine-level collaboration.

### Key Takeaway for This Project

The FM25 architecture — ScriptableObject data sources, path-based binding, designer-editable without code — maps directly onto the ScriptableObject theme preset idea in this project's goals. It's validation that the instinct is correct, and it's the pattern to study in the Unite 2024 demo project (downloadable via the Unity Discussions post linked in the References page).

---

## What the Unite 2024 Performance Session Said

The second relevant Unite 2024 session was *"Getting the Best Performance with UI Toolkit"* led by Nicolas Borromeo. The confirmed topics covered were:

Chained draw calls, buffer sizes, dynamic atlasing, and handling limitations including custom shaders.

These are the real levers that matter for Quest 2. No summary of this session is available in text form — watch it. It is the most directly applicable publicly available performance resource for UITK that exists.

---

## Practical Patterns That Have Emerged

These come from the Unity 6.3 roadmap threads, the BagelGame sample, and the FM25 talk combined — not from a single authoritative source.

### UXML for Structure, USS for Everything Visual

The consistent pattern across all official samples and the FM25 talk is strict separation: UXML defines hierarchy and names, USS defines all visual properties, C# only touches logic and binding. Runtime style manipulation via inline C# style properties is treated as a last resort, not a default.

**Why this matters for your project**: if creatives need to adjust visual output, they should be able to do it entirely in USS without opening a C# file. The ScriptableObject theme system can swap USS variables, but the USS files themselves should be readable and editable by non-programmers.

### Data Binding Over Manual Updates

Unity 6 native `SerializedObject` binding is the expected approach for any value that changes at runtime. Manual `label.text = value` assignments in Update loops are considered legacy — they bypass the binding system and create maintenance overhead. The FM25 workflow confirmed this: binding paths propagate through the hierarchy from a root data source.

**The caveat**: binding has garbage allocation implications that matter on Quest 2. The `TrackPropertyValue` approach generates less GC pressure than full `Bind()` on frequently-updating properties. The Unite 2024 performance session covers this — watch it before choosing your binding strategy.

### Component Naming and File Structure

A clear directory split that emerged from community practice: UXML in a Layouts folder, USS in Styles, C# in Scripts, and art assets in their own folder. Screens are full UI compositions, Components are reusable pieces. Flat structure for screens (easy to browse), nested structure for components (because there will be many of them).

File naming uses PascalCase for C# (matching Unity conventions), and descriptive kebab-case for sprite assets. Version suffixes in filenames like `Buttons_final_REALLY.uss` are explicitly called out as anti-patterns — use Git branches for experimentation, not filename suffixes.

### USS Custom Properties (Variables) as the Theme Interface

The consistent approach for theming is `:root` custom properties in USS, updated at runtime by swapping stylesheet assets or patching variable values. This keeps the theme contract explicit — designers know which variables exist and what they control, and can adjust them without understanding the C# side.

---

## What Does Not Exist Yet

Being honest about gaps in publicly available practice:

**No established VR/XR UITK workflow.** FM25 is desktop/PC. BagelGame is desktop/PC. There are no shipped titles and no Unite talks specifically addressing UITK in a VR context. The Meta SDK interaction bridge is genuinely unsolved in public documentation — this project will be defining that pattern, not following one.

**No large-scale open source UITK projects.** The available reference implementations are all Unity-controlled samples. There is no community equivalent of the UGUI ecosystem that built up over a decade.

**Testing tooling is immature.** Unity 6.3 added some UI testing support, but there is no established equivalent to the web's component testing ecosystem. Unit testing ViewModels is straightforward; integration testing rendered UITK panels is not well documented.

---

## Patterns Worth Adopting From Adjacent Fields

Since production UITK patterns are sparse, the adjacent fields worth drawing from:

**Web component architecture** — the UXML/USS/C# split is structurally identical to HTML/CSS/JS. Patterns like BEM naming for USS classes, design token systems for theme variables, and component-driven development all transfer directly.

**Unity editor UITK** — the editor has used UITK since Unity 2019. Editor extension codebases are the largest body of real UITK code that exists. The patterns there (custom elements, binding, style management) are valid runtime patterns too, with some caveats around editor-only APIs.

**General Unity MVP/MVVM** — the Game Developer article on UI architecture for *Blood Runs Cold* is worth reading as a UGUI pattern reference. The system described was battle-tested in released mobile games and the architectural ideas (screen manager, presenter layer, event-driven updates) translate to UITK even if the implementation details differ.

---

## Summary: What to Actually Follow

| Source                         | What to take from it                                                                    |
| ------------------------------ | --------------------------------------------------------------------------------------- |
| FM25 Unite 2024 talk           | ScriptableObject data sources, designer-editable binding, tile/card component model     |
| Unite 2024 performance session | Draw call batching, atlas limits, binding GC — watch before finalising Quest 2 approach |
| BagelGame                      | World space panel setup, USS theming structure, shader filter integration               |
| Unity 6.3 roadmap threads      | Current feature availability, what's coming — treat as living reference                 |
| Web CSS/component patterns     | BEM naming, design tokens, component architecture — transfer directly to USS/UXML       |

Do not treat any single source as a complete playbook. The workflow for UITK in VR at this scale is being written now.

---

*All production references verified against public sources. FM25 architectural details sourced from the Unite 2024 session and Sports Interactive's own development blog.*
