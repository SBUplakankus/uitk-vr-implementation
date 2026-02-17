# USS Theming — ScriptableObjects, Variables, Runtime Switching

> **Approach**: Swap full StyleSheet assets for major themes + patch CSS custom
> properties for user tweaks  
> **Designer control**: Colours, typography, spacing, animation  
> **Persistence**: PlayerPrefs  
> **Note**: Code samples are demonstrative — not production-complete

---

## System Overview

```
UITheme (ScriptableObject)
    │  stores all designable values as serialized fields
    │  inspector-editable, no code required for designers
    │
    ▼
UIThemePreset (USS asset, one per theme)
    │  declares :root custom properties matching the ScriptableObject fields
    │  applied as the base stylesheet on the Panel Settings
    │
    ▼
ThemeManager (MonoBehaviour singleton)
    │  swaps StyleSheet assets for theme changes
    │  patches individual custom properties for user tweaks
    │  writes preferences to PlayerPrefs
    │
    ▼
Component USS files (buttons.uss, panels.uss, typography.uss …)
    │  consume var(--token-name) throughout
    │  never hardcode colour or size values
```

The key principle: **USS files never contain hardcoded values**. Every colour, size, duration, and font reference goes through a custom property (`var(--token)`). The theme system controls what those tokens resolve to. Swapping a theme changes everything in one operation.

---

## Design Tokens — the Token Naming Convention

All custom properties follow a three-level naming convention:

```
--category-role-variant

--color-primary            base primary colour
--color-primary-hover      primary on hover
--color-primary-disabled   primary when disabled
--color-surface            panel background
--color-surface-raised     card / elevated surface
--color-on-surface         text sitting on a surface
--color-on-primary         text sitting on a primary button

--size-text-body           body text size
--size-text-label          label text size
--size-text-title-sm       small title
--size-text-title-lg       large title

--space-xs                 4px
--space-sm                 8px
--space-md                 16px
--space-lg                 24px
--space-xl                 40px

--radius-sm                4px
--radius-md                8px
--radius-lg                16px
--radius-pill              999px

--duration-fast            0.1s
--duration-normal          0.25s
--duration-slow            0.4s
--ease-out                 cubic-bezier(0.0, 0.0, 0.2, 1)
```

Naming things this way means a designer can read any USS rule and know exactly what token to change to affect it, without reading C# code.

---

## UITheme ScriptableObject

The ScriptableObject is the designer-facing interface. Every property maps to a CSS custom property token. Field names mirror token names to make the mapping obvious.

```csharp
[CreateAssetMenu(fileName = "NewTheme", menuName = "Training App/UI Theme")]
public class UITheme : ScriptableObject
{
    [Header("Meta")]
    public string ThemeName = "Default";
    public StyleSheet StyleSheetAsset; // the USS file to swap in

    [Header("Colours — Primary")]
    public Color ColorPrimary         = new Color(0.27f, 0.51f, 0.95f);
    public Color ColorPrimaryHover    = new Color(0.38f, 0.60f, 1.00f);
    public Color ColorPrimaryDisabled = new Color(0.27f, 0.51f, 0.95f, 0.4f);
    public Color ColorOnPrimary       = Color.white;

    [Header("Colours — Surface")]
    public Color ColorBackground      = new Color(0.06f, 0.06f, 0.10f);
    public Color ColorSurface         = new Color(0.10f, 0.12f, 0.18f);
    public Color ColorSurfaceRaised   = new Color(0.14f, 0.17f, 0.24f);
    public Color ColorOnSurface       = new Color(0.90f, 0.90f, 0.95f);
    public Color ColorOnSurfaceMuted  = new Color(0.55f, 0.57f, 0.65f);

    [Header("Colours — Semantic")]
    public Color ColorSuccess         = new Color(0.20f, 0.80f, 0.50f);
    public Color ColorWarning         = new Color(0.95f, 0.75f, 0.20f);
    public Color ColorError           = new Color(0.95f, 0.35f, 0.35f);

    [Header("Colours — Glass / Overlay")]
    [Range(0f, 1f)]
    public float GlassFillAlpha       = 0.12f;
    [Range(0f, 1f)]
    public float GlassBorderAlpha     = 0.15f;

    [Header("Typography")]
    public Font  BodyFont;
    public float SizeTextBody         = 16f;
    public float SizeTextLabel        = 13f;
    public float SizeTextCaption      = 11f;
    public float SizeTextTitleSm      = 20f;
    public float SizeTextTitleLg      = 32f;

    [Header("Spacing")]
    public float SpaceXS              = 4f;
    public float SpaceSM              = 8f;
    public float SpaceMD              = 16f;
    public float SpaceLG              = 24f;
    public float SpaceXL              = 40f;

    [Header("Border Radius")]
    public float RadiusSM             = 4f;
    public float RadiusMD             = 8f;
    public float RadiusLG             = 16f;

    [Header("Animation")]
    public float DurationFast         = 0.1f;
    public float DurationNormal       = 0.25f;
    public float DurationSlow         = 0.4f;
}
```

