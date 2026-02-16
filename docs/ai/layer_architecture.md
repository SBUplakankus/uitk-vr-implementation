# Your 3-Layer Architecture: Controller → Host → View

## What You Actually Have

You have a **3-tier architecture** that's already well-separated:

```
┌─────────────────────────────────────────────────────────┐
│         StartMenuController (MonoBehaviour)             │
│  ┌───────────────────────────────────────────────────┐ │
│  │ SCENE ORCHESTRATION                               │ │
│  │ • Manages multiple panels (StartMenu, Settings)   │ │
│  │ • Handles navigation (LoadScene)                  │ │
│  │ • Application logic (Application.Quit)            │ │
│  │ • Panel visibility (toggle settings)              │ │
│  └───────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
                    │
                    │ controls
                    ▼
┌─────────────────────────────────────────────────────────┐
│         StartMenuPanelHost (MonoBehaviour)              │
│  ┌───────────────────────────────────────────────────┐ │
│  │ PANEL LIFECYCLE                                   │ │
│  │ • Generate() / Dispose()                          │ │
│  │ • Show() / Hide() animations                      │ │
│  │ • Event passthrough (OnPlayClicked, etc.)         │ │
│  │ • UIDocument reference                            │ │
│  └───────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
                    │
                    │ creates & subscribes
                    ▼
┌─────────────────────────────────────────────────────────┐
│         StartMenuPanelView (Pure C#)                    │
│  ┌───────────────────────────────────────────────────┐ │
│  │ UI GENERATION                                     │ │
│  │ • Creates UI via Factory                          │ │
│  │ • Wires button clicks to events                   │ │
│  │ • Exposes events (OnPlayClicked, etc.)            │ │
│  │ • No Unity dependencies                           │ │
│  └───────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

---

## Your Architecture in MVVM Terms

| Your Layer | MVVM Equivalent | Responsibilities |
|------------|-----------------|------------------|
| **Controller** | **Application Controller / Navigator** | Scene orchestration, multi-panel coordination, navigation |
| **Host** | **View Controller** | Panel lifecycle, Show/Hide, event routing |
| **View** | **View** | UI generation, event exposure |
| **(Missing)** | **ViewModel** | Presentation logic (only needed for complex panels) |

---

## The Key Insight

**Your architecture is actually BETTER than basic MVVM!**

You've split the Controller into two layers:
1. **Controller** (scene-level) - handles multiple panels
2. **Host** (panel-level) - handles single panel lifecycle

This is actually a **Hierarchical Controller pattern** and it's cleaner than MVVM for Unity UI because:
- ✅ Scene Controller manages panel coordination
- ✅ Panel Host manages individual panel lifecycle
- ✅ View is pure UI generation
- ✅ ViewModel added only when needed (not shown in your simple menu)

---

## Analysis of Your StartMenuController

### What It Does Right ✅

```csharp
public class StartMenuController : MonoBehaviour
{
    // GOOD: References to managed panels
    [SerializeField] private StartMenuPanelHost startMenuPanelHost;
    [SerializeField] private SettingsPanelHost settingsPanelHost;
    
    // GOOD: Scene-level state
    private bool _settingsActive;
    
    // GOOD: Panel coordination
    private void ToggleSettings()
    {
        _settingsActive = !_settingsActive;
        
        if(!_settingsActive)
            settingsPanelHost.Hide();
        else
            settingsPanelHost.Generate();
    }
    
    // GOOD: Application/navigation logic
    private void HandlePlay() => BootstrapManager.Instance.LoadScene(GameConstants.Hub);
    private void HandleQuit() => Application.Quit();
}
```

**This is exactly what a Scene Controller should do!**

### Minor Improvements

**1. SubscribeEvents() is redundant**

```csharp
// Current:
private void BindButtons()
{
    startMenuPanelHost.SubscribeEvents(); // ← This is called in Generate() already
    startMenuPanelHost.OnPlayClicked += HandlePlay;
    // ...
}

// Better:
private void BindButtons()
{
    // Remove SubscribeEvents() - Host already does this in Generate()
    startMenuPanelHost.OnPlayClicked += HandlePlay;
    startMenuPanelHost.OnSettingsClicked += HandleSettings;
    startMenuPanelHost.OnControlsClicked += HandleControls;
    startMenuPanelHost.OnQuitClicked += HandleQuit;
}
```

**2. Settings panel lifecycle**

```csharp
// Current: Settings regenerates every toggle
private void ToggleSettings()
{
    _settingsActive = !_settingsActive;
    
    if(!_settingsActive)
        settingsPanelHost.Hide();
    else
        settingsPanelHost.Generate(); // ← Recreates UI every time
}

