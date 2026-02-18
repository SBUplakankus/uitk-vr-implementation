# Bonus: Global Render Texture Overlay

> **Optional addition** — everything in the main styling doc works without this  
> **What this adds**: Full-screen scanlines, vignette, chromatic aberration as a
> single pass over the entire VR panel after all UITK content is drawn  
> **Pipeline**: uGUI RawImage or URP Blit sitting above the render texture

---

## The Architecture

The UITK panel renders to a `RenderTexture`. That texture is displayed on a world-space quad in the scene. The global overlay is a second uGUI `Image` or `RawImage` component that sits above the render texture in screen space, using a Shader Graph material that samples the render texture and applies effects to it as a full-screen pass.

```
Scene Camera
    ↓
UITK Panel → RenderTexture A
    ↓
World-space quad (displays RenderTexture A)
    ↓
Canvas (Screen Space - Camera)
    └─ RawImage (full-screen, Overlay material)
           ↓
           Shader samples RenderTexture A
           Applies: scanlines, vignette, chromatic aberration
           Outputs to screen
```

This is entirely separate from the UITK pipeline. The UITK document and all its panels are unaware of it. It is additive — disable the RawImage and the UI looks like the main styling doc. Enable it and the screen-wide effects layer on top.

---

## The Overlay Shader (Shader Graph, Canvas target)

This shader is assigned to the fullscreen RawImage. It samples the render texture and applies the effects.

**Required nodes:**
- `Sample Texture 2D` — sample the render texture passed as a Blackboard property `_MainTex`
- `Element Layout UV` or screen UV for sampling position
- Standard math nodes for each effect

### Vignette pass

```
Screen UV → remap to -1..1
    → Length
    → Power(vignetteExp: 1.8)
    → Multiply(_VignetteStrength)
    → Clamp(0,1)
    → 1 - result       ← centre bright, edges dark
    → Multiply(sampled render texture colour)
```

### Scanlines pass

```
Screen UV → Y component
    → Multiply(_LineFrequency: 200.0)   ← more lines than per-panel version
    → Fraction
    → Step(0.5)
    → 1 - result
    → Multiply(_LineIntensity: 0.02)
    → 1 - result
    → Multiply(after-vignette colour)
```

### Chromatic aberration pass

Samples the render texture three times at slightly offset UVs, one per colour channel:

```
UV offset for R channel: UV + float2(_Aberration, 0)
UV offset for G channel: UV (no offset)
UV offset for B channel: UV - float2(_Aberration, 0)

Sample R from offset R UV
Sample G from base UV
Sample B from offset B UV

Combine: float4(R.r, G.g, B.b, 1.0)
```

At `_Aberration: 0.002` this is nearly imperceptible — just enough that the panel reads as a screen with slightly imperfect optics. At `0.01` it reads as visible distortion. Default should be `0.001–0.003`.

---

## Performance on Quest 2

This is an extra full-screen pass on top of all UITK rendering. At Quest 2 resolution:

- Additional render texture read: bandwidth cost, not compute
- Chromatic aberration: 3 texture samples per pixel — noticeable cost
- Scanlines + vignette only: 1 texture sample + cheap arithmetic — acceptable

**Recommendation:** Ship with scanlines + vignette only. Add chromatic aberration as an option that defaults off and only enable it if profiling shows headroom.

---

## C# Setup

```csharp
public class RenderTextureOverlay : MonoBehaviour
{
    [SerializeField] private RenderTexture _uitkRenderTexture;
    [SerializeField] private RawImage      _overlayImage;
    [SerializeField] private Material      _overlayMaterial;

    [Range(0f, 0.4f)]
    [SerializeField] private float _vignetteStrength = 0.20f;

    [Range(0f, 0.06f)]
    [SerializeField] private float _lineIntensity = 0.020f;

    [Range(0f, 0.010f)]
    [SerializeField] private float _aberration = 0.001f;

    private void Start()
    {
        _overlayImage.texture  = _uitkRenderTexture;
        _overlayImage.material = _overlayMaterial;
        ApplyProperties();
    }

    private void ApplyProperties()
    {
        _overlayMaterial.SetFloat("_VignetteStrength", _vignetteStrength);
        _overlayMaterial.SetFloat("_LineIntensity",    _lineIntensity);
        _overlayMaterial.SetFloat("_Aberration",       _aberration);
    }

    // Called by ThemeManager if overlay settings are in UITheme
    public void ApplyTheme(UITheme theme)
    {
        _vignetteStrength = theme.OverlayVignetteStrength;
        _lineIntensity    = theme.OverlayLineIntensity;
        _aberration       = theme.OverlayAberration;
        ApplyProperties();
    }
}
```

---

*This doc covers the optional global overlay pipeline. It is not required for the main UITK visual system described in the primary styling doc.*