Designers create theme assets via **Assets → Create → Training App → UI Theme**, fill in the inspector, and hand the asset to a developer to register. No code is touched.

---

## ThemeRegistry ScriptableObject

A single asset that lists all available themes. Referenced by `ThemeManager` and the settings dropdown. Designers add themes here by dragging assets in — no code required.

```csharp
[CreateAssetMenu(fileName = "ThemeRegistry", menuName = "Training App/Theme Registry")]
public class ThemeRegistry : ScriptableObject
{
    public UITheme DefaultTheme;
    public List<UITheme> AllThemes = new();

    public UITheme GetByName(string name)
        => AllThemes.Find(t => t.ThemeName == name) ?? DefaultTheme;

    public List<string> GetThemeNames()
        => AllThemes.ConvertAll(t => t.ThemeName);
}
```

---

## USS Structure — Component Files

USS files are split by concern. They never hardcode values — everything goes through tokens.

### base-variables.uss

This file is the fallback. It declares every token with a default value so the UI is never broken if a theme hasn't been applied yet. Loaded first in Panel Settings.

```css
:root {
    /* Colours */
    --color-primary:          rgb(69, 130, 242);
    --color-primary-hover:    rgb(97, 153, 255);
    --color-primary-disabled: rgba(69, 130, 242, 0.4);
    --color-on-primary:       rgb(255, 255, 255);

    --color-background:       rgb(15, 15, 26);
    --color-surface:          rgb(26, 31, 46);
    --color-surface-raised:   rgb(36, 43, 64);
    --color-on-surface:       rgb(230, 230, 242);
    --color-on-surface-muted: rgb(140, 145, 166);

    --color-success:          rgb(51, 204, 128);
    --color-warning:          rgb(242, 191, 51);
    --color-error:            rgb(242, 89, 89);

    /* Typography */
    --size-text-body:         16px;
    --size-text-label:        13px;
    --size-text-caption:      11px;
    --size-text-title-sm:     20px;
    --size-text-title-lg:     32px;

    /* Spacing */
    --space-xs:               4px;
    --space-sm:               8px;
    --space-md:               16px;
    --space-lg:               24px;
    --space-xl:               40px;

    /* Radius */
    --radius-sm:              4px;
    --radius-md:              8px;
    --radius-lg:              16px;
    --radius-pill:            999px;

    /* Animation */
    --duration-fast:          0.1s;
    --duration-normal:        0.25s;
    --duration-slow:          0.4s;
}
```

### theme-dark.uss / theme-high-contrast.uss

One file per theme. Overrides only the tokens that differ from the default. Swapping this file in is the entire theme change operation.

