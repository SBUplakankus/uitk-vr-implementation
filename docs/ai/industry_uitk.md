# Industry-Standard UI Toolkit Practices: Code-First Architecture

> **Focus**: Enterprise-grade UI Toolkit patterns for team environments  
> **Audience**: Teams transitioning from code-generated UI to scalable architectures  
> **Approach**: Code-first with separation of concerns and data binding

## Table of Contents

1. [Architecture Patterns Overview](#1-architecture-patterns-overview)
2. [Code-First UI Generation](#2-code-first-ui-generation)
3. [Separation of Concerns](#3-separation-of-concerns)
4. [Data Binding Strategies](#4-data-binding-strategies)
5. [Factory Pattern Implementation](#5-factory-pattern-implementation)
6. [View-Controller Patterns](#6-view-controller-patterns)
7. [Runtime Customization](#7-runtime-customization)
8. [Team Collaboration Patterns](#8-team-collaboration-patterns)
9. [Testing Strategies](#9-testing-strategies)
10. [Performance & Scalability](#10-performance--scalability)
11. [Production Patterns](#11-production-patterns)

---

## 1. Architecture Patterns Overview

### 1.1 Common UI Architecture Patterns

**Model-View-Controller (MVC)**
```
Model (Data) ←→ Controller (Logic) ←→ View (UI)
     ↓                                    ↑
     └────────── Events ──────────────────┘
```

**Model-View-Presenter (MVP)**
```
Model (Data) ←→ Presenter (Logic) ←→ View (Interface)
```

**Model-View-ViewModel (MVVM)**
```
Model (Data) ←→ ViewModel (Logic + State) ←→ View (UI + Binding)
```

### 1.2 UI Toolkit Best Pattern Choice

**Recommendation: MVVM with UI Toolkit**

**Why MVVM for UI Toolkit:**
- Unity 6 native data binding support
- Clear separation of logic and presentation
- Easy to unit test ViewModels
- Scales well for large teams
- Designer-friendly (artists work on UXML/USS)
- Developers work on ViewModels and Models

**Architecture Diagram:**
```
┌─────────────────────────────────────────────────┐
│                   Application                    │
└───────────────────┬─────────────────────────────┘
                    │
        ┌───────────┴───────────┐
        │                       │
┌───────▼────────┐    ┌────────▼────────┐
│  UI Services   │    │  Game Services  │
│  - UIRouter    │    │  - GameManager  │
│  - UIFactory   │    │  - AudioManager │
└───────┬────────┘    └────────┬────────┘
        │                      │
        └──────────┬───────────┘
                   │
         ┌─────────▼──────────┐
         │   View Controllers │
         │  (Screen Logic)    │
         └─────────┬──────────┘
                   │
    ┌──────────────┼──────────────┐
    │              │              │
┌───▼────┐   ┌────▼─────┐   ┌───▼────┐
│ View   │   │ ViewModel│   │ Model  │
│ (UXML  │◄──┤ (Binding │◄──┤ (Data) │
│  USS)  │   │  Logic)  │   │        │
└────────┘   └──────────┘   └────────┘
```

### 1.3 Layer Responsibilities

**Model Layer**
- Pure data structures
- Business logic
- No Unity dependencies
- Easily unit testable

**ViewModel Layer**
- Presentation logic
- Command handling
- Property change notifications
- Data transformation for view

**View Layer**
- UXML structure
- USS styling
- Minimal C# (wiring only)
- Event routing to ViewModel

**Controller Layer**
- Orchestrates Views and ViewModels
- Lifecycle management
- Navigation
- Dependency injection

---

## 2. Code-First UI Generation

### 2.1 Pure Code UI Generation

**Example: Building UI Programmatically**

```csharp
using UnityEngine;
using UnityEngine.UIElements;

public static class UIFactory
{
    // Create a styled button
    public static Button CreateButton(string text, string[] classes = null)
    {
        var button = new Button { text = text };
        
        // Apply base styling
        button.style.minHeight = 50;
        button.style.minWidth = 200;
        button.style.marginBottom = 10;
        button.style.borderRadius = new Length(8, LengthUnit.Pixel);
        
        // Apply classes if provided
        if (classes != null)
        {
            foreach (var className in classes)
            {
                button.AddToClassList(className);
            }
        }
        
        return button;
    }
    
    // Create a labeled input field
    public static VisualElement CreateLabeledField(string label, VisualElement field)
    {
        var container = new VisualElement();
        container.AddToClassList("labeled-field");
        
        var labelElement = new Label(label);
        labelElement.AddToClassList("field-label");
        
        container.Add(labelElement);
        container.Add(field);
        
        return container;
    }
    
    // Create a panel with header
    public static VisualElement CreatePanel(string title, VisualElement content)
    {
        var panel = new VisualElement();
        panel.AddToClassList("panel");
        
        // Header
        var header = new VisualElement();
        header.AddToClassList("panel-header");
        
        var titleLabel = new Label(title);
        titleLabel.AddToClassList("panel-title");
        header.Add(titleLabel);
        
        // Content container
        var contentContainer = new VisualElement();
        contentContainer.AddToClassList("panel-content");
        contentContainer.Add(content);
        
        panel.Add(header);
        panel.Add(contentContainer);
        
        return panel;
    }
}
```

**Usage Example:**
```csharp
public class MenuView : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;
    
    private void OnEnable()
    {
        var root = _uiDocument.rootVisualElement;
        root.Clear(); // Clear existing content
        
        // Build UI programmatically
        var container = new VisualElement();
        container.AddToClassList("menu-container");
        
        // Create buttons
        var playButton = UIFactory.CreateButton("Play", new[] { "primary-button" });
        var settingsButton = UIFactory.CreateButton("Settings", new[] { "secondary-button" });
        var quitButton = UIFactory.CreateButton("Quit", new[] { "secondary-button" });
        
        // Add callbacks
        playButton.clicked += OnPlayClicked;
        settingsButton.clicked += OnSettingsClicked;
        quitButton.clicked += OnQuitClicked;
        
        // Add to container
        container.Add(playButton);
        container.Add(settingsButton);
        container.Add(quitButton);
        
        // Add to root
        root.Add(container);
    }
    
    private void OnPlayClicked() { /* ... */ }
    private void OnSettingsClicked() { /* ... */ }
    private void OnQuitClicked() { /* ... */ }
}
```

### 2.2 Template-Based Generation

**Hybrid Approach: UXML Templates + Code Generation**

```csharp
using UnityEngine;
using UnityEngine.UIElements;

public class InventoryItemTemplate
{
    // UXML template (can be embedded or loaded from file)
    private static VisualTreeAsset _template;
    
    public static VisualElement Create(InventoryItemData data)
    {
        // Load template if not cached
        if (_template == null)
        {
            _template = Resources.Load<VisualTreeAsset>("UI/Templates/InventoryItem");
        }
        
        // Clone template
        var instance = _template.CloneTree();
        
        // Populate with data
        var icon = instance.Q<VisualElement>("Icon");
        var nameLabel = instance.Q<Label>("ItemName");
        var quantityLabel = instance.Q<Label>("Quantity");
        
        icon.style.backgroundImage = new StyleBackground(data.Icon);
        nameLabel.text = data.Name;
        quantityLabel.text = data.Quantity.ToString();
        
        // Add interaction
        instance.RegisterCallback<ClickEvent>(evt => OnItemClicked(data));
        
        return instance;
    }
    
    private static void OnItemClicked(InventoryItemData data)
    {
        Debug.Log($"Clicked item: {data.Name}");
    }
}
```

**InventoryItem.uxml Template:**
```xml
<ui:UXML xmlns:ui="UnityEngine.UIElements">
    <ui:VisualElement class="inventory-item">
        <ui:VisualElement name="Icon" class="item-icon" />
        <ui:VisualElement class="item-info">
            <ui:Label name="ItemName" class="item-name" />
            <ui:Label name="Quantity" class="item-quantity" />
        </ui:VisualElement>
    </ui:VisualElement>
</ui:UXML>
```

### 2.3 Builder Pattern for Complex UI

```csharp
using UnityEngine.UIElements;

public class FormBuilder
{
    private VisualElement _form;
    private VisualElement _currentSection;
    
    public FormBuilder()
    {
        _form = new VisualElement();
        _form.AddToClassList("form");
    }
    
    public FormBuilder AddSection(string title)
    {
        _currentSection = new VisualElement();
        _currentSection.AddToClassList("form-section");
        
        if (!string.IsNullOrEmpty(title))
        {
            var sectionTitle = new Label(title);
            sectionTitle.AddToClassList("section-title");
            _currentSection.Add(sectionTitle);
        }
        
        _form.Add(_currentSection);
        return this;
    }
    
    public FormBuilder AddTextField(string label, string bindingPath)
    {
        var field = new TextField(label);
        field.bindingPath = bindingPath;
        field.AddToClassList("form-field");
        
        _currentSection.Add(field);
        return this;
    }
    
    public FormBuilder AddSlider(string label, float min, float max, string bindingPath)
    {
        var container = new VisualElement();
        container.AddToClassList("form-field");
        
        var labelElement = new Label(label);
        labelElement.AddToClassList("field-label");
        
        var slider = new Slider(min, max);
        slider.bindingPath = bindingPath;
        slider.AddToClassList("slider-field");
        
        container.Add(labelElement);
        container.Add(slider);
        
        _currentSection.Add(container);
        return this;
    }
    
    public FormBuilder AddToggle(string label, string bindingPath)
    {
        var toggle = new Toggle(label);
        toggle.bindingPath = bindingPath;
        toggle.AddToClassList("form-field");
        
        _currentSection.Add(toggle);
        return this;
    }
    
    public FormBuilder AddDropdown(string label, System.Collections.Generic.List<string> choices, string bindingPath)
    {
        var container = new VisualElement();
        container.AddToClassList("form-field");
        
        var labelElement = new Label(label);
        labelElement.AddToClassList("field-label");
        
        var dropdown = new DropdownField(choices, 0);
        dropdown.bindingPath = bindingPath;
        dropdown.AddToClassList("dropdown-field");
        
        container.Add(labelElement);
        container.Add(dropdown);
        
        _currentSection.Add(container);
        return this;
    }
    
    public FormBuilder AddButton(string text, System.Action onClick)
    {
        var button = new Button(onClick) { text = text };
        button.AddToClassList("form-button");
        
        _currentSection.Add(button);
        return this;
    }
    
    public VisualElement Build()
    {
        return _form;
    }
}

// Usage:
var settingsForm = new FormBuilder()
    .AddSection("Video Settings")
    .AddDropdown("Resolution", new List<string> { "1920x1080", "2560x1440", "3840x2160" }, "resolution")
    .AddToggle("Fullscreen", "isFullscreen")
    .AddSlider("Brightness", 0f, 1f, "brightness")
    .AddSection("Audio Settings")
    .AddSlider("Master Volume", 0f, 1f, "masterVolume")
    .AddSlider("Music Volume", 0f, 1f, "musicVolume")
    .AddSlider("SFX Volume", 0f, 1f, "sfxVolume")
    .AddSection("")
    .AddButton("Apply", OnApplySettings)
    .AddButton("Reset", OnResetSettings)
    .Build();
```

---

## 3. Separation of Concerns

### 3.1 Layer Structure

**Project Organization:**
```
Assets/
├── Scripts/
│   ├── Core/
│   │   ├── Models/           # Data models
│   │   ├── Services/         # Business logic services
│   │   └── Utilities/        # Helpers, extensions
│   │
│   ├── UI/
│   │   ├── ViewModels/       # Presentation logic
│   │   ├── Views/            # View scripts (minimal)
│   │   ├── Controllers/      # Screen controllers
│   │   ├── Components/       # Reusable UI components
│   │   ├── Factories/        # UI element factories
│   │   └── Bindings/         # Custom binding adapters
│   │
│   └── Application/
│       ├── Bootstrap/        # App initialization
│       └── Configuration/    # Settings, config
│
└── UI/
    ├── Documents/            # UIDocument prefabs
    ├── Templates/            # UXML templates
    ├── Styles/               # USS stylesheets
    └── Assets/               # Icons, sprites
```

### 3.2 Interface-Based Design

**View Interface:**
```csharp
using UnityEngine.UIElements;

// View interface defines what the view can do
public interface IView
{
    VisualElement Root { get; }
    void Show();
    void Hide();
    void Dispose();
}

// Generic view interface with data type
public interface IView<TData> : IView
{
    void Initialize(TData data);
}

// Example: Main Menu View
public interface IMainMenuView : IView
{
    event System.Action PlayClicked;
    event System.Action SettingsClicked;
    event System.Action QuitClicked;
}
```

**View Implementation:**
```csharp
using UnityEngine;
using UnityEngine.UIElements;

public class MainMenuView : MonoBehaviour, IMainMenuView
{
    [SerializeField] private UIDocument _uiDocument;
    
    private VisualElement _root;
    private Button _playButton;
    private Button _settingsButton;
    private Button _quitButton;
    
    // Events
    public event System.Action PlayClicked;
    public event System.Action SettingsClicked;
    public event System.Action QuitClicked;
    
    // Properties
    public VisualElement Root => _root;
    
    private void Awake()
    {
        _root = _uiDocument.rootVisualElement;
        
        // Query elements
        _playButton = _root.Q<Button>("PlayButton");
        _settingsButton = _root.Q<Button>("SettingsButton");
        _quitButton = _root.Q<Button>("QuitButton");
        
        // Wire up events
        _playButton.clicked += () => PlayClicked?.Invoke();
        _settingsButton.clicked += () => SettingsClicked?.Invoke();
        _quitButton.clicked += () => QuitClicked?.Invoke();
    }
    
    public void Show()
    {
        _root.style.display = DisplayStyle.Flex;
        _root.style.opacity = 1;
    }
    
    public void Hide()
    {
        _root.style.display = DisplayStyle.None;
    }
    
    public void Dispose()
    {
        // Clean up event subscriptions
        PlayClicked = null;
        SettingsClicked = null;
        QuitClicked = null;
    }
}
```

**Controller:**
```csharp
using UnityEngine;

public class MainMenuController : MonoBehaviour
{
    [SerializeField] private MainMenuView _view;
    
    // Injected dependencies
    private ISceneLoader _sceneLoader;
    private IAudioService _audioService;
    
    public void Initialize(ISceneLoader sceneLoader, IAudioService audioService)
    {
        _sceneLoader = sceneLoader;
        _audioService = audioService;
    }
    
    private void OnEnable()
    {
        // Subscribe to view events
        _view.PlayClicked += OnPlayClicked;
        _view.SettingsClicked += OnSettingsClicked;
        _view.QuitClicked += OnQuitClicked;
        
        _view.Show();
    }
    
    private void OnDisable()
    {
        // Unsubscribe from view events
        _view.PlayClicked -= OnPlayClicked;
        _view.SettingsClicked -= OnSettingsClicked;
        _view.QuitClicked -= OnQuitClicked;
    }
    
    private void OnPlayClicked()
    {
        _audioService.PlaySFX("ButtonClick");
        _sceneLoader.LoadScene("GameScene");
    }
    
    private void OnSettingsClicked()
    {
        _audioService.PlaySFX("ButtonClick");
        // Open settings panel
    }
    
    private void OnQuitClicked()
    {
        _audioService.PlaySFX("ButtonClick");
        Application.Quit();
    }
}
```

### 3.3 Command Pattern for Actions

**Command Interface:**
```csharp
public interface ICommand
{
    void Execute();
    void Undo();
    bool CanExecute();
}

public interface ICommand<T> : ICommand
{
    void Execute(T parameter);
}
```

**Command Implementation:**
```csharp
using UnityEngine;

public class LoadSceneCommand : ICommand<string>
{
    private readonly ISceneLoader _sceneLoader;
    private readonly IAudioService _audioService;
    
    public LoadSceneCommand(ISceneLoader sceneLoader, IAudioService audioService)
    {
        _sceneLoader = sceneLoader;
        _audioService = audioService;
    }
    
    public bool CanExecute()
    {
        return !_sceneLoader.IsLoading;
    }
    
    public void Execute(string sceneName)
    {
        if (!CanExecute()) return;
        
        _audioService.PlaySFX("SceneTransition");
        _sceneLoader.LoadScene(sceneName);
    }
    
    public void Execute()
    {
        // Default implementation
    }
    
    public void Undo()
    {
        // Not applicable for scene loading
    }
}
```

**Using Commands in Views:**
```csharp
public class MainMenuViewModel
{
    private readonly ICommand<string> _loadSceneCommand;
    
    public MainMenuViewModel(ICommand<string> loadSceneCommand)
    {
        _loadSceneCommand = loadSceneCommand;
    }
    
    public void OnPlayButtonClicked()
    {
        if (_loadSceneCommand.CanExecute())
        {
            _loadSceneCommand.Execute("GameScene");
        }
    }
}
```

---

## 4. Data Binding Strategies

### 4.1 Observable Pattern (Manual Binding)

**Observable Property:**
```csharp
using System;

public class ObservableProperty<T>
{
    private T _value;
    
    public event Action<T> ValueChanged;
    
    public T Value
    {
        get => _value;
        set
        {
            if (!Equals(_value, value))
            {
                _value = value;
                ValueChanged?.Invoke(_value);
            }
        }
    }
    
    public ObservableProperty(T initialValue = default)
    {
        _value = initialValue;
    }
}
```

**Observable Collection:**
```csharp
using System;
using System.Collections.Generic;

public class ObservableCollection<T> : List<T>
{
    public event Action<T> ItemAdded;
    public event Action<T> ItemRemoved;
    public event Action CollectionCleared;
    public event Action CollectionChanged;
    
    public new void Add(T item)
    {
        base.Add(item);
        ItemAdded?.Invoke(item);
        CollectionChanged?.Invoke();
    }
    
    public new bool Remove(T item)
    {
        bool removed = base.Remove(item);
        if (removed)
        {
            ItemRemoved?.Invoke(item);
            CollectionChanged?.Invoke();
        }
        return removed;
    }
    
    public new void Clear()
    {
        base.Clear();
        CollectionCleared?.Invoke();
        CollectionChanged?.Invoke();
    }
}
```

**ViewModel with Observables:**
```csharp
public class PlayerViewModel
{
    // Observable properties
    public ObservableProperty<string> PlayerName { get; }
    public ObservableProperty<int> Health { get; }
    public ObservableProperty<int> MaxHealth { get; }
    public ObservableProperty<int> Score { get; }
    
    // Computed property
    public ObservableProperty<float> HealthPercentage { get; }
    
    public PlayerViewModel(PlayerModel model)
    {
        PlayerName = new ObservableProperty<string>(model.Name);
        Health = new ObservableProperty<int>(model.Health);
        MaxHealth = new ObservableProperty<int>(model.MaxHealth);
        Score = new ObservableProperty<int>(model.Score);
        HealthPercentage = new ObservableProperty<float>(1f);
        
        // Update computed property when dependencies change
        Health.ValueChanged += _ => UpdateHealthPercentage();
        MaxHealth.ValueChanged += _ => UpdateHealthPercentage();
    }
    
    private void UpdateHealthPercentage()
    {
        HealthPercentage.Value = (float)Health.Value / MaxHealth.Value;
    }
    
    public void TakeDamage(int damage)
    {
        Health.Value = Mathf.Max(0, Health.Value - damage);
    }
    
    public void Heal(int amount)
    {
        Health.Value = Mathf.Min(MaxHealth.Value, Health.Value + amount);
    }
    
    public void AddScore(int points)
    {
        Score.Value += points;
    }
}
```

**View Binding:**
```csharp
using UnityEngine;
using UnityEngine.UIElements;

public class PlayerHUDView : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;
    
    private Label _playerNameLabel;
    private Label _healthLabel;
    private ProgressBar _healthBar;
    private Label _scoreLabel;
    
    private PlayerViewModel _viewModel;
    
    public void Initialize(PlayerViewModel viewModel)
    {
        _viewModel = viewModel;
        
        // Get UI elements
        var root = _uiDocument.rootVisualElement;
        _playerNameLabel = root.Q<Label>("PlayerName");
        _healthLabel = root.Q<Label>("HealthValue");
        _healthBar = root.Q<ProgressBar>("HealthBar");
        _scoreLabel = root.Q<Label>("ScoreValue");
        
        // Bind to observables
        _viewModel.PlayerName.ValueChanged += OnPlayerNameChanged;
        _viewModel.Health.ValueChanged += OnHealthChanged;
        _viewModel.HealthPercentage.ValueChanged += OnHealthPercentageChanged;
        _viewModel.Score.ValueChanged += OnScoreChanged;
        
        // Initialize with current values
        OnPlayerNameChanged(_viewModel.PlayerName.Value);
        OnHealthChanged(_viewModel.Health.Value);
        OnHealthPercentageChanged(_viewModel.HealthPercentage.Value);
        OnScoreChanged(_viewModel.Score.Value);
    }
    
    private void OnDestroy()
    {
        // Unsubscribe from observables
        if (_viewModel != null)
        {
            _viewModel.PlayerName.ValueChanged -= OnPlayerNameChanged;
            _viewModel.Health.ValueChanged -= OnHealthChanged;
            _viewModel.HealthPercentage.ValueChanged -= OnHealthPercentageChanged;
            _viewModel.Score.ValueChanged -= OnScoreChanged;
        }
    }
    
    private void OnPlayerNameChanged(string name)
    {
        _playerNameLabel.text = name;
    }
    
    private void OnHealthChanged(int health)
    {
        _healthLabel.text = $"{health}/{_viewModel.MaxHealth.Value}";
    }
    
    private void OnHealthPercentageChanged(float percentage)
    {
        _healthBar.value = percentage * 100f;
        
        // Update health bar color based on percentage
        if (percentage < 0.3f)
            _healthBar.RemoveFromClassList("health-ok");
            _healthBar.AddToClassList("health-low");
        else
            _healthBar.RemoveFromClassList("health-low");
            _healthBar.AddToClassList("health-ok");
    }
    
    private void OnScoreChanged(int score)
    {
        _scoreLabel.text = score.ToString("N0");
    }
}
```

### 4.2 Unity 6 Native Data Binding

**ScriptableObject Model with Binding:**
```csharp
using UnityEngine;

[CreateAssetMenu(fileName = "GameSettings", menuName = "Game/Settings")]
public class GameSettings : ScriptableObject
{
    [SerializeField] private float _masterVolume = 1f;
    [SerializeField] private float _musicVolume = 1f;
    [SerializeField] private float _sfxVolume = 1f;
    [SerializeField] private bool _isFullscreen = true;
    [SerializeField] private int _resolutionIndex = 0;
    
    public float MasterVolume
    {
        get => _masterVolume;
        set => _masterVolume = Mathf.Clamp01(value);
    }
    
    public float MusicVolume
    {
        get => _musicVolume;
        set => _musicVolume = Mathf.Clamp01(value);
    }
    
    public float SfxVolume
    {
        get => _sfxVolume;
        set => _sfxVolume = Mathf.Clamp01(value);
    }
    
    public bool IsFullscreen
    {
        get => _isFullscreen;
        set => _isFullscreen = value;
    }
    
    public int ResolutionIndex
    {
        get => _resolutionIndex;
        set => _resolutionIndex = value;
    }
}
```

**UXML with Binding Paths:**
```xml
<ui:UXML xmlns:ui="UnityEngine.UIElements">
    <ui:VisualElement class="settings-panel">
        
        <ui:Label text="Audio Settings" class="section-title" />
        
        <ui:Slider name="MasterVolumeSlider" 
                   label="Master Volume"
                   binding-path="_masterVolume"
                   low-value="0" 
                   high-value="1"
                   class="volume-slider" />
        
        <ui:Slider name="MusicVolumeSlider" 
                   label="Music Volume"
                   binding-path="_musicVolume"
                   low-value="0" 
                   high-value="1"
                   class="volume-slider" />
        
        <ui:Slider name="SfxVolumeSlider" 
                   label="SFX Volume"
                   binding-path="_sfxVolume"
                   low-value="0" 
                   high-value="1"
                   class="volume-slider" />
        
        <ui:Label text="Video Settings" class="section-title" />
        
        <ui:Toggle name="FullscreenToggle" 
                   label="Fullscreen"
                   binding-path="_isFullscreen"
                   class="settings-toggle" />
        
        <ui:DropdownField name="ResolutionDropdown" 
                          label="Resolution"
                          binding-path="_resolutionIndex"
                          class="settings-dropdown" />
        
    </ui:VisualElement>
</ui:UXML>
```

**View Controller with Binding:**
```csharp
using UnityEngine;
using UnityEngine.UIElements;

public class SettingsViewController : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;
    [SerializeField] private GameSettings _settings;
    
    private SerializedObject _serializedSettings;
    private Slider _masterVolumeSlider;
    private AudioSource _audioSource;
    
    private void OnEnable()
    {
        var root = _uiDocument.rootVisualElement;
        
        // Setup binding
        _serializedSettings = new SerializedObject(_settings);
        root.Bind(_serializedSettings);
        
        // Get slider reference for immediate audio feedback
        _masterVolumeSlider = root.Q<Slider>("MasterVolumeSlider");
        
        // Add immediate feedback for volume changes
        _masterVolumeSlider.RegisterValueChangedCallback(evt =>
        {
            AudioListener.volume = evt.newValue;
        });
    }
    
    private void OnDisable()
    {
        // Unbind
        var root = _uiDocument.rootVisualElement;
        root.Unbind();
    }
}
```

### 4.3 Reactive Binding with UniRx (Advanced)

**Note**: This requires UniRx package from GitHub.

```csharp
using UniRx;
using UnityEngine;

public class ReactivePlayerViewModel
{
    // Reactive properties
    public ReactiveProperty<string> PlayerName { get; }
    public ReactiveProperty<int> Health { get; }
    public ReactiveProperty<int> MaxHealth { get; }
    public ReactiveProperty<int> Score { get; }
    
    // Computed reactive property
    public IReadOnlyReactiveProperty<float> HealthPercentage { get; }
    
    public ReactivePlayerViewModel(PlayerModel model)
    {
        PlayerName = new ReactiveProperty<string>(model.Name);
        Health = new ReactiveProperty<int>(model.Health);
        MaxHealth = new ReactiveProperty<int>(model.MaxHealth);
        Score = new ReactiveProperty<int>(model.Score);
        
        // Computed property using CombineLatest
        HealthPercentage = Health
            .CombineLatest(MaxHealth, (h, mh) => (float)h / mh)
            .ToReadOnlyReactiveProperty();
    }
}

// View binding with UniRx
public class ReactivePlayerHUDView : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;
    
    private Label _healthLabel;
    private ProgressBar _healthBar;
    private ReactivePlayerViewModel _viewModel;
    private CompositeDisposable _disposables = new CompositeDisposable();
    
    public void Initialize(ReactivePlayerViewModel viewModel)
    {
        _viewModel = viewModel;
        
        var root = _uiDocument.rootVisualElement;
        _healthLabel = root.Q<Label>("HealthValue");
        _healthBar = root.Q<ProgressBar>("HealthBar");
        
        // Bind with automatic disposal
        _viewModel.Health
            .CombineLatest(_viewModel.MaxHealth, (h, mh) => $"{h}/{mh}")
            .Subscribe(text => _healthLabel.text = text)
            .AddTo(_disposables);
        
        _viewModel.HealthPercentage
            .Subscribe(percentage => _healthBar.value = percentage * 100f)
            .AddTo(_disposables);
    }
    
    private void OnDestroy()
    {
        _disposables.Dispose();
    }
}
```

---

## 5. Factory Pattern Implementation

### 5.1 UI Element Factory

**Base Factory Interface:**
```csharp
using UnityEngine.UIElements;

public interface IUIElementFactory<out T> where T : VisualElement
{
    T Create();
}

public interface IUIElementFactory<out T, in TData> where T : VisualElement
{
    T Create(TData data);
}
```

**Button Factory:**
```csharp
using UnityEngine.UIElements;

public enum ButtonStyle
{
    Primary,
    Secondary,
    Danger,
    Success
}

public class ButtonFactory : IUIElementFactory<Button, ButtonFactoryData>
{
    public Button Create(ButtonFactoryData data)
    {
        var button = new Button
        {
            text = data.Text
        };
        
        // Apply style class
        button.AddToClassList("btn");
        button.AddToClassList(GetStyleClass(data.Style));
        
        // Apply size class if specified
        if (data.Size != ButtonSize.Default)
        {
            button.AddToClassList(GetSizeClass(data.Size));
        }
        
        // Add icon if specified
        if (data.Icon != null)
        {
            var icon = new VisualElement();
            icon.AddToClassList("btn-icon");
            icon.style.backgroundImage = new StyleBackground(data.Icon);
            button.Insert(0, icon);
        }
        
        // Add callback if specified
        if (data.OnClick != null)
        {
            button.clicked += data.OnClick;
        }
        
        return button;
    }
    
    private string GetStyleClass(ButtonStyle style)
    {
        return style switch
        {
            ButtonStyle.Primary => "btn-primary",
            ButtonStyle.Secondary => "btn-secondary",
            ButtonStyle.Danger => "btn-danger",
            ButtonStyle.Success => "btn-success",
            _ => "btn-primary"
        };
    }
    
    private string GetSizeClass(ButtonSize size)
    {
        return size switch
        {
            ButtonSize.Small => "btn-sm",
            ButtonSize.Large => "btn-lg",
            _ => ""
        };
    }
}

public class ButtonFactoryData
{
    public string Text { get; set; }
    public ButtonStyle Style { get; set; } = ButtonStyle.Primary;
    public ButtonSize Size { get; set; } = ButtonSize.Default;
    public Sprite Icon { get; set; }
    public System.Action OnClick { get; set; }
}

public enum ButtonSize
{
    Default,
    Small,
    Large
}
```

**Usage:**
```csharp
var buttonFactory = new ButtonFactory();

var playButton = buttonFactory.Create(new ButtonFactoryData
{
    Text = "Play",
    Style = ButtonStyle.Primary,
    Size = ButtonSize.Large,
    OnClick = () => Debug.Log("Play clicked")
});

var cancelButton = buttonFactory.Create(new ButtonFactoryData
{
    Text = "Cancel",
    Style = ButtonStyle.Secondary,
    OnClick = () => Debug.Log("Cancel clicked")
});
```

### 5.2 Panel Factory

**Panel Factory:**
```csharp
using UnityEngine.UIElements;

public class PanelFactory : IUIElementFactory<VisualElement, PanelFactoryData>
{
    public VisualElement Create(PanelFactoryData data)
    {
        var panel = new VisualElement();
        panel.AddToClassList("panel");
        panel.AddToClassList(GetThemeClass(data.Theme));
        
        // Header
        if (!string.IsNullOrEmpty(data.Title))
        {
            var header = CreateHeader(data.Title, data.HeaderActions);
            panel.Add(header);
        }
        
        // Content
        var content = new VisualElement();
        content.AddToClassList("panel-content");
        
        if (data.Content != null)
        {
            content.Add(data.Content);
        }
        
        panel.Add(content);
        
        // Footer
        if (data.FooterButtons != null && data.FooterButtons.Count > 0)
        {
            var footer = CreateFooter(data.FooterButtons);
            panel.Add(footer);
        }
        
        return panel;
    }
    
    private VisualElement CreateHeader(string title, System.Collections.Generic.List<Button> actions)
    {
        var header = new VisualElement();
        header.AddToClassList("panel-header");
        
        var titleLabel = new Label(title);
        titleLabel.AddToClassList("panel-title");
        header.Add(titleLabel);
        
        if (actions != null && actions.Count > 0)
        {
            var actionsContainer = new VisualElement();
            actionsContainer.AddToClassList("panel-header-actions");
            
            foreach (var action in actions)
            {
                actionsContainer.Add(action);
            }
            
            header.Add(actionsContainer);
        }
        
        return header;
    }
    
    private VisualElement CreateFooter(System.Collections.Generic.List<Button> buttons)
    {
        var footer = new VisualElement();
        footer.AddToClassList("panel-footer");
        
        foreach (var button in buttons)
        {
            footer.Add(button);
        }
        
        return footer;
    }
    
    private string GetThemeClass(PanelTheme theme)
    {
        return theme switch
        {
            PanelTheme.Light => "panel-light",
            PanelTheme.Dark => "panel-dark",
            PanelTheme.Glass => "panel-glass",
            _ => "panel-dark"
        };
    }
}

public class PanelFactoryData
{
    public string Title { get; set; }
    public PanelTheme Theme { get; set; } = PanelTheme.Dark;
    public VisualElement Content { get; set; }
    public System.Collections.Generic.List<Button> HeaderActions { get; set; }
    public System.Collections.Generic.List<Button> FooterButtons { get; set; }
}

public enum PanelTheme
{
    Light,
    Dark,
    Glass
}
```

### 5.3 Abstract Factory Pattern

**UI Component Family:**
```csharp
using UnityEngine.UIElements;

// Abstract factory interface
public interface IUIComponentFactory
{
    Button CreateButton(string text, System.Action onClick = null);
    TextField CreateTextField(string label, string bindingPath = null);
    VisualElement CreatePanel(string title);
    Label CreateLabel(string text);
}

// Modern theme factory
public class ModernUIFactory : IUIComponentFactory
{
    public Button CreateButton(string text, System.Action onClick = null)
    {
        var button = new Button { text = text };
        button.AddToClassList("modern-button");
        
        if (onClick != null)
        {
            button.clicked += onClick;
        }
        
        return button;
    }
    
    public TextField CreateTextField(string label, string bindingPath = null)
    {
        var field = new TextField(label);
        field.AddToClassList("modern-textfield");
        
        if (!string.IsNullOrEmpty(bindingPath))
        {
            field.bindingPath = bindingPath;
        }
        
        return field;
    }
    
    public VisualElement CreatePanel(string title)
    {
        var panel = new VisualElement();
        panel.AddToClassList("modern-panel");
        
        if (!string.IsNullOrEmpty(title))
        {
            var titleLabel = new Label(title);
            titleLabel.AddToClassList("modern-panel-title");
            panel.Add(titleLabel);
        }
        
        return panel;
    }
    
    public Label CreateLabel(string text)
    {
        var label = new Label(text);
        label.AddToClassList("modern-label");
        return label;
    }
}

// Retro theme factory
public class RetroUIFactory : IUIComponentFactory
{
    public Button CreateButton(string text, System.Action onClick = null)
    {
        var button = new Button { text = text };
        button.AddToClassList("retro-button");
        
        if (onClick != null)
        {
            button.clicked += onClick;
        }
        
        return button;
    }
    
    public TextField CreateTextField(string label, string bindingPath = null)
    {
        var field = new TextField(label);
        field.AddToClassList("retro-textfield");
        
        if (!string.IsNullOrEmpty(bindingPath))
        {
            field.bindingPath = bindingPath;
        }
        
        return field;
    }
    
    public VisualElement CreatePanel(string title)
    {
        var panel = new VisualElement();
        panel.AddToClassList("retro-panel");
        
        if (!string.IsNullOrEmpty(title))
        {
            var titleLabel = new Label(title);
            titleLabel.AddToClassList("retro-panel-title");
            panel.Add(titleLabel);
        }
        
        return panel;
    }
    
    public Label CreateLabel(string text)
    {
        var label = new Label(text);
        label.AddToClassList("retro-label");
        return label;
    }
}

// Usage with dependency injection
public class UIBuilder
{
    private readonly IUIComponentFactory _factory;
    
    public UIBuilder(IUIComponentFactory factory)
    {
        _factory = factory;
    }
    
    public VisualElement BuildLoginForm()
    {
        var panel = _factory.CreatePanel("Login");
        
        var usernameField = _factory.CreateTextField("Username");
        var passwordField = _factory.CreateTextField("Password");
        
        var loginButton = _factory.CreateButton("Login", OnLoginClicked);
        var cancelButton = _factory.CreateButton("Cancel", OnCancelClicked);
        
        panel.Add(usernameField);
        panel.Add(passwordField);
        panel.Add(loginButton);
        panel.Add(cancelButton);
        
        return panel;
    }
    
    private void OnLoginClicked() { /* ... */ }
    private void OnCancelClicked() { /* ... */ }
}
```

---

## 6. View-Controller Patterns

### 6.1 Screen Controller Pattern

**Base Screen Controller:**
```csharp
using UnityEngine;
using UnityEngine.UIElements;

public abstract class ScreenController : MonoBehaviour
{
    [SerializeField] protected UIDocument _uiDocument;
    
    protected VisualElement Root => _uiDocument.rootVisualElement;
    
    public virtual void Initialize()
    {
        // Override in derived classes
    }
    
    public virtual void Show()
    {
        Root.style.display = DisplayStyle.Flex;
        OnShown();
    }
    
    public virtual void Hide()
    {
        Root.style.display = DisplayStyle.None;
        OnHidden();
    }
    
    protected virtual void OnShown() { }
    protected virtual void OnHidden() { }
    
    public virtual void Dispose()
    {
        // Clean up resources
    }
}
```

**Concrete Screen Controller:**
```csharp
public class InventoryScreenController : ScreenController
{
    [SerializeField] private InventoryViewModel _viewModel;
    
    private VisualElement _itemsContainer;
    private Button _closeButton;
    private TextField _searchField;
    
    public override void Initialize()
    {
        base.Initialize();
        
        // Query elements
        _itemsContainer = Root.Q<VisualElement>("ItemsContainer");
        _closeButton = Root.Q<Button>("CloseButton");
        _searchField = Root.Q<TextField>("SearchField");
        
        // Wire up events
        _closeButton.clicked += OnCloseClicked;
        _searchField.RegisterValueChangedCallback(OnSearchChanged);
        
        // Bind to ViewModel
        BindToViewModel();
    }
    
    private void BindToViewModel()
    {
        _viewModel.Items.CollectionChanged += RefreshItems;
        RefreshItems();
    }
    
    private void RefreshItems()
    {
        _itemsContainer.Clear();
        
        foreach (var item in _viewModel.FilteredItems)
        {
            var itemElement = CreateItemElement(item);
            _itemsContainer.Add(itemElement);
        }
    }
    
    private VisualElement CreateItemElement(InventoryItemData item)
    {
        var itemElement = new VisualElement();
        itemElement.AddToClassList("inventory-item");
        
        var icon = new VisualElement();
        icon.style.backgroundImage = new StyleBackground(item.Icon);
        icon.AddToClassList("item-icon");
        
        var nameLabel = new Label(item.Name);
        nameLabel.AddToClassList("item-name");
        
        var quantityLabel = new Label($"x{item.Quantity}");
        quantityLabel.AddToClassList("item-quantity");
        
        itemElement.Add(icon);
        itemElement.Add(nameLabel);
        itemElement.Add(quantityLabel);
        
        // Click handler
        itemElement.RegisterCallback<ClickEvent>(evt => OnItemClicked(item));
        
        return itemElement;
    }
    
    private void OnItemClicked(InventoryItemData item)
    {
        _viewModel.SelectItem(item);
    }
    
    private void OnSearchChanged(ChangeEvent<string> evt)
    {
        _viewModel.SetSearchFilter(evt.newValue);
    }
    
    private void OnCloseClicked()
    {
        Hide();
    }
    
    public override void Dispose()
    {
        base.Dispose();
        
        _viewModel.Items.CollectionChanged -= RefreshItems;
        _closeButton.clicked -= OnCloseClicked;
    }
}
```

### 6.2 Screen Manager / Navigation System

**Screen Manager:**
```csharp
using System.Collections.Generic;
using UnityEngine;

public class ScreenManager : MonoBehaviour
{
    private static ScreenManager _instance;
    public static ScreenManager Instance => _instance;
    
    private Dictionary<string, ScreenController> _screens = new Dictionary<string, ScreenController>();
    private Stack<ScreenController> _history = new Stack<ScreenController>();
    private ScreenController _currentScreen;
    
    private void Awake()
    {
        if (_instance != null && _instance != this)
        {
            Destroy(gameObject);
            return;
        }
        
        _instance = this;
        DontDestroyOnLoad(gameObject);
    }
    
    public void RegisterScreen(string screenId, ScreenController screen)
    {
        if (!_screens.ContainsKey(screenId))
        {
            _screens.Add(screenId, screen);
            screen.Initialize();
            screen.Hide();
        }
    }
    
    public void ShowScreen(string screenId, bool addToHistory = true)
    {
        if (!_screens.TryGetValue(screenId, out var screen))
        {
            Debug.LogError($"Screen '{screenId}' not registered!");
            return;
        }
        
        // Hide current screen
        if (_currentScreen != null)
        {
            _currentScreen.Hide();
            
            if (addToHistory)
            {
                _history.Push(_currentScreen);
            }
        }
        
        // Show new screen
        _currentScreen = screen;
        _currentScreen.Show();
    }
    
    public void GoBack()
    {
        if (_history.Count > 0)
        {
            var previousScreen = _history.Pop();
            
            if (_currentScreen != null)
            {
                _currentScreen.Hide();
            }
            
            _currentScreen = previousScreen;
            _currentScreen.Show();
        }
    }
    
    public void ClearHistory()
    {
        _history.Clear();
    }
    
    public T GetScreen<T>(string screenId) where T : ScreenController
    {
        if (_screens.TryGetValue(screenId, out var screen))
        {
            return screen as T;
        }
        
        return null;
    }
}

// Usage:
public class GameBootstrap : MonoBehaviour
{
    [SerializeField] private ScreenController _mainMenuScreen;
    [SerializeField] private ScreenController _gameplayScreen;
    [SerializeField] private ScreenController _settingsScreen;
    
    private void Start()
    {
        var screenManager = ScreenManager.Instance;
        
        screenManager.RegisterScreen("MainMenu", _mainMenuScreen);
        screenManager.RegisterScreen("Gameplay", _gameplayScreen);
        screenManager.RegisterScreen("Settings", _settingsScreen);
        
        screenManager.ShowScreen("MainMenu");
    }
}
```

### 6.3 Presenter Pattern (MVP)

**Presenter Interface:**
```csharp
public interface IPresenter
{
    void Initialize();
    void OnViewShown();
    void OnViewHidden();
    void Dispose();
}

public interface IPresenter<TView> : IPresenter where TView : IView
{
    TView View { get; set; }
}
```

**Concrete Presenter:**
```csharp
public class InventoryPresenter : IPresenter<IInventoryView>
{
    public IInventoryView View { get; set; }
    
    private readonly InventoryService _inventoryService;
    private readonly IUIFactory _uiFactory;
    
    public InventoryPresenter(InventoryService inventoryService, IUIFactory uiFactory)
    {
        _inventoryService = inventoryService;
        _uiFactory = uiFactory;
    }
    
    public void Initialize()
    {
        // Subscribe to view events
        View.ItemSelected += OnItemSelected;
        View.SearchChanged += OnSearchChanged;
        View.CloseRequested += OnCloseRequested;
        
        // Subscribe to model events
        _inventoryService.InventoryChanged += OnInventoryChanged;
        
        // Initial data load
        RefreshView();
    }
    
    public void OnViewShown()
    {
        RefreshView();
    }
    
    public void OnViewHidden()
    {
        // Cleanup if needed
    }
    
    public void Dispose()
    {
        // Unsubscribe from events
        View.ItemSelected -= OnItemSelected;
        View.SearchChanged -= OnSearchChanged;
        View.CloseRequested -= OnCloseRequested;
        
        _inventoryService.InventoryChanged -= OnInventoryChanged;
    }
    
    private void RefreshView()
    {
        var items = _inventoryService.GetAllItems();
        View.DisplayItems(items);
    }
    
    private void OnItemSelected(InventoryItemData item)
    {
        View.ShowItemDetails(item);
    }
    
    private void OnSearchChanged(string searchText)
    {
        var filteredItems = _inventoryService.SearchItems(searchText);
        View.DisplayItems(filteredItems);
    }
    
    private void OnCloseRequested()
    {
        View.Hide();
    }
    
    private void OnInventoryChanged()
    {
        RefreshView();
    }
}
```

---

## 7. Runtime Customization

### 7.1 Dynamic Theme System

**Theme Configuration:**
```csharp
using UnityEngine;
using System.Collections.Generic;

[CreateAssetMenu(fileName = "UITheme", menuName = "UI/Theme")]
public class UITheme : ScriptableObject
{
    [System.Serializable]
    public class ColorPalette
    {
        public Color primary;
        public Color secondary;
        public Color background;
        public Color surface;
        public Color error;
        public Color warning;
        public Color success;
        public Color textPrimary;
        public Color textSecondary;
    }
    
    [System.Serializable]
    public class Typography
    {
        public int fontSizeSmall = 12;
        public int fontSizeMedium = 16;
        public int fontSizeLarge = 24;
        public int fontSizeXLarge = 32;
    }
    
    [System.Serializable]
    public class Spacing
    {
        public int small = 8;
        public int medium = 16;
        public int large = 24;
        public int xLarge = 32;
    }
    
    public string themeName;
    public ColorPalette colors;
    public Typography typography;
    public Spacing spacing;
    public StyleSheet styleSheet;
}
```

**Theme Manager:**
```csharp
using UnityEngine;
using UnityEngine.UIElements;
using System.Collections.Generic;

public class ThemeManager : MonoBehaviour
{
    private static ThemeManager _instance;
    public static ThemeManager Instance => _instance;
    
    [SerializeField] private List<UITheme> _availableThemes;
    [SerializeField] private int _currentThemeIndex = 0;
    
    private List<UIDocument> _registeredDocuments = new List<UIDocument>();
    private UITheme _currentTheme;
    
    public UITheme CurrentTheme => _currentTheme;
    
    public event System.Action<UITheme> ThemeChanged;
    
    private void Awake()
    {
        if (_instance != null && _instance != this)
        {
            Destroy(gameObject);
            return;
        }
        
        _instance = this;
        DontDestroyOnLoad(gameObject);
        
        ApplyTheme(_currentThemeIndex);
    }
    
    public void RegisterDocument(UIDocument document)
    {
        if (!_registeredDocuments.Contains(document))
        {
            _registeredDocuments.Add(document);
            ApplyThemeToDocument(document, _currentTheme);
        }
    }
    
    public void UnregisterDocument(UIDocument document)
    {
        _registeredDocuments.Remove(document);
    }
    
    public void ApplyTheme(int themeIndex)
    {
        if (themeIndex < 0 || themeIndex >= _availableThemes.Count)
            return;
        
        _currentThemeIndex = themeIndex;
        _currentTheme = _availableThemes[themeIndex];
        
        // Apply to all registered documents
        foreach (var document in _registeredDocuments)
        {
            ApplyThemeToDocument(document, _currentTheme);
        }
        
        ThemeChanged?.Invoke(_currentTheme);
    }
    
    public void ApplyThemeByName(string themeName)
    {
        int index = _availableThemes.FindIndex(t => t.themeName == themeName);
        if (index >= 0)
        {
            ApplyTheme(index);
        }
    }
    
    public void NextTheme()
    {
        int nextIndex = (_currentThemeIndex + 1) % _availableThemes.Count;
        ApplyTheme(nextIndex);
    }
    
    private void ApplyThemeToDocument(UIDocument document, UITheme theme)
    {
        var root = document.rootVisualElement;
        
        // Remove previous theme stylesheets
        foreach (var availableTheme in _availableThemes)
        {
            if (availableTheme.styleSheet != null)
            {
                root.styleSheets.Remove(availableTheme.styleSheet);
            }
        }
        
        // Add new theme stylesheet
        if (theme.styleSheet != null)
        {
            root.styleSheets.Add(theme.styleSheet);
        }
        
        // Apply CSS variables
        ApplyThemeVariables(root, theme);
    }
    
    private void ApplyThemeVariables(VisualElement root, UITheme theme)
    {
        // Color variables
        root.style.SetCustomProperty("--color-primary", theme.colors.primary);
        root.style.SetCustomProperty("--color-secondary", theme.colors.secondary);
        root.style.SetCustomProperty("--color-background", theme.colors.background);
        root.style.SetCustomProperty("--color-surface", theme.colors.surface);
        root.style.SetCustomProperty("--color-error", theme.colors.error);
        root.style.SetCustomProperty("--color-warning", theme.colors.warning);
        root.style.SetCustomProperty("--color-success", theme.colors.success);
        root.style.SetCustomProperty("--color-text-primary", theme.colors.textPrimary);
        root.style.SetCustomProperty("--color-text-secondary", theme.colors.textSecondary);
        
        // Typography variables
        root.style.SetCustomProperty("--font-size-sm", new Length(theme.typography.fontSizeSmall, LengthUnit.Pixel));
        root.style.SetCustomProperty("--font-size-md", new Length(theme.typography.fontSizeMedium, LengthUnit.Pixel));
        root.style.SetCustomProperty("--font-size-lg", new Length(theme.typography.fontSizeLarge, LengthUnit.Pixel));
        root.style.SetCustomProperty("--font-size-xl", new Length(theme.typography.fontSizeXLarge, LengthUnit.Pixel));
        
        // Spacing variables
        root.style.SetCustomProperty("--spacing-sm", new Length(theme.spacing.small, LengthUnit.Pixel));
        root.style.SetCustomProperty("--spacing-md", new Length(theme.spacing.medium, LengthUnit.Pixel));
        root.style.SetCustomProperty("--spacing-lg", new Length(theme.spacing.large, LengthUnit.Pixel));
        root.style.SetCustomProperty("--spacing-xl", new Length(theme.spacing.xLarge, LengthUnit.Pixel));
    }
}

// Extension method for setting custom properties
public static class StyleExtensions
{
    public static void SetCustomProperty(this IStyle style, string propertyName, Color value)
    {
        // Note: In Unity 6, you can use actual CSS custom properties
        // For now, we apply colors directly
        // This is a placeholder for the actual implementation
    }
    
    public static void SetCustomProperty(this IStyle style, string propertyName, Length value)
    {
        // Similar placeholder
    }
}
```

### 7.2 Runtime Style Modification

**Dynamic Style Controller:**
```csharp
using UnityEngine;
using UnityEngine.UIElements;

public class DynamicStyleController : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;
    
    // Runtime style modification examples
    public void SetButtonStyle(Button button, Color backgroundColor, Color textColor)
    {
        button.style.backgroundColor = backgroundColor;
        button.style.color = textColor;
    }
    
    public void AnimateElementScale(VisualElement element, float targetScale, float duration)
    {
        StartCoroutine(AnimateScaleCoroutine(element, targetScale, duration));
    }
    
    private System.Collections.IEnumerator AnimateScaleCoroutine(VisualElement element, float targetScale, float duration)
    {
        float startScale = element.resolvedStyle.scale.value.x;
        float elapsedTime = 0f;
        
        while (elapsedTime < duration)
        {
            elapsedTime += Time.deltaTime;
            float t = elapsedTime / duration;
            float currentScale = Mathf.Lerp(startScale, targetScale, t);
            
            element.style.scale = new Scale(new Vector3(currentScale, currentScale, 1f));
            
            yield return null;
        }
        
        element.style.scale = new Scale(new Vector3(targetScale, targetScale, 1f));
    }
    
    public void AnimateElementOpacity(VisualElement element, float targetOpacity, float duration)
    {
        StartCoroutine(AnimateOpacityCoroutine(element, targetOpacity, duration));
    }
    
    private System.Collections.IEnumerator AnimateOpacityCoroutine(VisualElement element, float targetOpacity, float duration)
    {
        float startOpacity = element.resolvedStyle.opacity;
        float elapsedTime = 0f;
        
        while (elapsedTime < duration)
        {
            elapsedTime += Time.deltaTime;
            float t = elapsedTime / duration;
            float currentOpacity = Mathf.Lerp(startOpacity, targetOpacity, t);
            
            element.style.opacity = currentOpacity;
            
            yield return null;
        }
        
        element.style.opacity = targetOpacity;
    }
    
    // Color transition
    public void AnimateBackgroundColor(VisualElement element, Color targetColor, float duration)
    {
        StartCoroutine(AnimateColorCoroutine(element, targetColor, duration));
    }
    
    private System.Collections.IEnumerator AnimateColorCoroutine(VisualElement element, Color targetColor, float duration)
    {
        Color startColor = element.resolvedStyle.backgroundColor;
        float elapsedTime = 0f;
        
        while (elapsedTime < duration)
        {
            elapsedTime += Time.deltaTime;
            float t = elapsedTime / duration;
            Color currentColor = Color.Lerp(startColor, targetColor, t);
            
            element.style.backgroundColor = currentColor;
            
            yield return null;
        }
        
        element.style.backgroundColor = targetColor;
    }
}
```

### 7.3 User Customization System

**User Preferences:**
```csharp
using UnityEngine;

[System.Serializable]
public class UserUIPreferences
{
    public string selectedTheme = "Dark";
    public float uiScale = 1.0f;
    public bool highContrastMode = false;
    public bool reducedMotion = false;
    public string language = "en";
    
    public void Save()
    {
        string json = JsonUtility.ToJson(this);
        PlayerPrefs.SetString("UserUIPreferences", json);
        PlayerPrefs.Save();
    }
    
    public static UserUIPreferences Load()
    {
        if (PlayerPrefs.HasKey("UserUIPreferences"))
        {
            string json = PlayerPrefs.GetString("UserUIPreferences");
            return JsonUtility.FromJson<UserUIPreferences>(json);
        }
        
        return new UserUIPreferences();
    }
}
```

**Preference Controller:**
```csharp
using UnityEngine;
using UnityEngine.UIElements;

public class UIPreferenceController : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;
    
    private UserUIPreferences _preferences;
    private Slider _scaleSlider;
    private Toggle _highContrastToggle;
    private Toggle _reducedMotionToggle;
    private DropdownField _themeDropdown;
    
    private void OnEnable()
    {
        _preferences = UserUIPreferences.Load();
        
        var root = _uiDocument.rootVisualElement;
        
        _scaleSlider = root.Q<Slider>("UIScaleSlider");
        _highContrastToggle = root.Q<Toggle>("HighContrastToggle");
        _reducedMotionToggle = root.Q<Toggle>("ReducedMotionToggle");
        _themeDropdown = root.Q<DropdownField>("ThemeDropdown");
        
        // Initialize with saved preferences
        _scaleSlider.value = _preferences.uiScale;
        _highContrastToggle.value = _preferences.highContrastMode;
        _reducedMotionToggle.value = _preferences.reducedMotion;
        
        // Register callbacks
        _scaleSlider.RegisterValueChangedCallback(OnScaleChanged);
        _highContrastToggle.RegisterValueChangedCallback(OnHighContrastChanged);
        _reducedMotionToggle.RegisterValueChangedCallback(OnReducedMotionChanged);
        _themeDropdown.RegisterValueChangedCallback(OnThemeChanged);
        
        // Apply preferences
        ApplyPreferences();
    }
    
    private void ApplyPreferences()
    {
        // Apply scale
        _uiDocument.rootVisualElement.style.scale = new Scale(new Vector3(
            _preferences.uiScale,
            _preferences.uiScale,
            1f
        ));
        
        // Apply theme
        ThemeManager.Instance.ApplyThemeByName(_preferences.selectedTheme);
        
        // Apply high contrast
        if (_preferences.highContrastMode)
        {
            _uiDocument.rootVisualElement.AddToClassList("high-contrast");
        }
        
        // Apply reduced motion
        if (_preferences.reducedMotion)
        {
            _uiDocument.rootVisualElement.AddToClassList("reduced-motion");
        }
    }
    
    private void OnScaleChanged(ChangeEvent<float> evt)
    {
        _preferences.uiScale = evt.newValue;
        _uiDocument.rootVisualElement.style.scale = new Scale(new Vector3(
            evt.newValue,
            evt.newValue,
            1f
        ));
        _preferences.Save();
    }
    
    private void OnHighContrastChanged(ChangeEvent<bool> evt)
    {
        _preferences.highContrastMode = evt.newValue;
        
        if (evt.newValue)
        {
            _uiDocument.rootVisualElement.AddToClassList("high-contrast");
        }
        else
        {
            _uiDocument.rootVisualElement.RemoveFromClassList("high-contrast");
        }
        
        _preferences.Save();
    }
    
    private void OnReducedMotionChanged(ChangeEvent<bool> evt)
    {
        _preferences.reducedMotion = evt.newValue;
        
        if (evt.newValue)
        {
            _uiDocument.rootVisualElement.AddToClassList("reduced-motion");
        }
        else
        {
            _uiDocument.rootVisualElement.RemoveFromClassList("reduced-motion");
        }
        
        _preferences.Save();
    }
    
    private void OnThemeChanged(ChangeEvent<string> evt)
    {
        _preferences.selectedTheme = evt.newValue;
        ThemeManager.Instance.ApplyThemeByName(evt.newValue);
        _preferences.Save();
    }
}
```

**USS for Accessibility:**
```css
/* Reduced motion support */
.reduced-motion * {
    transition-duration: 0s !important;
    animation-duration: 0s !important;
}

/* High contrast mode */
.high-contrast .button {
    border-width: 3px;
    border-color: #FFFFFF;
}

.high-contrast .text {
    color: #FFFFFF;
    text-shadow: 1px 1px 2px #000000;
}
```

---

## 8. Team Collaboration Patterns

### 8.1 Component Library Structure

**Shared Component Library:**
```
Assets/UI/Components/
├── Buttons/
│   ├── PrimaryButton.cs
│   ├── SecondaryButton.cs
│   └── IconButton.cs
├── Forms/
│   ├── TextInputField.cs
│   ├── NumberInputField.cs
│   └── DropdownSelector.cs
├── Panels/
│   ├── DialogPanel.cs
│   ├── ModalPanel.cs
│   └── TooltipPanel.cs
└── Lists/
    ├── ScrollableList.cs
    ├── GridList.cs
    └── VirtualizedList.cs
```

**Base Component Class:**
```csharp
using UnityEngine.UIElements;

public abstract class UIComponent
{
    protected VisualElement Root { get; private set; }
    
    public VisualElement Element => Root;
    
    protected UIComponent()
    {
        Root = CreateElement();
        InitializeComponent();
    }
    
    protected abstract VisualElement CreateElement();
    
    protected virtual void InitializeComponent()
    {
        // Override in derived classes
    }
    
    public virtual void Dispose()
    {
        // Cleanup
        Root?.RemoveFromHierarchy();
        Root = null;
    }
}
```

**Example Component:**
```csharp
using UnityEngine.UIElements;

public class PrimaryButton : UIComponent
{
    private Button _button;
    
    public event System.Action Clicked;
    
    public string Text
    {
        get => _button.text;
        set => _button.text = value;
    }
    
    protected override VisualElement CreateElement()
    {
        _button = new Button();
        _button.AddToClassList("btn");
        _button.AddToClassList("btn-primary");
        
        return _button;
    }
    
    protected override void InitializeComponent()
    {
        _button.clicked += OnButtonClicked;
    }
    
    private void OnButtonClicked()
    {
        Clicked?.Invoke();
    }
    
    public override void Dispose()
    {
        _button.clicked -= OnButtonClicked;
        base.Dispose();
    }
}

// Usage:
var playButton = new PrimaryButton();
playButton.Text = "Play Game";
playButton.Clicked += OnPlayClicked;
root.Add(playButton.Element);
```

### 8.2 Style Guide & Documentation

**Component Documentation Template:**
```csharp
/// <summary>
/// A reusable dialog panel component.
/// </summary>
/// <example>
/// var dialog = new DialogPanel("Confirm Action", "Are you sure?");
/// dialog.AddButton("Yes", OnConfirm);
/// dialog.AddButton("No", OnCancel);
/// dialog.Show();
/// </example>
public class DialogPanel : UIComponent
{
    // Implementation...
}
```

**README for Component Library:**
```markdown
# UI Component Library

## Installation
1. Import package via Package Manager
2. Add stylesheet to your UIDocument: `Assets/UI/Styles/ComponentLibrary.uss`

## Usage

### Buttons

#### Primary Button
```csharp
var button = new PrimaryButton();
button.Text = "Click Me";
button.Clicked += () => Debug.Log("Clicked!");
```

#### Icon Button
```csharp
var iconButton = new IconButton(myIconSprite);
iconButton.Clicked += OnIconClicked;
```

### Forms

#### Text Input
```csharp
var textInput = new TextInputField("Username");
textInput.ValueChanged += OnUsernameChanged;
```

## Styling

All components use BEM naming convention:
- `.component-name` - Block
- `.component-name__element` - Element
- `.component-name--modifier` - Modifier

## Theme Support

Components automatically adapt to the active theme. Use ThemeManager to switch themes:

```csharp
ThemeManager.Instance.ApplyTheme("Dark");
```
```

### 8.3 Version Control Best Practices

**Directory Structure for Teams:**
```
Assets/
├── UI/
│   ├── Common/              # Shared by all devs (review required)
│   │   ├── Components/
│   │   ├── Styles/
│   │   └── Templates/
│   │
│   ├── Features/            # Feature-specific UI
│   │   ├── MainMenu/        # Developer A
│   │   ├── Gameplay/        # Developer B
│   │   └── Settings/        # Developer C
│   │
│   └── Prototypes/          # Experimental (excluded from builds)
│       └── NewFeature/
```

**.gitignore for UI Toolkit:**
```
# Temporary UI files
*.meta~
*.uxml~
*.uss~

# Build artifacts
/Assets/UI/Generated/

# Personal prototypes
/Assets/UI/Prototypes/
```

**USS Merge Strategy:**
```css
/* ===== CORE STYLES - DO NOT MODIFY WITHOUT TEAM REVIEW ===== */
/* Owner: Tech Lead */
/* Last Updated: 2024-01-15 */

:root {
    --primary-color: #8A2BE2;
    /* ... */
}

/* ===== FEATURE: MAIN MENU ===== */
/* Owner: Developer A */
/* Last Updated: 2024-01-20 */

.main-menu-container {
    /* ... */
}

/* ===== FEATURE: GAMEPLAY HUD ===== */
/* Owner: Developer B */
/* Last Updated: 2024-01-18 */

.hud-container {
    /* ... */
}
```

### 8.4 Code Review Checklist

**UI Code Review Checklist:**

```markdown
# UI Code Review Checklist

## Architecture
- [ ] Proper separation of concerns (View/ViewModel/Model)
- [ ] No business logic in views
- [ ] Interfaces used for dependencies
- [ ] Component is reusable

## Code Quality
- [ ] Follows team naming conventions
- [ ] Properly documented with XML comments
- [ ] No magic numbers (use constants)
- [ ] Memory leaks prevented (event unsubscription)

## Performance
- [ ] No expensive operations in Update/OnGUI
- [ ] UI elements pooled if created dynamically
- [ ] Bindings use efficient change detection
- [ ] No unnecessary rebuilds

## Styling
- [ ] Uses theme variables (not hardcoded colors)
- [ ] Follows BEM naming convention
- [ ] Responsive to different screen sizes
- [ ] Accessibility considered

## Testing
- [ ] Unit tests for ViewModel logic
- [ ] Manual testing on target platforms
- [ ] Works with all supported themes
```

---

## 9. Testing Strategies

### 9.1 Unit Testing ViewModels

**ViewModel Test:**
```csharp
using NUnit.Framework;

public class PlayerViewModelTests
{
    private PlayerViewModel _viewModel;
    private PlayerModel _model;
    
    [SetUp]
    public void SetUp()
    {
        _model = new PlayerModel
        {
            Name = "TestPlayer",
            Health = 100,
            MaxHealth = 100
        };
        
        _viewModel = new PlayerViewModel(_model);
    }
    
    [Test]
    public void TakeDamage_ReducesHealth()
    {
        // Arrange
        int damage = 25;
        
        // Act
        _viewModel.TakeDamage(damage);
        
        // Assert
        Assert.AreEqual(75, _viewModel.Health.Value);
    }
    
    [Test]
    public void TakeDamage_DoesNotGoBelowZero()
    {
        // Arrange
        int damage = 150;
        
        // Act
        _viewModel.TakeDamage(damage);
        
        // Assert
        Assert.AreEqual(0, _viewModel.Health.Value);
    }
    
    [Test]
    public void HealthPercentage_UpdatesWhenHealthChanges()
    {
        // Arrange
        float? capturedPercentage = null;
        _viewModel.HealthPercentage.ValueChanged += p => capturedPercentage = p;
        
        // Act
        _viewModel.TakeDamage(50);
        
        // Assert
        Assert.AreEqual(0.5f, capturedPercentage);
    }
    
    [Test]
    public void Heal_IncreasesHealth()
    {
        // Arrange
        _viewModel.TakeDamage(50);
        
        // Act
        _viewModel.Heal(25);
        
        // Assert
        Assert.AreEqual(75, _viewModel.Health.Value);
    }
    
    [Test]
    public void Heal_DoesNotExceedMaxHealth()
    {
        // Arrange
        _viewModel.TakeDamage(10);
        
        // Act
        _viewModel.Heal(50);
        
        // Assert
        Assert.AreEqual(100, _viewModel.Health.Value);
    }
}
```

### 9.2 Integration Testing

**View Integration Test:**
```csharp
using NUnit.Framework;
using UnityEngine;
using UnityEngine.UIElements;
using UnityEngine.TestTools;
using System.Collections;

public class PlayerHUDViewIntegrationTests
{
    private GameObject _testObject;
    private PlayerHUDView _view;
    private PlayerViewModel _viewModel;
    
    [UnitySetUp]
    public IEnumerator SetUp()
    {
        // Create test GameObject
        _testObject = new GameObject("TestHUD");
        
        // Add UIDocument component
        var uiDocument = _testObject.AddComponent<UIDocument>();
        
        // Load test UXML
        var visualTree = Resources.Load<VisualTreeAsset>("UI/Tests/PlayerHUDTest");
        uiDocument.visualTreeAsset = visualTree;
        
        // Add view component
        _view = _testObject.AddComponent<PlayerHUDView>();
        
        // Create ViewModel
        var model = new PlayerModel { Health = 100, MaxHealth = 100 };
        _viewModel = new PlayerViewModel(model);
        
        // Initialize
        _view.Initialize(_viewModel);
        
        yield return null; // Wait for UI to build
    }
    
    [UnityTearDown]
    public IEnumerator TearDown()
    {
        Object.Destroy(_testObject);
        yield return null;
    }
    
    [UnityTest]
    public IEnumerator HealthLabel_UpdatesWhenHealthChanges()
    {
        // Act
        _viewModel.TakeDamage(25);
        
        yield return null; // Wait for UI update
        
        // Assert
        var healthLabel = _view.GetComponent<UIDocument>()
            .rootVisualElement.Q<Label>("HealthValue");
        
        Assert.AreEqual("75/100", healthLabel.text);
    }
    
    [UnityTest]
    public IEnumerator HealthBar_UpdatesWhenHealthChanges()
    {
        // Act
        _viewModel.TakeDamage(50);
        
        yield return null;
        
        // Assert
        var healthBar = _view.GetComponent<UIDocument>()
            .rootVisualElement.Q<ProgressBar>("HealthBar");
        
        Assert.AreEqual(50f, healthBar.value);
    }
}
```

### 9.3 UI Automation Testing

**Automated UI Test Helper:**
```csharp
using UnityEngine.UIElements;

public static class UITestHelper
{
    public static void ClickButton(VisualElement root, string buttonName)
    {
        var button = root.Q<Button>(buttonName);
        if (button == null)
        {
            throw new System.Exception($"Button '{buttonName}' not found!");
        }
        
        // Simulate click
        using (var evt = ClickEvent.GetPooled())
        {
            evt.target = button;
            button.SendEvent(evt);
        }
    }
    
    public static void SetTextFieldValue(VisualElement root, string fieldName, string value)
    {
        var field = root.Q<TextField>(fieldName);
        if (field == null)
        {
            throw new System.Exception($"TextField '{fieldName}' not found!");
        }
        
        field.value = value;
    }
    
    public static string GetLabelText(VisualElement root, string labelName)
    {
        var label = root.Q<Label>(labelName);
        if (label == null)
        {
            throw new System.Exception($"Label '{labelName}' not found!");
        }
        
        return label.text;
    }
    
    public static bool IsElementVisible(VisualElement root, string elementName)
    {
        var element = root.Q(elementName);
        if (element == null)
        {
            return false;
        }
        
        return element.resolvedStyle.display != DisplayStyle.None 
            && element.resolvedStyle.opacity > 0;
    }
}
```

---

## 10. Performance & Scalability

### 10.1 Virtual Scrolling / List Virtualization

**Virtualized List Implementation:**
```csharp
using UnityEngine.UIElements;
using System.Collections.Generic;

public class VirtualizedListView<TData> where TData : class
{
    private ListView _listView;
    private List<TData> _items;
    private System.Func<TData, VisualElement> _makeItem;
    private System.Action<VisualElement, TData> _bindItem;
    
    public VirtualizedListView(
        ListView listView,
        System.Func<TData, VisualElement> makeItem,
        System.Action<VisualElement, TData> bindItem)
    {
        _listView = listView;
        _makeItem = makeItem;
        _bindItem = bindItem;
        _items = new List<TData>();
        
        SetupListView();
    }
    
    private void SetupListView()
    {
        _listView.makeItem = () => _makeItem(default);
        _listView.bindItem = (element, index) =>
        {
            if (index < _items.Count)
            {
                _bindItem(element, _items[index]);
            }
        };
        
        _listView.fixedItemHeight = 50; // Set appropriate height
        _listView.virtualizationMethod = CollectionVirtualizationMethod.FixedHeight;
    }
    
    public void SetItems(List<TData> items)
    {
        _items = items;
        _listView.itemsSource = _items;
        _listView.Rebuild();
    }
    
    public void AddItem(TData item)
    {
        _items.Add(item);
        _listView.RefreshItems();
    }
    
    public void RemoveItem(TData item)
    {
        _items.Remove(item);
        _listView.RefreshItems();
    }
    
    public void Clear()
    {
        _items.Clear();
        _listView.RefreshItems();
    }
}

// Usage:
var listView = root.Q<ListView>("ItemList");

var virtualizedList = new VirtualizedListView<InventoryItemData>(
    listView,
    makeItem: (data) =>
    {
        var container = new VisualElement();
        container.AddToClassList("list-item");
        
        var icon = new VisualElement();
        icon.AddToClassList("item-icon");
        
        var label = new Label();
        label.AddToClassList("item-label");
        
        container.Add(icon);
        container.Add(label);
        
        return container;
    },
    bindItem: (element, data) =>
    {
        var icon = element.Q<VisualElement>("item-icon");
        var label = element.Q<Label>("item-label");
        
        icon.style.backgroundImage = new StyleBackground(data.Icon);
        label.text = data.Name;
    }
);

virtualizedList.SetItems(inventoryItems);
```

### 10.2 Lazy Loading

**Lazy Loading Pattern:**
```csharp
using UnityEngine;
using UnityEngine.UIElements;
using System.Collections;
using System.Collections.Generic;

public class LazyLoadController : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;
    [SerializeField] private int _itemsPerPage = 20;
    
    private ScrollView _scrollView;
    private VisualElement _contentContainer;
    private List<ItemData> _allItems;
    private int _currentPage = 0;
    private bool _isLoading = false;
    
    private void OnEnable()
    {
        var root = _uiDocument.rootVisualElement;
        _scrollView = root.Q<ScrollView>("ItemScrollView");
        _contentContainer = root.Q<VisualElement>("ItemContainer");
        
        // Register scroll event
        _scrollView.verticalScroller.valueChanged += OnScrollChanged;
        
        // Load initial data
        LoadPage(0);
    }
    
    private void OnScrollChanged(float scrollPosition)
    {
        // Check if scrolled near bottom
        float scrollableHeight = _scrollView.contentContainer.layout.height - _scrollView.layout.height;
        float scrollPercentage = scrollPosition / scrollableHeight;
        
        if (scrollPercentage > 0.8f && !_isLoading)
        {
            LoadNextPage();
        }
    }
    
    private void LoadNextPage()
    {
        _currentPage++;
        LoadPage(_currentPage);
    }
    
    private void LoadPage(int pageIndex)
    {
        if (_isLoading) return;
        
        _isLoading = true;
        StartCoroutine(LoadPageCoroutine(pageIndex));
    }
    
    private IEnumerator LoadPageCoroutine(int pageIndex)
    {
        // Simulate network delay
        yield return new WaitForSeconds(0.5f);
        
        int startIndex = pageIndex * _itemsPerPage;
        int endIndex = Mathf.Min(startIndex + _itemsPerPage, _allItems.Count);
        
        for (int i = startIndex; i < endIndex; i++)
        {
            var itemElement = CreateItemElement(_allItems[i]);
            _contentContainer.Add(itemElement);
            
            // Yield periodically to prevent frame drops
            if (i % 5 == 0)
            {
                yield return null;
            }
        }
        
        _isLoading = false;
    }
    
    private VisualElement CreateItemElement(ItemData data)
    {
        // Create and return item element
        return new VisualElement();
    }
}
```

### 10.3 Memory Management

**UI Element Pool:**
```csharp
using UnityEngine.UIElements;
using System.Collections.Generic;

public class UIElementPool<T> where T : VisualElement, new()
{
    private Stack<T> _pool = new Stack<T>();
    private HashSet<T> _activeElements = new HashSet<T>();
    private VisualElement _container;
    private int _maxPoolSize;
    
    public UIElementPool(VisualElement container, int initialSize = 10, int maxSize = 100)
    {
        _container = container;
        _maxPoolSize = maxSize;
        
        // Pre-populate pool
        for (int i = 0; i < initialSize; i++)
        {
            var element = CreateNewElement();
            element.style.display = DisplayStyle.None;
            _pool.Push(element);
        }
    }
    
    private T CreateNewElement()
    {
        var element = new T();
        _container.Add(element);
        return element;
    }
    
    public T Get()
    {
        T element;
        
        if (_pool.Count > 0)
        {
            element = _pool.Pop();
        }
        else
        {
            element = CreateNewElement();
        }
        
        element.style.display = DisplayStyle.Flex;
        _activeElements.Add(element);
        
        return element;
    }
    
    public void Return(T element)
    {
        if (!_activeElements.Contains(element))
        {
            return; // Element not from this pool
        }
        
        element.style.display = DisplayStyle.None;
        _activeElements.Remove(element);
        
        // Only return to pool if under max size
        if (_pool.Count < _maxPoolSize)
        {
            _pool.Push(element);
        }
        else
        {
            // Remove from hierarchy to free memory
            element.RemoveFromHierarchy();
        }
    }
    
    public void ReturnAll()
    {
        foreach (var element in _activeElements)
        {
            element.style.display = DisplayStyle.None;
            
            if (_pool.Count < _maxPoolSize)
            {
                _pool.Push(element);
            }
            else
            {
                element.RemoveFromHierarchy();
            }
        }
        
        _activeElements.Clear();
    }
    
    public void Clear()
    {
        ReturnAll();
        
        while (_pool.Count > 0)
        {
            var element = _pool.Pop();
            element.RemoveFromHierarchy();
        }
    }
}
```

---

## 11. Production Patterns

### 11.1 Dependency Injection

**Simple Service Locator:**
```csharp
using System;
using System.Collections.Generic;

public class ServiceLocator
{
    private static ServiceLocator _instance;
    public static ServiceLocator Instance => _instance ?? (_instance = new ServiceLocator());
    
    private Dictionary<Type, object> _services = new Dictionary<Type, object>();
    
    public void Register<T>(T service)
    {
        var type = typeof(T);
        
        if (_services.ContainsKey(type))
        {
            _services[type] = service;
        }
        else
        {
            _services.Add(type, service);
        }
    }
    
    public T Get<T>()
    {
        var type = typeof(T);
        
        if (_services.TryGetValue(type, out var service))
        {
            return (T)service;
        }
        
        throw new Exception($"Service of type {type} not registered!");
    }
    
    public bool TryGet<T>(out T service)
    {
        var type = typeof(T);
        
        if (_services.TryGetValue(type, out var obj))
        {
            service = (T)obj;
            return true;
        }
        
        service = default;
        return false;
    }
    
    public void Clear()
    {
        _services.Clear();
    }
}

// Usage:
public class GameBootstrap : MonoBehaviour
{
    private void Awake()
    {
        // Register services
        ServiceLocator.Instance.Register<IAudioService>(new AudioService());
        ServiceLocator.Instance.Register<ISceneLoader>(new SceneLoader());
        ServiceLocator.Instance.Register<IPlayerData>(new PlayerData());
    }
}

public class MainMenuController : MonoBehaviour
{
    private IAudioService _audioService;
    private ISceneLoader _sceneLoader;
    
    private void Awake()
    {
        // Resolve dependencies
        _audioService = ServiceLocator.Instance.Get<IAudioService>();
        _sceneLoader = ServiceLocator.Instance.Get<ISceneLoader>();
    }
}
```

### 11.2 Event Bus Pattern

**Global Event Bus:**
```csharp
using System;
using System.Collections.Generic;

public class EventBus
{
    private static EventBus _instance;
    public static EventBus Instance => _instance ?? (_instance = new EventBus());
    
    private Dictionary<Type, Delegate> _eventHandlers = new Dictionary<Type, Delegate>();
    
    public void Subscribe<T>(Action<T> handler)
    {
        var eventType = typeof(T);
        
        if (_eventHandlers.TryGetValue(eventType, out var existingHandler))
        {
            _eventHandlers[eventType] = Delegate.Combine(existingHandler, handler);
        }
        else
        {
            _eventHandlers[eventType] = handler;
        }
    }
    
    public void Unsubscribe<T>(Action<T> handler)
    {
        var eventType = typeof(T);
        
        if (_eventHandlers.TryGetValue(eventType, out var existingHandler))
        {
            var newHandler = Delegate.Remove(existingHandler, handler);
            
            if (newHandler == null)
            {
                _eventHandlers.Remove(eventType);
            }
            else
            {
                _eventHandlers[eventType] = newHandler;
            }
        }
    }
    
    public void Publish<T>(T eventData)
    {
        var eventType = typeof(T);
        
        if (_eventHandlers.TryGetValue(eventType, out var handler))
        {
            (handler as Action<T>)?.Invoke(eventData);
        }
    }
    
    public void Clear()
    {
        _eventHandlers.Clear();
    }
}

// Event definitions
public class PlayerHealthChangedEvent
{
    public int NewHealth { get; set; }
    public int MaxHealth { get; set; }
}

public class ScoreUpdatedEvent
{
    public int NewScore { get; set; }
}

// Usage:
public class PlayerController : MonoBehaviour
{
    private void TakeDamage(int damage)
    {
        health -= damage;
        
        // Publish event
        EventBus.Instance.Publish(new PlayerHealthChangedEvent
        {
            NewHealth = health,
            MaxHealth = maxHealth
        });
    }
}

public class HUDController : MonoBehaviour
{
    private void OnEnable()
    {
        // Subscribe to events
        EventBus.Instance.Subscribe<PlayerHealthChangedEvent>(OnHealthChanged);
        EventBus.Instance.Subscribe<ScoreUpdatedEvent>(OnScoreUpdated);
    }
    
    private void OnDisable()
    {
        // Unsubscribe
        EventBus.Instance.Unsubscribe<PlayerHealthChangedEvent>(OnHealthChanged);
        EventBus.Instance.Unsubscribe<ScoreUpdatedEvent>(OnScoreUpdated);
    }
    
    private void OnHealthChanged(PlayerHealthChangedEvent evt)
    {
        UpdateHealthDisplay(evt.NewHealth, evt.MaxHealth);
    }
    
    private void OnScoreUpdated(ScoreUpdatedEvent evt)
    {
        UpdateScoreDisplay(evt.NewScore);
    }
}
```

### 11.3 Localization System

**Localization Manager:**
```csharp
using UnityEngine;
using System.Collections.Generic;

public class LocalizationManager : MonoBehaviour
{
    private static LocalizationManager _instance;
    public static LocalizationManager Instance => _instance;
    
    [SerializeField] private SystemLanguage _currentLanguage = SystemLanguage.English;
    
    private Dictionary<string, Dictionary<SystemLanguage, string>> _localizedStrings 
        = new Dictionary<string, Dictionary<SystemLanguage, string>>();
    
    public event System.Action<SystemLanguage> LanguageChanged;
    
    public SystemLanguage CurrentLanguage => _currentLanguage;
    
    private void Awake()
    {
        if (_instance != null && _instance != this)
        {
            Destroy(gameObject);
            return;
        }
        
        _instance = this;
        DontDestroyOnLoad(gameObject);
        
        LoadLocalizations();
    }
    
    private void LoadLocalizations()
    {
        // Load from JSON, CSV, or ScriptableObjects
        // Example structure:
        var locData = new Dictionary<string, Dictionary<SystemLanguage, string>>
        {
            ["menu.play"] = new Dictionary<SystemLanguage, string>
            {
                [SystemLanguage.English] = "Play",
                [SystemLanguage.Spanish] = "Jugar",
                [SystemLanguage.French] = "Jouer"
            },
            ["menu.settings"] = new Dictionary<SystemLanguage, string>
            {
                [SystemLanguage.English] = "Settings",
                [SystemLanguage.Spanish] = "Ajustes",
                [SystemLanguage.French] = "Paramètres"
            }
        };
        
        _localizedStrings = locData;
    }
    
    public string GetString(string key)
    {
        if (_localizedStrings.TryGetValue(key, out var translations))
        {
            if (translations.TryGetValue(_currentLanguage, out var translation))
            {
                return translation;
            }
            
            // Fallback to English
            if (translations.TryGetValue(SystemLanguage.English, out var fallback))
            {
                return fallback;
            }
        }
        
        return $"[{key}]"; // Return key if not found
    }
    
    public void SetLanguage(SystemLanguage language)
    {
        if (_currentLanguage != language)
        {
            _currentLanguage = language;
            LanguageChanged?.Invoke(_currentLanguage);
        }
    }
}
```

**Localized Label Component:**
```csharp
using UnityEngine;
using UnityEngine.UIElements;

public class LocalizedLabel : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;
    [SerializeField] private string _labelName;
    [SerializeField] private string _localizationKey;
    
    private Label _label;
    
    private void OnEnable()
    {
        _label = _uiDocument.rootVisualElement.Q<Label>(_labelName);
        
        // Subscribe to language changes
        LocalizationManager.Instance.LanguageChanged += OnLanguageChanged;
        
        // Set initial text
        UpdateText();
    }
    
    private void OnDisable()
    {
        LocalizationManager.Instance.LanguageChanged -= OnLanguageChanged;
    }
    
    private void OnLanguageChanged(SystemLanguage newLanguage)
    {
        UpdateText();
    }
    
    private void UpdateText()
    {
        if (_label != null)
        {
            _label.text = LocalizationManager.Instance.GetString(_localizationKey);
        }
    }
}
```

---

## Summary: Industry Best Practices

### Key Takeaways

1. **Architecture**: Use MVVM for UI Toolkit projects
2. **Code-First**: Generate UI programmatically with factory patterns
3. **Separation**: Keep logic in ViewModels, visuals in UXML/USS
4. **Data Binding**: Use Unity 6 native binding or observable patterns
5. **Team Workflow**: Component libraries + style guides
6. **Performance**: Virtualization, pooling, lazy loading
7. **Testing**: Unit test ViewModels, integration test views
8. **Production**: DI, event bus, localization

### Comparison: Your Past Approach vs. Industry Standard

| Aspect | Factory-View-Host-Controller | Industry MVVM Pattern |
|--------|------------------------------|----------------------|
| **Scalability** | Good for small teams | Excellent for large teams |
| **Testability** | Moderate | High (ViewModels easily tested) |
| **Designer Collaboration** | Limited | Excellent (UXML/USS separation) |
| **Code Reusability** | Component-based | Component + data binding |
| **Learning Curve** | Low | Medium |
| **Unity 6 Features** | Limited use | Full data binding support |

### Recommended Transition Path

**Phase 1**: Introduce ViewModels
- Keep your factory pattern
- Add ViewModel layer between controllers and views
- Start using observable properties

**Phase 2**: Adopt Data Binding
- Convert to UXML templates for structure
- Use USS for all styling
- Implement Unity 6 native binding

**Phase 3**: Full MVVM
- Refactor controllers to presenters
- Component library for team
- Automated testing

---

*This document provides production-ready patterns for scaling UI Toolkit projects across teams while maintaining code quality and performance.*
