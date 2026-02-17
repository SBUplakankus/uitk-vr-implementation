# Quest 2 VR UI Performance Reference

> **Target**: Meta Quest 2 (Snapdragon XR2, Adreno 650)  
> **Framework**: UI Toolkit + uGUI for Meta SDK interactables  
> **Sources**: Unity 6.3 official documentation, Meta developer guidelines, Unite 2024 performance session

---

## A Note on Numbers

**No performance number in this document should be treated as a fact until you measure it on your device.** What this document provides instead is verified architectural understanding and profiling methodology so you can get real numbers yourself.

---

## Frame Budget (What's Actually Confirmed)

The frame rate target is 72 FPS, giving ~13.8ms per frame. Of that, approximately 2ms is reserved for the timewarp and guardian system, leaving around 11.8ms for application rendering.

Your UI rendering is competing directly with world rendering, physics, and audio within that 11.8ms. There is no confirmed breakdown of how much of that budget UI "should" consume — that depends entirely on your scene complexity. Measure it.

---

## How UITK Rendering Actually Works (Unity 6.3 Official)

Understanding this is more useful than any made-up timing table.

### The Uber Shader

UI Toolkit consolidates all UI rendering functionality into a single "uber shader." Rather than rely on multiple shader variants, this shader uses dynamic branching to select the appropriate rendering path at runtime. This reduces CPU overhead by minimizing shader switches but does add some GPU cost due to the branching logic.

This is important: UITK's rendering is not a naive "one draw call per element" system. It's an opinionated pipeline with specific constraints you need to work within.

### The 8-Texture Limit

The uber shader supports up to eight textures within the same batch, allowing elements with different textures to render in the same draw call. Exceeding the eight-texture limit forces the batching system to split into separate batches, increasing overhead.

This is a hard constraint. If your VR panel uses more than 8 unique un-atlased textures in a single hierarchy, you will pay draw call overhead regardless of how well you've written your USS. Texture atlasing is the mitigation.

### Batching Fundamentals

To batch elements efficiently they must share the same GPU state — the same shaders, textures, mesh data, and other GPU-specific parameters. For example, a sequence of text elements using the same font and style can be batched together. However, inserting an image between them requires different GPU settings, forcing a new batch.

The practical implication for design: element order in the hierarchy matters. Interleaving text and images breaks batches even if everything uses the same stylesheet. Grouping same-type elements where possible reduces draw call overhead.

### Vertex Buffers

When a UIDocument creates a Panel at runtime, it pre-allocates a single vertex buffer. If the UI exceeds the capacity of this buffer, additional buffers are created, which fragments batching and increases draw calls. The Vertex Budget in Panel Settings configures the initial size of the vertex buffer — the default value of 0 lets Unity determine the size automatically, but for complex UIs, manually increasing this value can improve performance.

For world-space VR panels this is worth tuning once you have a complex UI built. Profile with the Frame Debugger to see if you're spilling across multiple vertex buffers.

### Update Mechanisms and Their Costs

The Unity 6.3 docs confirm four update mechanisms that affect performance, in rough order of cost:

**Style resolution** — triggered when USS classes are added/removed or inline styles are changed. Large or deeply nested hierarchies make this expensive. Minimise frequent class toggling on elements with many children.

**Layout recalculation** — triggered by changes to element size, position, or alignment. This is why animating `width`, `height`, `left`, or `top` properties is expensive — each frame triggers a full layout pass. Use `transform: translate/scale` for animation instead, which bypasses layout recalculation entirely.

**Vertex buffer updates** — triggered when element geometry changes (rounded corners, borders). Avoid modifying `border-radius` or border properties at runtime unless necessary.

**Rendering state changes** — triggered by masking or unique textures that break batching. Each mask in UITK requires a stencil operation that breaks the current batch. Use them sparingly.

---

## The USS-First Argument (What's Actually True)

USS gradients render as **vertex-coloured geometry** tessellated at build time, not at runtime. The GPU interpolates vertex colours during rasterisation, which is essentially free. A shader-based gradient evaluates per-pixel in the fragment stage, which costs real fill-rate.

On a mobile TBDR GPU like the Adreno 650, fill-rate is your scarcest resource when stereo rendering at 1832×1920 per eye. Fragment shader cost scales with screen coverage. USS gradient cost does not. The principle is sound even if the exact multiplier requires your own profiling to establish.

**What USS can do that keeps you out of the fragment shader:**

- Linear and radial gradients
- Solid fills with alpha
- Border radius (rounded corners)
- Border styling
- Box shadow (note: may not be supported on all UITK versions — check docs)
- Opacity
- Scale, translate, rotate via transforms
- Colour transitions via USS `transition` property

**What USS cannot do and therefore genuinely needs a shader:**

- Background blur (glass morphism with real blur — not simulated with opacity)
- Procedural animated noise or distortion
- UV-scrolling textures
- Depth-aware effects

The "fake" glass morphism approach — semi-transparent fill + thin border + subtle background gradient — is a legitimate design choice, not a compromise forced by performance. It's what most high-quality apps actually do.

---

## UITK-Specific VR Considerations