```css
/* theme-high-contrast.uss — only overrides what changes */
:root {
    --color-primary:          rgb(255, 255, 0);
    --color-primary-hover:    rgb(255, 255, 153);
    --color-on-primary:       rgb(0, 0, 0);

    --color-background:       rgb(0, 0, 0);
    --color-surface:          rgb(20, 20, 20);
    --color-on-surface:       rgb(255, 255, 255);
    --color-on-surface-muted: rgb(200, 200, 200);
}
```

### buttons.uss

```css
.btn-primary {
    background-color: var(--color-primary);
    color:            var(--color-on-primary);
    border-radius:    var(--radius-md);
    padding:          var(--space-sm) var(--space-md);
    font-size:        var(--size-text-label);

    transition-property:   background-color, scale;
    transition-duration:   var(--duration-fast), var(--duration-fast);
    transition-timing-function: ease-out;
}

.btn-primary:hover {
    background-color: var(--color-primary-hover);
    scale:            1.03 1.03;
}

.btn-primary:active {
    scale: 0.97 0.97;
}

.btn-primary:disabled {
    background-color: var(--color-primary-disabled);
    opacity:          0.6;
}

.btn-secondary {
    background-color: transparent;
    color:            var(--color-primary);
    border-color:     var(--color-primary);
    border-width:     1px;
    border-radius:    var(--radius-md);
    padding:          var(--space-sm) var(--space-md);
    font-size:        var(--size-text-label);

    transition-property: border-color, color;
    transition-duration: var(--duration-fast);
}

.btn-secondary:hover {
    border-color: var(--color-primary-hover);
    color:        var(--color-primary-hover);
}

.btn-ghost {
    background-color: transparent;
    color:            var(--color-on-surface-muted);
    border-width:     0;
    padding:          var(--space-sm);
    border-radius:    var(--radius-sm);

    transition-property: color, background-color;
    transition-duration: var(--duration-fast);
}

.btn-ghost:hover {
    color:            var(--color-on-surface);
    background-color: rgba(255, 255, 255, 0.06);
}

.btn-tab {
    background-color: transparent;
    color:            var(--color-on-surface-muted);
    border-width:     0 0 2px 0;
    border-color:     transparent;
    border-radius:    0;
    padding:          var(--space-sm) var(--space-md);
    font-size:        var(--size-text-label);

    transition-property: color, border-color;
    transition-duration: var(--duration-fast);
}

.btn-tab--active {
    color:        var(--color-primary);
    border-color: var(--color-primary);
}
```

### panels.uss

```css
.panel-root {
    flex-grow:        1;
    background-color: var(--color-background);
    padding:          var(--space-lg);
}

.panel-header {
    flex-direction:  row;
    justify-content: space-between;
    align-items:     center;
    margin-bottom:   var(--space-lg);
    padding-bottom:  var(--space-md);
    border-bottom-width: 1px;
    border-color:    rgba(255, 255, 255, 0.08);
}

.panel-body {
    flex-grow: 1;
    gap:       var(--space-md);
}

.panel-footer {
    margin-top:      var(--space-lg);
    padding-top:     var(--space-md);
    border-top-width: 1px;
    border-color:    rgba(255, 255, 255, 0.08);
}

.glass-panel {
    background-color: rgba(255, 255, 255, 0.08);
    border-width:     1px;
    border-color:     rgba(255, 255, 255, 0.12);
    border-radius:    var(--radius-lg);
    padding:          var(--space-lg);
}

.card {
    background-color: var(--color-surface-raised);
    border-radius:    var(--radius-md);
    padding:          var(--space-md);
    margin-bottom:    var(--space-sm);

    transition-property: background-color;
    transition-duration: var(--duration-fast);
}

.card:hover {
    background-color: rgba(255, 255, 255, 0.08);
}

.card--locked {
    opacity: 0.45;
}

.divider {
    height:           1px;
    background-color: rgba(255, 255, 255, 0.08);
    margin:           var(--space-md) 0;
}

.progress-track {
    height:           6px;
    background-color: var(--color-surface-raised);
    border-radius:    var(--radius-pill);
    overflow:         hidden;
}

.progress-fill {
    height:           100%;
    background-color: var(--color-primary);
    border-radius:    var(--radius-pill);

    transition-property: width;
    transition-duration: var(--duration-slow);
}
```