// Better: Generate once, then Show/Hide
private void OnEnable()
{
    startMenuPanelHost.Generate();
    settingsPanelHost.Generate(); // Generate both upfront
    settingsPanelHost.Hide();     // But hide settings initially
    
    BindButtons();
}

private void ToggleSettings()
{
    _settingsActive = !_settingsActive;
    
    if (_settingsActive)
        settingsPanelHost.Show();
    else
        settingsPanelHost.Hide();
}
```

---

## Improved Version (Your Style)

```csharp
using Constants;
using Systems.Core;
using UI.Hosts;
using UnityEngine;

namespace UI.Controllers
{
    /// <summary>
    /// Scene-level controller managing the Start Menu screen.
    /// Coordinates multiple panels and handles navigation.
    /// </summary>
    public class StartMenuController : MonoBehaviour
    {
        #region Fields

        [Header("Panels")]
        [SerializeField] private StartMenuPanelHost _startMenuPanel;
        [SerializeField] private SettingsPanelHost _settingsPanel;
        [SerializeField] private ControlsPanelHost _controlsPanel; // If you add this later

        #endregion

        #region Panel Management

        private void ShowPanel(BasePanelHost panel)
        {
            // Hide all panels
            _startMenuPanel.Hide();
            _settingsPanel.Hide();
            // _controlsPanel.Hide();
            
            // Show requested panel
            panel.Show();
        }

        private void ShowStartMenu() => ShowPanel(_startMenuPanel);
        private void ShowSettings() => ShowPanel(_settingsPanel);
        // private void ShowControls() => ShowPanel(_controlsPanel);

        #endregion

        #region Event Subscription

        private void SubscribeToEvents()
        {
            // Start Menu events
            _startMenuPanel.OnPlayClicked += HandlePlay;
            _startMenuPanel.OnSettingsClicked += HandleSettings;
            _startMenuPanel.OnControlsClicked += HandleControls;
            _startMenuPanel.OnQuitClicked += HandleQuit;
            
            // Settings panel events (when you add them)
            // _settingsPanel.OnBackClicked += ShowStartMenu;
        }

        private void UnsubscribeFromEvents()
        {
            _startMenuPanel.OnPlayClicked -= HandlePlay;
            _startMenuPanel.OnSettingsClicked -= HandleSettings;
            _startMenuPanel.OnControlsClicked -= HandleControls;
            _startMenuPanel.OnQuitClicked -= HandleQuit;
            
            // _settingsPanel.OnBackClicked -= ShowStartMenu;
        }

        #endregion

        #region Event Handlers

        private void HandlePlay()
        {
            BootstrapManager.Instance.LoadScene(GameConstants.Hub);
        }

        private void HandleSettings()
        {
            ShowSettings();
        }

        private void HandleControls()
        {
            // Temporary - replace with ShowControls() when panel exists
            BootstrapManager.Instance.LoadScene(GameConstants.GoblinCampDay);
        }

        private void HandleQuit()
        {
            #if UNITY_EDITOR
            UnityEditor.EditorApplication.isPlaying = false;
            #else
            Application.Quit();
            #endif
        }

        #endregion

        #region Unity Lifecycle

        private void OnEnable()
        {
            // Generate all panels
            _startMenuPanel.Generate();
            _settingsPanel.Generate();
            // _controlsPanel.Generate();
            
            // Show only start menu
            ShowStartMenu();
            
            // Subscribe to events
            SubscribeToEvents();
        }

        private void OnDisable()
        {
            UnsubscribeFromEvents();
        }

        #endregion
    }
}
```

---

## When to Add ViewModel (In Your 3-Layer System)

### Current Structure (No ViewModel Needed)

```
StartMenuController (Scene orchestration)
    ↓
StartMenuPanelHost (Panel lifecycle)
    ↓
StartMenuPanelView (UI generation)
```

**This is perfect for simple panels!**

### When You Need ViewModel

**Example: Settings Panel with State**

```
SettingsController (Scene orchestration - optional)
    ↓
SettingsPanelHost (Panel lifecycle)
    ↓
SettingsViewModel (Presentation logic) ← NEW
    ↓
SettingsPanelView (UI generation + binding)
```

**Example with code:**

```csharp
// SettingsPanelHost.cs
public class SettingsPanelHost : BasePanelHost
{
    [SerializeField] private GameSettings _gameSettings;
    
    private SettingsViewModel _viewModel;
    private SettingsPanelView _view;
    