These are specific to the world-space render texture pipeline used for UITK in VR, which differs from standard screen-space UI.

**Render texture resolution is your call.** Unlike screen-space UI where resolution is fixed to the display, you choose the Panel Settings render texture resolution for world-space panels. Higher resolution = sharper text, more GPU memory, more bandwidth. Lower resolution = blurrier but cheaper. This is a design tradeoff you need to tune on device, not a formula problem.

**World-space UITK renders to a texture first, then to a quad in the scene.** This means every world-space UI panel incurs at least one additional render target blit. If you have multiple UITK panels visible simultaneously (e.g. multiple tablet screens), each has its own render texture cost.

**Single Pass Instanced rendering and UITK.** Meta's SPI renders both eyes in one pass. Standard scene geometry benefits from this. Whether your UITK render texture setup interacts cleanly with SPI depends on how you configure the panel and the world-space quad's material. Test this specifically — some setups have been reported to render the texture correctly to one eye only without explicit configuration.

**The Poke Interactable layer is uGUI, not UITK.** The visual layer (UITK) and interaction layer (uGUI + Meta SDK) are separate systems. The event bridge between them — translating poke contacts into UITK pointer events — is custom work. This bridge itself has performance implications if it runs every frame. Design it to be event-driven rather than polling.

---

## Confirmed Batching Breakers (Avoid These)

Sourced from Unity 6.3 official documentation:

- **More than 8 unique textures** in a single panel hierarchy — breaks the uber shader batch
- **Masking** — each mask requires a stencil operation that forces a batch break
- **Custom materials on individual elements** — exits the uber shader entirely, each unique material is a separate draw call
- **Vertex buffer overflow** — too many elements without adjusting Vertex Budget in Panel Settings
- **Interleaving different element types** in the hierarchy (text between images, etc.) — forces batch breaks even with shared styles

---

## Profiling Methodology (Meta's Official Guidance)

Lock your CPU/GPU level before profiling to get consistent results. To use Unity Profiler with a standalone Meta Quest headset, you can profile your app as it runs on the headset via ADB or WiFi. Make sure to lock your CPU/GPU level before profiling to get consistent profiling results and measure improvements.

Focus on bottlenecks first and change one thing at a time, such as resolution, hardware, quality, or configuration. It can be useful to disable Multithreaded Rendering in Player Settings during debugging — this slows down the renderer but gives a clearer view of frame time. Turn it back on when done.

**Tools to use:**

- Unity Profiler via ADB — for CPU/GPU frame time
- Unity Frame Debugger — for draw call inspection and batch analysis
- RenderDoc (sideloaded on Quest) — for detailed GPU frame capture
- Unity's Overdraw mode in Scene View — to spot expensive fill-rate areas

**Never profile UI on desktop hardware.** Quest 2's GPU crosses the 1 TFLOP mark, putting it in the same performance territory as the original Xbox One. A desktop GPU with 10-30 TFLOPS will make your shaders look essentially free. The performance cliff between desktop and Quest 2 is real and large.

---

## What to Actually Benchmark

Since we're not providing fabricated timing tables, here are the benchmarks worth running yourself once you have a working UI:

1. **Baseline draw call count** — open Frame Debugger, count draw calls with a representative screen visible. Note it down. Every optimisation should be compared against this number.

2. **USS-only panel vs same panel with custom shader** — implement a visual effect both ways, profile each separately on device. This gives you your actual multiplier for your specific hardware and effect.

3. **Render texture resolution sweep** — render the same panel at 512×512, 1024×1024, and 2048×2048. Profile GPU frame time each time. Find the crossover where quality gain stops justifying cost.

4. **Texture count at batch limit** — build a panel with 7, 8, and 9 unique textures. Profile the draw call count. Confirm the 8-texture limit is real in your Unity version and see what happens when you cross it.

5. **Layout animation vs transform animation** — animate a panel sliding in via `left` property vs `translate` property. Profile the update mechanism cost each way. This is the most impactful optimisation for common VR menu transitions.

---

## Summary: The Principles That Are Actually Verified

| Principle                                                            | Source                             | Confidence |
| -------------------------------------------------------------------- | ---------------------------------- | ---------- |
| USS gradients use vertex geometry, not fragment shaders              | Unity architecture docs            | High       |
| Uber shader handles up to 8 textures per batch                       | Unity 6.3 Manual                   | High       |
| Exceeding 8 textures breaks batching                                 | Unity 6.3 Manual                   | High       |
| Masking breaks batches                                               | Unity 6.3 Manual                   | High       |
| Layout property animation is more expensive than transform animation | Unity 6.3 Manual                   | High       |
| Frame budget is ~11.8ms after timewarp reservation                   | Meta dev docs                      | High       |
| Desktop profiling does not represent Quest 2 performance             | General knowledge                  | High       |
| Specific ms timings for USS vs shader effects                        | **Not sourced — measure yourself** | **N/A**    |
| "USS is 15x faster than shaders"                                     | **Fabricated — do not use**        | **N/A**    |

---

*This document supersedes the previous performance analysis. All performance claims are sourced or explicitly marked as requiring device measurement.*