### typography.uss

```css
.title-large {
    font-size:    var(--size-text-title-lg);
    color:        var(--color-on-surface);
    -unity-font-style: bold;
    letter-spacing: 0.5px;
}

.title-medium {
    font-size:    var(--size-text-title-sm);
    color:        var(--color-on-surface);
    -unity-font-style: bold;
}

.body-text {
    font-size: var(--size-text-body);
    color:     var(--color-on-surface);
}

.label-text {
    font-size: var(--size-text-label);
    color:     var(--color-on-surface);
    -unity-font-style: bold;
}

.caption-text {
    font-size: var(--size-text-caption);
    color:     var(--color-on-surface-muted);
}

.input-label {
    font-size:     var(--size-text-caption);
    color:         var(--color-on-surface-muted);
    margin-bottom: var(--space-xs);
    -unity-font-style: bold;
    letter-spacing: 0.5px;
    text-transform: uppercase; /* Note: check Unity 6.3 text-transform support */
}
```

### inputs.uss

```css
.input-group {
    gap:           var(--space-xs);
    margin-bottom: var(--space-md);
}

.input-field {
    background-color: var(--color-surface-raised);
    border-color:     rgba(255, 255, 255, 0.12);
    border-width:     1px;
    border-radius:    var(--radius-md);
    color:            var(--color-on-surface);
    padding:          var(--space-sm) var(--space-md);
    font-size:        var(--size-text-body);

    transition-property: border-color;
    transition-duration: var(--duration-fast);
}

.input-field:focus {
    border-color: var(--color-primary);
}

.input-field--error {
    border-color: var(--color-error);
}

/* Slider track */
.input-field .unity-base-slider__tracker {
    background-color: var(--color-surface-raised);
    border-radius:    var(--radius-pill);
    height:           4px;
}

/* Slider fill */
.input-field .unity-base-slider__dragger {
    background-color: var(--color-primary);
    border-radius:    var(--radius-pill);
    width:            18px;
    height:           18px;
    margin-top:       -7px;
    border-width:     0;
}

/* Tab bar */
.tab-bar {
    flex-direction: row;
    border-bottom-width: 1px;
    border-color:   rgba(255, 255, 255, 0.08);
    margin-bottom:  var(--space-md);
}
```

### animations.uss

```css
/* Panel entry — add class on Show(), remove on Hide() */
.fade-in {
    opacity: 1;
    transition-property: opacity;
    transition-duration: var(--duration-normal);
}

.hidden {
    display: none;
}

/* Step list items */
.step-item {
    flex-direction: row;
    align-items:    center;
    padding:        var(--space-sm) var(--space-md);
    border-radius:  var(--radius-sm);
    gap:            var(--space-sm);
    color:          var(--color-on-surface-muted);

    transition-property: color, background-color;
    transition-duration: var(--duration-fast);
}

.step-item--active {
    color:            var(--color-on-surface);
    background-color: rgba(255, 255, 255, 0.05);
}

.step-item--complete {
    color: var(--color-success);
}

/* Reduced motion support — add to root element when user opts in */
.reduced-motion * {
    transition-duration: 0s !important;
}
```

---

## USS Load Order in Panel Settings

Order matters — later files override earlier ones for the same selector.

```
Panel Settings → Style Sheets (loaded in this order):
  1. base-variables.uss    ← token defaults, never removed
  2. theme-dark.uss        ← swapped for theme changes
  3. typography.uss        ← consumes tokens
  4. panels.uss
  5. buttons.uss
  6. inputs.uss
  7. animations.uss
```