    public override void Generate()
    {
        Dispose();
        
        // Create ViewModel from settings
        var model = new SettingsModel
        {
            Volume = _gameSettings.Volume,
            Quality = _gameSettings.Quality
        };
        
        _viewModel = new SettingsViewModel(model);
        
        // Subscribe to ViewModel events (game logic)
        _viewModel.OnApplyRequested += ApplySettings;
        _viewModel.OnVolumePreview += PreviewVolume;
        
        // Create View with ViewModel
        _view = new SettingsPanelView(
            uiDocument.rootVisualElement,
            styleSheet,
            _viewModel // Pass ViewModel to View
        );
        
        Show();
    }
    
    // Game logic (Unity-specific)
    private void PreviewVolume(float volume)
    {
        AudioListener.volume = volume;
    }
    
    private void ApplySettings()
    {
        var settings = _viewModel.GetCurrentSettings();
        _gameSettings.Volume = settings.Volume;
        _gameSettings.Quality = settings.Quality;
        _gameSettings.Save();
        
        // Apply to Unity
        QualitySettings.SetQualityLevel(settings.Quality);
    }
    
    protected override void Dispose()
    {
        if (_viewModel != null)
        {
            _viewModel.OnApplyRequested -= ApplySettings;
            _viewModel.OnVolumePreview -= PreviewVolume;
        }
        
        _view?.Dispose();
        _view = null;
        _viewModel = null;
    }
}
```

---

## Complete Architecture Pattern

### For Simple Panels (Your StartMenu)

```
Layer 1: Scene Controller (manages multiple panels, navigation)
    └─ StartMenuController.cs
    
Layer 2: Panel Host (lifecycle, Show/Hide)
    └─ StartMenuPanelHost.cs
    
Layer 3: View (UI generation)
    └─ StartMenuPanelView.cs
```

### For Complex Panels (Settings, Inventory, Character Sheet)

```
Layer 1: Scene Controller (optional - only if coordinating multiple panels)
    └─ SettingsController.cs (optional)
    
Layer 2: Panel Host (lifecycle, game logic)
    └─ SettingsPanelHost.cs
    
Layer 2.5: ViewModel (presentation logic, testable) ← NEW
    └─ SettingsViewModel.cs
    
Layer 3: View (UI generation + binding)
    └─ SettingsPanelView.cs
```

---

## Directory Structure (Recommended)

```
Assets/Scripts/UI/
├── Controllers/
│   ├── StartMenuController.cs       # Scene-level orchestration
│   ├── GameplayHUDController.cs     # Coordinates HUD panels
│   └── InventoryController.cs       # Coordinates inventory UI
│
├── Hosts/                            # Panel lifecycle managers
│   ├── Base/
│   │   └── BasePanelHost.cs
│   ├── StartMenuPanelHost.cs
│   ├── SettingsPanelHost.cs
│   └── InventoryPanelHost.cs
│
├── ViewModels/                       # Presentation logic (add as needed)
│   ├── SettingsViewModel.cs
│   ├── InventoryViewModel.cs
│   └── CharacterStatsViewModel.cs
│
├── Views/                            # UI generation
│   ├── Base/
│   │   └── BasePanelView.cs
│   ├── StartMenuPanelView.cs
│   ├── SettingsPanelView.cs
│   └── InventoryPanelView.cs
│
├── Models/                           # Data structures
│   ├── SettingsModel.cs
│   ├── InventoryModel.cs
│   └── CharacterStatsModel.cs
│
└── Common/
    ├── ObservableProperty.cs
    └── ObservableCollection.cs
```

---

## Decision Matrix: Do I Need ViewModel?

| Panel Type | Complexity | Needs ViewModel? | Example |
|------------|------------|------------------|---------|
| **Simple Menu** | Just buttons → actions | ❌ NO | Your StartMenu |
| **Pause Menu** | Buttons + resume/quit | ❌ NO | Pause overlay |
| **Confirmation Dialog** | Yes/No buttons | ❌ NO | "Are you sure?" |
| **Settings Panel** | Sliders, formatting, validation | ✅ YES | Volume: 75%, Quality names |
| **Inventory** | Filtering, sorting, selection | ✅ YES | Filter by type, sort by value |
| **Character Sheet** | Computed stats, percentage bars | ✅ YES | Attack: 45 (+15), Health: 80% |
| **Form Input** | Validation, error messages | ✅ YES | "Password must be 8+ chars" |
| **Live Stats HUD** | Updating values, health % | ⚠️ MAYBE | If formatting is complex |

**Rule of Thumb:**
```
Does the panel have:
- Data formatting? (percentages, dates, currency)
- Validation logic? (forms, inputs)
- Computed properties? (total price, health %)
- Complex state? (filters, selections)