Only slot 2 changes at runtime. Everything else stays loaded.

---

## ThemeManager

Handles stylesheet swapping and individual token patching. The patching path is used for user tweaks (e.g. a custom accent colour) that don't warrant a full theme swap.

```csharp
public class ThemeManager : MonoBehaviour
{
    public static ThemeManager Instance { get; private set; }

    [SerializeField] private ThemeRegistry _registry;
    [SerializeField] private PanelSettings _panelSettings; // drag in from inspector

    private UITheme   _activeTheme;
    private StyleSheet _activeThemeSheet;

    // ── Lifecycle ──────────────────────────────────────────────────────────

    private void Awake()
    {
        if (Instance != null) { Destroy(gameObject); return; }
        Instance = this;
        DontDestroyOnLoad(gameObject);
    }

    private void Start()
    {
        var savedTheme = PlayerPrefs.GetString("ThemeName", _registry.DefaultTheme.ThemeName);
        Apply(_registry.GetByName(savedTheme));
    }

    // ── Public API ─────────────────────────────────────────────────────────

    // Called by SettingsHost when a theme is selected
    public void Apply(string themeName)
        => Apply(_registry.GetByName(themeName));

    public void Apply(UITheme theme)
    {
        if (theme == null) return;

        SwapThemeStyleSheet(theme);
        PatchTokensFromTheme(theme);

        _activeTheme = theme;
        PlayerPrefs.SetString("ThemeName", theme.ThemeName);
    }

    // Patch a single colour token — for user accent colour customisation
    public void PatchColor(string token, Color color)
    {
        var root = GetPanelRoot();
        if (root == null) return;
        root.style.SetCustomProperty(token, color);
    }

    public void SetHighContrast(bool on)
    {
        var root = GetPanelRoot();
        if (root == null) return;
        root.EnableInClassList("high-contrast", on);
        PlayerPrefs.SetInt("HighContrast", on ? 1 : 0);
    }

    public void SetReducedMotion(bool on)
    {
        var root = GetPanelRoot();
        if (root == null) return;
        root.EnableInClassList("reduced-motion", on);
        PlayerPrefs.SetInt("ReducedMotion", on ? 1 : 0);
    }

    // ── Internal ──────────────────────────────────────────────────────────

    private void SwapThemeStyleSheet(UITheme theme)
    {
        if (theme.StyleSheetAsset == null) return;

        var root = GetPanelRoot();
        if (root == null) return;

        // Remove previous theme sheet if present
        if (_activeThemeSheet != null && root.styleSheets.Contains(_activeThemeSheet))
            root.styleSheets.Remove(_activeThemeSheet);

        // Insert theme sheet at index 1 (after base-variables, before components)
        root.styleSheets.Insert(1, theme.StyleSheetAsset);
        _activeThemeSheet = theme.StyleSheetAsset;
    }

    private void PatchTokensFromTheme(UITheme theme)
    {
        var root = GetPanelRoot();
        if (root == null) return;

        // Colours
        root.style.SetCustomProperty("--color-primary",          theme.ColorPrimary);
        root.style.SetCustomProperty("--color-primary-hover",    theme.ColorPrimaryHover);
        root.style.SetCustomProperty("--color-primary-disabled", theme.ColorPrimaryDisabled);
        root.style.SetCustomProperty("--color-on-primary",       theme.ColorOnPrimary);
        root.style.SetCustomProperty("--color-background",       theme.ColorBackground);
        root.style.SetCustomProperty("--color-surface",          theme.ColorSurface);
        root.style.SetCustomProperty("--color-surface-raised",   theme.ColorSurfaceRaised);
        root.style.SetCustomProperty("--color-on-surface",       theme.ColorOnSurface);
        root.style.SetCustomProperty("--color-on-surface-muted", theme.ColorOnSurfaceMuted);
        root.style.SetCustomProperty("--color-success",          theme.ColorSuccess);
        root.style.SetCustomProperty("--color-warning",          theme.ColorWarning);
        root.style.SetCustomProperty("--color-error",            theme.ColorError);

        // Spacing
        root.style.SetCustomProperty("--space-xs", new Length(theme.SpaceXS, LengthUnit.Pixel));
        root.style.SetCustomProperty("--space-sm", new Length(theme.SpaceSM, LengthUnit.Pixel));
        root.style.SetCustomProperty("--space-md", new Length(theme.SpaceMD, LengthUnit.Pixel));
        root.style.SetCustomProperty("--space-lg", new Length(theme.SpaceLG, LengthUnit.Pixel));
        root.style.SetCustomProperty("--space-xl", new Length(theme.SpaceXL, LengthUnit.Pixel));

        // Typography
        root.style.SetCustomProperty("--size-text-body",     new Length(theme.SizeTextBody,     LengthUnit.Pixel));
        root.style.SetCustomProperty("--size-text-label",    new Length(theme.SizeTextLabel,    LengthUnit.Pixel));
        root.style.SetCustomProperty("--size-text-caption",  new Length(theme.SizeTextCaption,  LengthUnit.Pixel));
        root.style.SetCustomProperty("--size-text-title-sm", new Length(theme.SizeTextTitleSm,  LengthUnit.Pixel));
        root.style.SetCustomProperty("--size-text-title-lg", new Length(theme.SizeTextTitleLg,  LengthUnit.Pixel));

        // Radius
        root.style.SetCustomProperty("--radius-sm", new Length(theme.RadiusSM, LengthUnit.Pixel));
        root.style.SetCustomProperty("--radius-md", new Length(theme.RadiusMD, LengthUnit.Pixel));
        root.style.SetCustomProperty("--radius-lg", new Length(theme.RadiusLG, LengthUnit.Pixel));

        // Animation
        root.style.SetCustomProperty("--duration-fast",   new TimeValue(theme.DurationFast,   TimeUnit.Second));
        root.style.SetCustomProperty("--duration-normal", new TimeValue(theme.DurationNormal, TimeUnit.Second));
        root.style.SetCustomProperty("--duration-slow",   new TimeValue(theme.DurationSlow,   TimeUnit.Second));
    }

    private VisualElement GetPanelRoot()
        => _panelSettings?.visualTree?.rootVisualElement;
}
```

### SetCustomProperty Extension

Unity 6.3 doesn't yet expose a fully clean public API for setting arbitrary custom properties on a `IStyle`. This extension method handles the current workaround using the inline style override path. Verify against your Unity version — this API surface is actively evolving.

```csharp
public static class StyleExtensions
{
    public static void SetCustomProperty(this IStyle style, string property, Color value)
    {
        // Unity 6.x: custom property patching on VisualElement inline styles
        // The exact API depends on Unity version — check release notes for your build.
        // This is the pattern used in Unity's own samples as of 6.3.
        var colorValue = new StyleColor(value);
        style.SetCustomProperty(property, colorValue.ToString());
    }

    public static void SetCustomProperty(this IStyle style, string property, Length value)
    {
        style.SetCustomProperty(property, value.ToString());
    }

    public static void SetCustomProperty(this IStyle style, string property, TimeValue value)
    {
        style.SetCustomProperty(property, $"{value.value}s");
    }
}
```

---

## UserPreferences ScriptableObject

Separate from theme presets — this holds the user's personal overrides and accessibility flags. Saved to `PlayerPrefs` as JSON.

```csharp
[CreateAssetMenu(fileName = "UserPreferences", menuName = "Training App/User Preferences")]
public class UserPreferences : ScriptableObject
{
    [Header("Theme")]
    public string SelectedTheme   = "Default";

    [Header("Accessibility")]
    public bool   HighContrast    = false;
    public bool   ReducedMotion   = false;

    [Header("Custom Accent")]
    public bool   UseCustomAccent = false;
    public Color  CustomAccent    = new Color(0.27f, 0.51f, 0.95f);

    private const string PrefsKey = "UserPreferences";

    public void Save()
    {
        PlayerPrefs.SetString(PrefsKey, JsonUtility.ToJson(this));
        PlayerPrefs.Save();
    }

    public void Load()
    {
        if (PlayerPrefs.HasKey(PrefsKey))
            JsonUtility.FromJsonOverwrite(PlayerPrefs.GetString(PrefsKey), this);
    }

    public void Apply(ThemeManager themeManager)
    {
        themeManager.Apply(SelectedTheme);
        themeManager.SetHighContrast(HighContrast);
        themeManager.SetReducedMotion(ReducedMotion);

        if (UseCustomAccent)
            themeManager.PatchColor("--color-primary", CustomAccent);
    }
}
```