YES → Add ViewModel
NO  → Keep as Controller → Host → View
```

---

## Your Pattern is Actually Superior for Unity

**Standard MVVM:**
```
ViewModel → View
```
Problems:
- ViewModel doesn't know about panel lifecycle
- View manages its own Show/Hide
- No centralized panel coordination

**Your Pattern:**
```
Controller (scene) → Host (panel lifecycle) → View (UI)
                         ↓
                    ViewModel (when needed)
```
Benefits:
- ✅ Controller manages multiple panels
- ✅ Host manages panel lifecycle + animations
- ✅ View is pure UI generation
- ✅ ViewModel added only when presentation logic exists
- ✅ Clean separation of concerns

---

## Quick Reference: Your Architecture

### Current (Simple Panels)
```csharp
// Layer 1: Scene Controller
public class StartMenuController : MonoBehaviour
{
    [SerializeField] private StartMenuPanelHost _startMenuPanel;
    
    private void OnEnable()
    {
        _startMenuPanel.Generate();
        _startMenuPanel.OnPlayClicked += HandlePlay;
    }
    
    private void HandlePlay()
    {
        BootstrapManager.Instance.LoadScene("Hub");
    }
}

// Layer 2: Panel Host
public class StartMenuPanelHost : BasePanelHost
{
    public event Action OnPlayClicked;
    private StartMenuPanelView _view;
    
    public override void Generate()
    {
        _view = new StartMenuPanelView(uiDocument.rootVisualElement, styleSheet);
        _view.OnPlayClicked += () => OnPlayClicked?.Invoke();
        Show();
    }
}

// Layer 3: View
public class StartMenuPanelView : BasePanelView
{
    public event Action OnPlayClicked;
    
    protected override void GenerateUI(VisualElement root)
    {
        var playButton = UIToolkitFactory.CreateButton("Play");
        playButton.clicked += () => OnPlayClicked?.Invoke();
        // ...
    }
}
```

### With ViewModel (Complex Panels)
```csharp
// Layer 2: Panel Host
public class SettingsPanelHost : BasePanelHost
{
    private SettingsViewModel _viewModel;
    private SettingsPanelView _view;
    
    public override void Generate()
    {
        var model = new SettingsModel { Volume = 0.5f };
        _viewModel = new SettingsViewModel(model);
        _viewModel.OnVolumePreview += volume => AudioListener.volume = volume;
        
        _view = new SettingsPanelView(
            uiDocument.rootVisualElement,
            styleSheet,
            _viewModel // Pass ViewModel
        );
        
        Show();
    }
}

// Layer 2.5: ViewModel
public class SettingsViewModel
{
    public ObservableProperty<float> Volume { get; }
    public ObservableProperty<string> VolumeText { get; }
    public event Action<float> OnVolumePreview;
    
    public void UpdateVolume(float value)
    {
        Volume.Value = value;
        VolumeText.Value = $"Volume: {value:P0}";
        OnVolumePreview?.Invoke(value);
    }
}

// Layer 3: View
public class SettingsPanelView : BasePanelView
{
    private SettingsViewModel _viewModel;
    
    public SettingsPanelView(VisualElement root, StyleSheet styleSheet, SettingsViewModel viewModel)
    {
        _viewModel = viewModel;
        GenerateUI(root);
        BindToViewModel();
    }
    
    private void BindToViewModel()
    {
        var slider = Container.Q<Slider>("VolumeSlider");
        slider.RegisterValueChangedCallback(evt => _viewModel.UpdateVolume(evt.newValue));
        
        var label = Container.Q<Label>("VolumeLabel");
        _viewModel.VolumeText.ValueChanged += text => label.text = text;
    }
}
```

---

## Summary

**Your Current Architecture:**
```
✅ Scene Controller (multi-panel coordination)
✅ Panel Host (lifecycle, Show/Hide, animations)
✅ View (UI generation)
⚠️ ViewModel (add only when needed)
```

**This is BETTER than standard MVVM for Unity because:**
1. Separates scene-level vs panel-level concerns
2. Panel lifecycle is explicit (Host layer)
3. Animations/transitions managed in one place
4. ViewModel only added when there's actual presentation logic

**What to change:**
1. Minor cleanup in StartMenuController (remove redundant SubscribeEvents call)
2. Consider generating panels once, then Show/Hide
3. Add ViewModel layer when panels get complex (Settings, Inventory, Stats)

**You're already doing it right!** Just add ViewModel when you have:
- Formatting logic
- Validation
- Computed properties
- Complex state management

For simple menus like your StartMenu, your current pattern is perfect! 🎯