---

## Wiring It into the Settings Host

The Settings Host now connects the View's events to both the theme system and the preferences object.

```csharp
public class SettingsPanelHost : BasePanelHost, IPokeTarget
{
    [SerializeField] private UserPreferences _prefs;

    public event Action OnBackRequested;

    private SettingsPanelView _view;

    public override void Generate()
    {
        base.Generate();
        _view = new SettingsPanelView(Root);

        _view.OnBackClicked         += () => OnBackRequested?.Invoke();

        _view.OnThemeChanged        += name =>
        {
            _prefs.SelectedTheme = name;
            _prefs.Save();
            ThemeManager.Instance.Apply(name);
        };

        _view.OnHighContrastChanged += on =>
        {
            _prefs.HighContrast = on;
            _prefs.Save();
            ThemeManager.Instance.SetHighContrast(on);
        };

        _view.OnMasterVolumeChanged += value =>
        {
            _prefs.MasterVolume = value;  // add to UserPreferences if needed
            AudioManager.SetMaster(value);
        };

        Hide();
    }

    public override void Show()
    {
        _view?.SetValues(
            AudioManager.Master,
            AudioManager.Music,
            AudioManager.SFX,
            _prefs.SelectedTheme,
            _prefs.HighContrast
        );
        base.Show();
    }

    public void OnPoked(string buttonId) => _view?.HandlePoke(buttonId);
}
```

---

## Designer Workflow

With this setup, a designer can:

1. **Create a new theme**: Assets → Create → Training App → UI Theme, fill in the inspector colour pickers and sliders, drag the corresponding USS file into the StyleSheetAsset field.

2. **Register the theme**: Drag the new asset into the `ThemeRegistry.AllThemes` list. It will appear automatically in the settings dropdown.

3. **Adjust spacing or type scale**: Open any theme asset, change the spacing or font size values, hit play. No code touched.

4. **Test in editor**: ThemeManager.Apply() can be called from a custom inspector button (or just hit play with the desired theme asset set as DefaultTheme).

The USS files themselves are still designer-editable for fine-grained control — but because they only contain `var(--token)` references, changing a value in the token system propagates everywhere without hunting through component files.

---

## File Structure

```
Assets/
├── Settings/
│   ├── ThemeRegistry.asset
│   ├── UserPreferences.asset
│   └── Themes/
│       ├── Theme_Default.asset
│       ├── Theme_Dark.asset
│       ├── Theme_Light.asset
│       └── Theme_HighContrast.asset
│
└── UI/
    ├── Styles/
    │   ├── base-variables.uss    ← token defaults
    │   ├── theme-dark.uss        ← theme overrides (one per theme)
    │   ├── theme-high-contrast.uss
    │   ├── typography.uss
    │   ├── panels.uss
    │   ├── buttons.uss
    │   ├── inputs.uss
    │   └── animations.uss
    │
    └── Scripts/
        ├── UITheme.cs
        ├── ThemeRegistry.cs
        ├── UserPreferences.cs
        ├── ThemeManager.cs
        └── StyleExtensions.cs
```
