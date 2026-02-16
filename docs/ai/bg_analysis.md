# Unity Bagel Game - Complete UI Toolkit Analysis

> **Project**: Unity BagelGame Sample (Unity 6.3 LTS)  
> **Repository**: [Unity-Technologies/BagelGame](https://github.com/Unity-Technologies/BagelGame)  
> **Purpose**: Full vertical game demonstrating UI Toolkit for 3D games

## Table of Contents
1. [Project Overview](#project-overview)
2. [UI Toolkit Architecture](#ui-toolkit-architecture)
3. [Data Binding System](#data-binding-system)
4. [Shader Graph Integration](#shader-graph-integration)
5. [World Space UI](#world-space-ui)
6. [Theme System](#theme-system)
7. [Input System Integration](#input-system-integration)
8. [Performance Patterns](#performance-patterns)
9. [Code Examples](#code-examples)
10. [Quest 2 VR Adaptation](#quest-2-vr-adaptation)

---

## 1. Project Overview

### What Bagel Game Demonstrates

**Core Features**:
- **World Space UI**: UI canvases positioned in 3D game space
- **Custom Shaders**: Shader Graph integration with UI Toolkit
- **Data Bindings**: Runtime data synchronization (Unity 6 feature)
- **StyleSheets (USS)**: Dynamic theming and styling
- **Input System**: New Input System integration
- **UXML Templates**: Reusable UI component templates

**Game Concept**: Physics-based bagel navigation game where UI is integrated into the 3D world rather than overlaid on screen.

### Technology Stack

```
UI Rendering:
├── UI Toolkit (Runtime)
│   ├── UXML (Structure)
│   ├── USS (Styling)
│   └── C# (Logic)
├── Shader Graph (Visual Effects)
└── Unity Input System (Control)
```

---

## 2. UI Toolkit Architecture

### 2.1 Document Structure (UXML)

UI Toolkit uses UXML (Unity XML) for declarative UI structure, similar to HTML.

**Example: Main Menu Structure**
```xml
<ui:UXML xmlns:ui="UnityEngine.UIElements" 
         xmlns:uie="UnityEditor.UIElements">
    
    <!-- Root container -->
    <ui:VisualElement name="MainMenu" class="menu-root">
        
        <!-- Header section -->
        <ui:VisualElement name="Header" class="menu-header">
            <ui:Label text="BAGEL GAME" class="title-text" />
        </ui:VisualElement>
        
        <!-- Button container -->
        <ui:VisualElement name="ButtonContainer" class="button-group">
            <ui:Button name="PlayButton" text="Play" class="menu-button" />
            <ui:Button name="SettingsButton" text="Settings" class="menu-button" />
            <ui:Button name="QuitButton" text="Quit" class="menu-button" />
        </ui:VisualElement>
        
        <!-- Game stats display -->
        <ui:VisualElement name="StatsPanel" class="stats-panel">
            <ui:Label name="ScoreLabel" binding-path="CurrentScore" class="stat-value" />
            <ui:Label name="ToppingsLabel" binding-path="ToppingsRemaining" class="stat-value" />
        </ui:VisualElement>
        
    </ui:VisualElement>
</ui:UXML>
```

**Key UXML Concepts**:
- `class` attribute links to USS styles
- `binding-path` enables data binding
- `name` provides C# script access
- Nested hierarchy creates parent-child relationships

### 2.2 Stylesheet System (USS)

USS is Unity's CSS-like styling language for UI Toolkit.

**Example: Premium Modern Styling**
```css
/* ===== ROOT VARIABLES (THEMING) ===== */
:root {
    /* Color palette */
    --primary-color: rgb(138, 43, 226);
    --secondary-color: rgb(0, 217, 255);
    --bg-dark: rgb(10, 10, 15);
    --bg-mid: rgb(20, 20, 30);
    
    /* Gradient colors */
    --glow-purple: rgba(138, 43, 226, 0.4);
    --glow-cyan: rgba(0, 217, 255, 0.3);
    --glow-pink: rgba(255, 71, 133, 0.25);
    
    /* Spacing */
    --spacing-small: 8px;
    --spacing-medium: 16px;
    --spacing-large: 32px;
    
    /* Animation */
    --transition-speed: 0.3s;
}

/* ===== LAYOUT CONTAINERS ===== */
.menu-root {
    flex-grow: 1;
    background-image: 
        radial-gradient(circle at 20% 30%, var(--glow-purple) 0%, transparent 50%),
        radial-gradient(circle at 80% 70%, var(--glow-cyan) 0%, transparent 50%),
        radial-gradient(ellipse at 50% 100%, var(--bg-mid) 0%, var(--bg-dark) 100%);
    padding: var(--spacing-large);
}

.menu-header {
    align-items: center;
    justify-content: center;
    margin-bottom: var(--spacing-large);
}

/* ===== TEXT STYLES ===== */
.title-text {
    font-size: 72px;
    color: rgb(255, 255, 255);
    -unity-font-style: bold;
    text-shadow: 0px 4px 20px var(--glow-cyan);
    -unity-text-align: middle-center;
}

.stat-value {
    font-size: 24px;
    color: var(--secondary-color);
    -unity-font-style: bold;
}

/* ===== BUTTON STYLES ===== */
.menu-button {
    min-width: 200px;
    min-height: 60px;
    margin: var(--spacing-small);
    
    /* Modern glass morphism effect */
    background-color: rgba(255, 255, 255, 0.05);
    border-width: 2px;
    border-color: rgba(255, 255, 255, 0.1);
    border-radius: 12px;
    
    /* Text styling */
    font-size: 20px;
    color: rgb(255, 255, 255);
    -unity-font-style: bold;
    
    /* Transition */
    transition-property: background-color, border-color, scale;
    transition-duration: var(--transition-speed);
}

/* Button hover state */
.menu-button:hover {
    background-color: rgba(255, 255, 255, 0.12);
    border-color: var(--secondary-color);
    scale: 1.05;
}

/* Button active/pressed state */
.menu-button:active {
    background-color: rgba(0, 217, 255, 0.2);
    scale: 0.98;
}

/* ===== STATS PANEL ===== */
.stats-panel {
    flex-direction: row;
    justify-content: space-between;
    padding: var(--spacing-medium);
    
    background-image: radial-gradient(
        circle at 50% 50%, 
        rgba(255, 255, 255, 0.08), 
        transparent
    );
    border-radius: 16px;
    border-width: 1px;
    border-color: rgba(255, 255, 255, 0.1);
}
```

**USS Features Used**:
- CSS variables (`:root`)
- Radial gradients for modern web aesthetic
- Flexbox layout (`flex-direction`, `justify-content`)
- Pseudo-classes (`:hover`, `:active`)
- Transitions for smooth animations
- Border radius for rounded corners

### 2.3 C# Document Controller

**Example: MenuController.cs**
```csharp
using UnityEngine;
using UnityEngine.UIElements;

public class MenuController : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;
    
    // Root visual element
    private VisualElement _root;
    
    // UI element references
    private Button _playButton;
    private Button _settingsButton;
    private Button _quitButton;
    
    private Label _scoreLabel;
    private Label _toppingsLabel;
    
    // Game data reference
    [SerializeField] private GameData _gameData;
    
    private void OnEnable()
    {
        // Get root from UI Document
        _root = _uiDocument.rootVisualElement;
        
        // Query elements by name
        _playButton = _root.Q<Button>("PlayButton");
        _settingsButton = _root.Q<Button>("SettingsButton");
        _quitButton = _root.Q<Button>("QuitButton");
        
        _scoreLabel = _root.Q<Label>("ScoreLabel");
        _toppingsLabel = _root.Q<Label>("ToppingsLabel");
        
        // Register button callbacks
        _playButton.clicked += OnPlayClicked;
        _settingsButton.clicked += OnSettingsClicked;
        _quitButton.clicked += OnQuitClicked;
        
        // Setup data binding
        SetupDataBinding();
    }
    
    private void OnDisable()
    {
        // Unregister callbacks to prevent memory leaks
        _playButton.clicked -= OnPlayClicked;
        _settingsButton.clicked -= OnSettingsClicked;
        _quitButton.clicked -= OnQuitClicked;
    }
    
    private void SetupDataBinding()
    {
        // Bind to serialized object (Unity 6 feature)
        var so = new SerializedObject(_gameData);
        _root.Bind(so);
    }
    
    private void OnPlayClicked()
    {
        Debug.Log("Play button clicked!");
        // Load game scene
    }
    
    private void OnSettingsClicked()
    {
        Debug.Log("Settings button clicked!");
        // Show settings panel
    }
    
    private void OnQuitClicked()
    {
        Application.Quit();
    }
}
```

**Key C# Patterns**:
- `UIDocument` component provides access to UXML
- `Q<T>()` queries elements by name and type
- Event registration with `clicked +=`
- Proper cleanup in `OnDisable()`

---

## 3. Data Binding System

### 3.1 Unity 6 Runtime Data Binding

Unity 6 introduces runtime data binding for UI Toolkit, allowing automatic synchronization between data and UI.

**Example: Game Data ScriptableObject**
```csharp
using UnityEngine;

[CreateAssetMenu(fileName = "GameData", menuName = "Game/GameData")]
public class GameData : ScriptableObject
{
    [SerializeField] private int _currentScore;
    [SerializeField] private int _toppingsRemaining;
    [SerializeField] private float _timeRemaining;
    [SerializeField] private bool _isPaused;
    
    // Properties with change notifications
    public int CurrentScore
    {
        get => _currentScore;
        set
        {
            if (_currentScore != value)
            {
                _currentScore = value;
                // Serialization system will notify bound UI
            }
        }
    }
    
    public int ToppingsRemaining
    {
        get => _toppingsRemaining;
        set
        {
            if (_toppingsRemaining != value)
            {
                _toppingsRemaining = value;
            }
        }
    }
    
    public float TimeRemaining
    {
        get => _timeRemaining;
        set
        {
            if (!Mathf.Approximately(_timeRemaining, value))
            {
                _timeRemaining = value;
            }
        }
    }
    
    public bool IsPaused
    {
        get => _isPaused;
        set
        {
            if (_isPaused != value)
            {
                _isPaused = value;
            }
        }
    }
}
```

### 3.2 Binding in UXML

```xml
<ui:UXML xmlns:ui="UnityEngine.UIElements">
    <ui:VisualElement name="HUD">
        
        <!-- Direct binding to serialized property -->
        <ui:Label name="ScoreLabel" 
                  binding-path="CurrentScore" 
                  class="hud-value" />
        
        <!-- Binding with format string -->
        <ui:Label name="ToppingsLabel" 
                  binding-path="ToppingsRemaining"
                  class="hud-value" />
        
        <!-- Progress bar binding -->
        <ui:ProgressBar name="TimeBar" 
                        binding-path="TimeRemaining"
                        low-value="0"
                        high-value="100"
                        class="time-progress" />
        
        <!-- Toggle binding -->
        <ui:Toggle name="PauseToggle" 
                   binding-path="IsPaused"
                   label="Paused" />
        
    </ui:VisualElement>
</ui:UXML>
```

### 3.3 Advanced Binding with Value Converters

**Example: Custom Value Converter**
```csharp
using UnityEngine;
using UnityEngine.UIElements;

// Convert score int to formatted string
public class ScoreToStringConverter : UxmlAttributeConverter<int, string>
{
    public override string FromAttribute(int value)
    {
        return $"Score: {value:N0}";
    }
}

// Convert time to MM:SS format
public class TimeToStringConverter : UxmlAttributeConverter<float, string>
{
    public override string FromAttribute(float value)
    {
        int minutes = Mathf.FloorToInt(value / 60f);
        int seconds = Mathf.FloorToInt(value % 60f);
        return $"{minutes:00}:{seconds:00}";
    }
}
```

**Using Converters in C#**
```csharp
public class HUDController : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;
    [SerializeField] private GameData _gameData;
    
    private Label _scoreLabel;
    private Label _timeLabel;
    
    private void OnEnable()
    {
        var root = _uiDocument.rootVisualElement;
        
        // Get labels
        _scoreLabel = root.Q<Label>("ScoreLabel");
        _timeLabel = root.Q<Label>("TimeLabel");
        
        // Setup binding with SerializedObject
        var so = new SerializedObject(_gameData);
        
        // Bind with property tracking
        root.TrackPropertyValue(so.FindProperty("_currentScore"), property =>
        {
            _scoreLabel.text = $"Score: {_gameData.CurrentScore:N0}";
        });
        
        root.TrackPropertyValue(so.FindProperty("_timeRemaining"), property =>
        {
            int minutes = Mathf.FloorToInt(_gameData.TimeRemaining / 60f);
            int seconds = Mathf.FloorToInt(_gameData.TimeRemaining % 60f);
            _timeLabel.text = $"{minutes:00}:{seconds:00}";
        });
        
        // Bind entire object
        root.Bind(so);
    }
}
```

### 3.4 Two-Way Binding Pattern

**Example: Settings Panel with Two-Way Binding**
```csharp
using UnityEngine;
using UnityEngine.UIElements;

public class SettingsController : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;
    [SerializeField] private GameSettings _settings;
    
    private Slider _volumeSlider;
    private Slider _sensitivitySlider;
    private Toggle _invertYToggle;
    
    private void OnEnable()
    {
        var root = _uiDocument.rootVisualElement;
        
        // Get elements
        _volumeSlider = root.Q<Slider>("VolumeSlider");
        _sensitivitySlider = root.Q<Slider>("SensitivitySlider");
        _invertYToggle = root.Q<Toggle>("InvertYToggle");
        
        // Bind to SerializedObject for automatic two-way binding
        var so = new SerializedObject(_settings);
        root.Bind(so);
        
        // Optional: Manual event handling for immediate feedback
        _volumeSlider.RegisterValueChangedCallback(evt =>
        {
            AudioListener.volume = evt.newValue;
        });
    }
}
```

**UXML for Settings**
```xml
<ui:UXML xmlns:ui="UnityEngine.UIElements">
    <ui:VisualElement name="SettingsPanel" class="settings-container">
        
        <ui:Slider name="VolumeSlider" 
                   binding-path="_volume"
                   low-value="0" 
                   high-value="1"
                   label="Volume"
                   class="setting-slider" />
        
        <ui:Slider name="SensitivitySlider" 
                   binding-path="_sensitivity"
                   low-value="0.1" 
                   high-value="2.0"
                   label="Sensitivity"
                   class="setting-slider" />
        
        <ui:Toggle name="InvertYToggle" 
                   binding-path="_invertY"
                   label="Invert Y-Axis"
                   class="setting-toggle" />
        
    </ui:VisualElement>
</ui:UXML>
```

---

## 4. Shader Graph Integration

### 4.1 Custom Shader Graph for UI

UI Toolkit can use Shader Graph shaders applied via custom materials.

**Example: Glass Morphism Shader Setup**

**Shader Graph Node Structure**:
```
Inputs:
├── Vertex Color (from UI element)
├── UV0 (texture coordinates)
├── Scene Color (for blur effect)
└── Custom Properties (blur intensity, tint color)

Processing:
├── Scene Color Blur (spiral sampling)
├── Gradient Overlay
├── Border Detection
└── Alpha Blending

Output:
└── Fragment (Base Color + Alpha)
```

**C# Material Controller**
```csharp
using UnityEngine;
using UnityEngine.UIElements;

public class GlassPanelController : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;
    [SerializeField] private Material _glassMaterial;
    
    private VisualElement _glassPanel;
    
    // Shader property IDs (cached for performance)
    private static readonly int BlurIntensity = Shader.PropertyToID("_BlurIntensity");
    private static readonly int TintColor = Shader.PropertyToID("_TintColor");
    private static readonly int BorderWidth = Shader.PropertyToID("_BorderWidth");
    
    private void OnEnable()
    {
        var root = _uiDocument.rootVisualElement;
        _glassPanel = root.Q<VisualElement>("GlassPanel");
        
        // Apply custom material to element
        ApplyGlassMaterial();
    }
    
    private void ApplyGlassMaterial()
    {
        // Create material instance (important for runtime modification)
        var materialInstance = new Material(_glassMaterial);
        
        // Set initial properties
        materialInstance.SetFloat(BlurIntensity, 0.5f);
        materialInstance.SetColor(TintColor, new Color(1f, 1f, 1f, 0.1f));
        materialInstance.SetFloat(BorderWidth, 2f);
        
        // Apply to visual element via inline style
        _glassPanel.style.unityBackgroundImageTintColor = Color.white;
        
        // For custom shader, use experimental API
        _glassPanel.experimental.SetMaterial(materialInstance);
    }
    
    // Runtime shader parameter update
    public void SetBlurIntensity(float intensity)
    {
        var material = _glassPanel.experimental.GetMaterial();
        if (material != null)
        {
            material.SetFloat(BlurIntensity, intensity);
        }
    }
}
```

### 4.2 Procedural UI Shapes with Shader Graph

**Example: Rounded Rectangle Button Shader**

**Shader Properties**:
```hlsl
// Properties exposed in Shader Graph
Properties
{
    _MainColor ("Main Color", Color) = (1, 1, 1, 1)
    _BorderColor ("Border Color", Color) = (0.8, 0.8, 0.8, 1)
    _BorderWidth ("Border Width", Float) = 2.0
    _CornerRadius ("Corner Radius", Float) = 12.0
    _GlowIntensity ("Glow Intensity", Float) = 0.0
}
```

**USS Integration**
```css
.glass-panel {
    width: 400px;
    height: 300px;
    
    /* Material application happens in C# */
    /* USS handles layout and base properties */
    border-radius: 16px;
    padding: 20px;
}

/* Hover effect triggers shader parameter change */
.glass-panel:hover {
    /* Handled via C# callback */
}
```

### 4.3 Animated Shader Effects

**Example: Pulsing Glow Effect**
```csharp
using UnityEngine;
using UnityEngine.UIElements;

public class AnimatedGlowController : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;
    [SerializeField] private Material _glowMaterial;
    
    private VisualElement _glowElement;
    private Material _materialInstance;
    
    private static readonly int GlowIntensity = Shader.PropertyToID("_GlowIntensity");
    
    [SerializeField] private float _pulseSpeed = 2f;
    [SerializeField] private float _minIntensity = 0.2f;
    [SerializeField] private float _maxIntensity = 1.0f;
    
    private void OnEnable()
    {
        var root = _uiDocument.rootVisualElement;
        _glowElement = root.Q<VisualElement>("GlowButton");
        
        _materialInstance = new Material(_glowMaterial);
        _glowElement.experimental.SetMaterial(_materialInstance);
    }
    
    private void Update()
    {
        if (_materialInstance != null)
        {
            // Pulsing animation using sine wave
            float intensity = Mathf.Lerp(
                _minIntensity,
                _maxIntensity,
                (Mathf.Sin(Time.time * _pulseSpeed) + 1f) * 0.5f
            );
            
            _materialInstance.SetFloat(GlowIntensity, intensity);
        }
    }
    
    private void OnDisable()
    {
        // Clean up material instance
        if (_materialInstance != null)
        {
            Destroy(_materialInstance);
        }
    }
}
```

---

## 5. World Space UI

### 5.1 Panel Settings for World Space

**Example: WorldSpaceUISetup.cs**
```csharp
using UnityEngine;
using UnityEngine.UIElements;

[RequireComponent(typeof(UIDocument))]
public class WorldSpaceUISetup : MonoBehaviour
{
    [SerializeField] private Camera _targetCamera;
    [SerializeField] private float _distanceFromCamera = 5f;
    [SerializeField] private Vector2 _panelSize = new Vector2(1920, 1080);
    
    private UIDocument _uiDocument;
    private PanelSettings _panelSettings;
    
    private void Awake()
    {
        _uiDocument = GetComponent<UIDocument>();
        
        // Create PanelSettings asset at runtime
        SetupWorldSpacePanel();
    }
    
    private void SetupWorldSpacePanel()
    {
        // Create new PanelSettings
        _panelSettings = ScriptableObject.CreateInstance<PanelSettings>();
        
        // Configure for world space
        _panelSettings.targetDisplay = 0;
        _panelSettings.clearDepthStencil = false;
        _panelSettings.sortingOrder = 0;
        
        // Set render mode to World Space
        _panelSettings.SetScreenToPanelSpaceFunction((screenPosition) =>
        {
            // Convert screen position to world space panel position
            Ray ray = _targetCamera.ScreenPointToRay(screenPosition);
            Plane panelPlane = new Plane(-transform.forward, transform.position);
            
            if (panelPlane.Raycast(ray, out float distance))
            {
                Vector3 worldPoint = ray.GetPoint(distance);
                Vector3 localPoint = transform.InverseTransformPoint(worldPoint);
                
                // Convert to panel coordinates
                float u = (localPoint.x / transform.localScale.x + 0.5f) * _panelSize.x;
                float v = (-localPoint.y / transform.localScale.y + 0.5f) * _panelSize.y;
                
                return new Vector2(u, v);
            }
            
            return screenPosition;
        });
        
        // Apply to UI Document
        _uiDocument.panelSettings = _panelSettings;
    }
    
    private void Update()
    {
        // Optional: Make panel face camera
        transform.LookAt(_targetCamera.transform);
        transform.Rotate(0, 180, 0); // Flip to face camera
    }
}
```

### 5.2 Quest 2 VR World Space UI

**Example: VR UI Panel for Meta Quest**
```csharp
using UnityEngine;
using UnityEngine.UIElements;

public class VRWorldSpaceUI : MonoBehaviour
{
    [Header("VR Settings")]
    [SerializeField] private Camera _vrCamera; // Center eye camera
    [SerializeField] private float _uiDistance = 2.5f;
    [SerializeField] private float _uiScale = 0.001f; // Scaling for comfortable viewing
    
    [Header("UI Settings")]
    [SerializeField] private UIDocument _uiDocument;
    [SerializeField] private Vector2 _panelResolution = new Vector2(1920, 1080);
    
    private PanelSettings _panelSettings;
    private RenderTexture _renderTexture;
    
    private void Awake()
    {
        SetupVRPanel();
    }
    
    private void SetupVRPanel()
    {
        // Create render texture for UI
        _renderTexture = new RenderTexture(
            (int)_panelResolution.x,
            (int)_panelResolution.y,
            0,
            RenderTextureFormat.ARGB32
        );
        _renderTexture.Create();
        
        // Create panel settings
        _panelSettings = ScriptableObject.CreateInstance<PanelSettings>();
        _panelSettings.targetTexture = _renderTexture;
        _panelSettings.clearColor = Color.clear;
        
        // Apply to UI Document
        _uiDocument.panelSettings = _panelSettings;
        
        // Create quad mesh for displaying UI in world
        CreateUIQuad();
    }
    
    private void CreateUIQuad()
    {
        // Create quad mesh
        GameObject quadObject = new GameObject("UI_Quad");
        quadObject.transform.SetParent(transform);
        quadObject.transform.localPosition = Vector3.zero;
        quadObject.transform.localRotation = Quaternion.identity;
        
        // Add mesh components
        MeshFilter meshFilter = quadObject.AddComponent<MeshFilter>();
        MeshRenderer meshRenderer = quadObject.AddComponent<MeshRenderer>();
        
        // Create quad mesh
        float aspectRatio = _panelResolution.x / _panelResolution.y;
        float width = aspectRatio * _uiScale;
        float height = _uiScale;
        
        Mesh quadMesh = new Mesh
        {
            vertices = new[]
            {
                new Vector3(-width * 0.5f, -height * 0.5f, 0),
                new Vector3(width * 0.5f, -height * 0.5f, 0),
                new Vector3(width * 0.5f, height * 0.5f, 0),
                new Vector3(-width * 0.5f, height * 0.5f, 0)
            },
            uv = new[]
            {
                new Vector2(0, 0),
                new Vector2(1, 0),
                new Vector2(1, 1),
                new Vector2(0, 1)
            },
            triangles = new[] { 0, 2, 1, 0, 3, 2 }
        };
        
        quadMesh.RecalculateNormals();
        meshFilter.mesh = quadMesh;
        
        // Create material with UI texture
        Material uiMaterial = new Material(Shader.Find("UI/Default"));
        uiMaterial.mainTexture = _renderTexture;
        meshRenderer.material = uiMaterial;
    }
    
    private void Update()
    {
        // Position UI in front of camera
        Vector3 targetPosition = _vrCamera.transform.position + 
                                 _vrCamera.transform.forward * _uiDistance;
        transform.position = Vector3.Lerp(transform.position, targetPosition, Time.deltaTime * 5f);
        
        // Face camera
        transform.LookAt(_vrCamera.transform);
        transform.Rotate(0, 180, 0);
    }
}
```

### 5.3 VR Interaction with UI Toolkit

**Example: Ray Interactor for UI**
```csharp
using UnityEngine;
using UnityEngine.UIElements;
using UnityEngine.InputSystem;

public class VRUIRayInteractor : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;
    [SerializeField] private Transform _rayOrigin; // Controller transform
    [SerializeField] private LineRenderer _lineRenderer;
    [SerializeField] private float _maxRayDistance = 10f;
    
    [Header("Input Actions")]
    [SerializeField] private InputActionReference _selectAction;
    
    private VisualElement _currentHoveredElement;
    private Vector2 _currentPanelPosition;
    
    private void OnEnable()
    {
        _selectAction.action.performed += OnSelectPerformed;
    }
    
    private void OnDisable()
    {
        _selectAction.action.performed -= OnSelectPerformed;
    }
    
    private void Update()
    {
        UpdateRaycast();
    }
    
    private void UpdateRaycast()
    {
        Ray ray = new Ray(_rayOrigin.position, _rayOrigin.forward);
        
        // Raycast against UI plane
        Plane uiPlane = new Plane(-transform.forward, transform.position);
        
        if (uiPlane.Raycast(ray, out float distance) && distance <= _maxRayDistance)
        {
            Vector3 hitPoint = ray.GetPoint(distance);
            
            // Update line renderer
            _lineRenderer.SetPosition(0, _rayOrigin.position);
            _lineRenderer.SetPosition(1, hitPoint);
            
            // Convert world hit to panel coordinates
            Vector3 localHit = transform.InverseTransformPoint(hitPoint);
            _currentPanelPosition = WorldToPanel(localHit);
            
            // Send pointer event to UI
            SimulatePointerMove(_currentPanelPosition);
        }
        else
        {
            // No hit - reset
            _lineRenderer.SetPosition(1, ray.GetPoint(_maxRayDistance));
            SimulatePointerExit();
        }
    }
    
    private Vector2 WorldToPanel(Vector3 localPosition)
    {
        var panelSettings = _uiDocument.panelSettings;
        
        // Convert local position to panel coordinates
        float u = (localPosition.x + 0.5f) * panelSettings.targetTexture.width;
        float v = (-localPosition.y + 0.5f) * panelSettings.targetTexture.height;
        
        return new Vector2(u, v);
    }
    
    private void SimulatePointerMove(Vector2 panelPosition)
    {
        var root = _uiDocument.rootVisualElement;
        var element = root.panel.Pick(panelPosition);
        
        // Handle hover state changes
        if (element != _currentHoveredElement)
        {
            // Exit previous element
            if (_currentHoveredElement != null)
            {
                using (var evt = PointerLeaveEvent.GetPooled())
                {
                    evt.target = _currentHoveredElement;
                    _currentHoveredElement.SendEvent(evt);
                }
            }
            
            // Enter new element
            if (element != null)
            {
                using (var evt = PointerEnterEvent.GetPooled())
                {
                    evt.target = element;
                    element.SendEvent(evt);
                }
            }
            
            _currentHoveredElement = element;
        }
    }
    
    private void SimulatePointerExit()
    {
        if (_currentHoveredElement != null)
        {
            using (var evt = PointerLeaveEvent.GetPooled())
            {
                evt.target = _currentHoveredElement;
                _currentHoveredElement.SendEvent(evt);
            }
            _currentHoveredElement = null;
        }
    }
    
    private void OnSelectPerformed(InputAction.CallbackContext context)
    {
        if (_currentHoveredElement != null)
        {
            // Send click event
            using (var evt = ClickEvent.GetPooled())
            {
                evt.target = _currentHoveredElement;
                _currentHoveredElement.SendEvent(evt);
            }
        }
    }
}
```

---

## 6. Theme System

### 6.1 Dynamic Theme Switching

**Example: ThemeManager.cs**
```csharp
using UnityEngine;
using UnityEngine.UIElements;
using System.Collections.Generic;

[System.Serializable]
public class ThemeColors
{
    public string themeName;
    public Color primaryColor;
    public Color secondaryColor;
    public Color backgroundColor;
    public Color glowColor1;
    public Color glowColor2;
}

public class ThemeManager : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;
    [SerializeField] private List<ThemeColors> _themes;
    [SerializeField] private int _currentThemeIndex = 0;
    
    private VisualElement _root;
    
    private void OnEnable()
    {
        _root = _uiDocument.rootVisualElement;
        ApplyTheme(_currentThemeIndex);
    }
    
    public void ApplyTheme(int themeIndex)
    {
        if (themeIndex < 0 || themeIndex >= _themes.Count)
            return;
        
        _currentThemeIndex = themeIndex;
        var theme = _themes[themeIndex];
        
        // Update CSS variables dynamically
        _root.style.SetCustomProperty("--primary-color", theme.primaryColor);
        _root.style.SetCustomProperty("--secondary-color", theme.secondaryColor);
        _root.style.SetCustomProperty("--bg-dark", theme.backgroundColor);
        _root.style.SetCustomProperty("--glow-color-1", theme.glowColor1);
        _root.style.SetCustomProperty("--glow-color-2", theme.glowColor2);
    }
    
    public void NextTheme()
    {
        int nextIndex = (_currentThemeIndex + 1) % _themes.Count;
        ApplyTheme(nextIndex);
    }
    
    public void SetThemeByName(string themeName)
    {
        int index = _themes.FindIndex(t => t.themeName == themeName);
        if (index >= 0)
        {
            ApplyTheme(index);
        }
    }
}
```

### 6.2 USS Variable Update

**USS with Variables**
```css
:root {
    --primary-color: rgb(138, 43, 226);
    --secondary-color: rgb(0, 217, 255);
    --bg-dark: rgb(10, 10, 15);
    --glow-color-1: rgba(138, 43, 226, 0.4);
    --glow-color-2: rgba(0, 217, 255, 0.3);
}

.themed-button {
    background-color: var(--primary-color);
    border-color: var(--secondary-color);
    transition-duration: 0.3s;
}

.themed-panel {
    background-image: 
        radial-gradient(at 30% 40%, var(--glow-color-1), transparent),
        radial-gradient(at 70% 60%, var(--glow-color-2), transparent),
        var(--bg-dark);
}
```

**Runtime Update Extension**
```csharp
using UnityEngine.UIElements;

public static class StyleExtensions
{
    public static void SetCustomProperty(this IStyle style, string propertyName, Color value)
    {
        // Convert Color to CSS color string
        string colorString = $"rgb({value.r * 255f}, {value.g * 255f}, {value.b * 255f})";
        
        // Update custom property
        // Note: This is a simplified example. Actual implementation may vary.
        style.backgroundColor = value; // Fallback
    }
}
```

---

## 7. Input System Integration

### 7.1 New Input System Setup

**Example: UIInputHandler.cs**
```csharp
using UnityEngine;
using UnityEngine.UIElements;
using UnityEngine.InputSystem;

public class UIInputHandler : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;
    
    [Header("Input Actions")]
    [SerializeField] private InputActionReference _navigateAction;
    [SerializeField] private InputActionReference _submitAction;
    [SerializeField] private InputActionReference _cancelAction;
    
    private VisualElement _root;
    private Focusable _currentFocusedElement;
    
    private void OnEnable()
    {
        _root = _uiDocument.rootVisualElement;
        
        // Enable input actions
        _navigateAction.action.Enable();
        _submitAction.action.Enable();
        _cancelAction.action.Enable();
        
        // Register callbacks
        _navigateAction.action.performed += OnNavigate;
        _submitAction.action.performed += OnSubmit;
        _cancelAction.action.performed += OnCancel;
        
        // Set initial focus
        SetInitialFocus();
    }
    
    private void OnDisable()
    {
        // Unregister callbacks
        _navigateAction.action.performed -= OnNavigate;
        _submitAction.action.performed -= OnSubmit;
        _cancelAction.action.performed -= OnCancel;
        
        // Disable input actions
        _navigateAction.action.Disable();
        _submitAction.action.Disable();
        _cancelAction.action.Disable();
    }
    
    private void SetInitialFocus()
    {
        // Find first focusable element
        var focusable = _root.Q<Button>();
        if (focusable != null)
        {
            focusable.Focus();
            _currentFocusedElement = focusable;
        }
    }
    
    private void OnNavigate(InputAction.CallbackContext context)
    {
        Vector2 direction = context.ReadValue<Vector2>();
        
        if (Mathf.Abs(direction.y) > Mathf.Abs(direction.x))
        {
            // Vertical navigation
            if (direction.y > 0)
                NavigateUp();
            else
                NavigateDown();
        }
        else
        {
            // Horizontal navigation
            if (direction.x > 0)
                NavigateRight();
            else
                NavigateLeft();
        }
    }
    
    private void OnSubmit(InputAction.CallbackContext context)
    {
        if (_currentFocusedElement is Button button)
        {
            // Simulate button click
            using (var evt = ClickEvent.GetPooled())
            {
                evt.target = button;
                button.SendEvent(evt);
            }
        }
    }
    
    private void OnCancel(InputAction.CallbackContext context)
    {
        // Handle back/cancel action
        Debug.Log("Cancel pressed");
    }
    
    private void NavigateUp()
    {
        // Implement focus navigation logic
        var focusController = _root.focusController;
        focusController.GetFocusableParentForPointerEvent(_currentFocusedElement as VisualElement, out Focusable next);
        if (next != null)
        {
            next.Focus();
            _currentFocusedElement = next;
        }
    }
    
    private void NavigateDown() { /* Similar to NavigateUp */ }
    private void NavigateLeft() { /* Similar to NavigateUp */ }
    private void NavigateRight() { /* Similar to NavigateUp */ }
}
```

### 7.2 Gamepad/Controller Navigation

**USS for Focus States**
```css
/* Focus indicator */
.menu-button:focus {
    border-color: var(--secondary-color);
    border-width: 3px;
    scale: 1.05;
}

/* Selected state */
.menu-button.selected {
    background-color: var(--primary-color);
}

/* Transition for smooth focus changes */
.menu-button {
    transition-property: border-color, border-width, scale;
    transition-duration: 0.2s;
}
```

---

## 8. Performance Patterns

### 8.1 Object Pooling for UI Elements

**Example: UI Element Pool**
```csharp
using UnityEngine;
using UnityEngine.UIElements;
using System.Collections.Generic;

public class UIElementPool<T> where T : VisualElement, new()
{
    private Stack<T> _pool = new Stack<T>();
    private VisualElement _container;
    
    public UIElementPool(VisualElement container, int initialSize = 10)
    {
        _container = container;
        
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
        return element;
    }
    
    public void Return(T element)
    {
        element.style.display = DisplayStyle.None;
        _pool.Push(element);
    }
}

// Usage example
public class InventoryController : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;
    
    private VisualElement _itemContainer;
    private UIElementPool<VisualElement> _itemPool;
    
    private void OnEnable()
    {
        var root = _uiDocument.rootVisualElement;
        _itemContainer = root.Q<VisualElement>("ItemContainer");
        
        // Create pool
        _itemPool = new UIElementPool<VisualElement>(_itemContainer, 20);
    }
    
    public void DisplayItems(List<string> items)
    {
        // Return all items to pool first
        for (int i = _itemContainer.childCount - 1; i >= 0; i--)
        {
            var child = _itemContainer[i] as VisualElement;
            if (child != null)
            {
                _itemPool.Return(child);
            }
        }
        
        // Get items from pool and populate
        foreach (var item in items)
        {
            var element = _itemPool.Get();
            var label = element.Q<Label>();
            if (label != null)
            {
                label.text = item;
            }
        }
    }
}
```

### 8.2 USS Optimization for Quest 2

**Performance-Optimized USS**
```css
/* QUEST 2 OPTIMIZATION PATTERNS */

/* Use transform instead of position for animations (GPU accelerated) */
.animated-element {
    transition-property: translate, scale, rotate;
    transition-duration: 0.3s;
    translate: 0 0; /* Use translate instead of position */
}

/* Avoid expensive box-shadows on low-end hardware */
.no-shadow-mobile {
    /* box-shadow: 0 4px 8px rgba(0,0,0,0.2); */ /* Disabled */
}

/* Use simple gradients instead of complex multi-layer ones */
.optimized-gradient {
    background-image: linear-gradient(
        135deg,
        var(--glow-primary),
        var(--glow-secondary)
    );
    /* Instead of 3-4 layered radial gradients */
}

/* Reduce border-radius complexity */
.simple-corners {
    border-radius: 8px; /* Instead of 16px or per-corner values */
}

/* Disable transitions on low-end mode */
.performance-mode .menu-button {
    transition-duration: 0s;
}

/* Use opacity for show/hide instead of display */
.hidden-optimized {
    opacity: 0;
    /* visibility: hidden; */
    /* Instead of display: none which triggers layout */
}
```

### 8.3 Efficient Data Binding Updates

**Example: Batched Updates**
```csharp
using UnityEngine;
using UnityEngine.UIElements;
using System.Collections;

public class OptimizedDataUpdater : MonoBehaviour
{
    [SerializeField] private GameData _gameData;
    [SerializeField] private UIDocument _uiDocument;
    
    private SerializedObject _serializedObject;
    private Coroutine _updateCoroutine;
    
    private void OnEnable()
    {
        _serializedObject = new SerializedObject(_gameData);
        var root = _uiDocument.rootVisualElement;
        root.Bind(_serializedObject);
        
        // Start batched update coroutine
        _updateCoroutine = StartCoroutine(BatchedUpdateRoutine());
    }
    
    private void OnDisable()
    {
        if (_updateCoroutine != null)
        {
            StopCoroutine(_updateCoroutine);
        }
    }
    
    private IEnumerator BatchedUpdateRoutine()
    {
        // Update UI at lower frequency than game loop
        var wait = new WaitForSeconds(0.1f); // 10 updates per second
        
        while (true)
        {
            // Apply all pending changes at once
            _serializedObject.Update();
            
            yield return wait;
        }
    }
}
```

---

## 9. Code Examples

### 9.1 Complete Main Menu Implementation

**MainMenu.uxml**
```xml
<ui:UXML xmlns:ui="UnityEngine.UIElements">
    <Style src="MainMenu.uss" />
    
    <ui:VisualElement name="MainMenuRoot" class="menu-container">
        
        <!-- Background with gradient -->
        <ui:VisualElement name="Background" class="menu-background" />
        
        <!-- Content container -->
        <ui:VisualElement name="Content" class="content-container">
            
            <!-- Title -->
            <ui:Label text="BAGEL GAME" class="game-title" />
            
            <!-- Button group -->
            <ui:VisualElement name="ButtonGroup" class="button-container">
                <ui:Button name="PlayButton" text="PLAY" class="menu-button primary-button" />
                <ui:Button name="OptionsButton" text="OPTIONS" class="menu-button" />
                <ui:Button name="CreditsButton" text="CREDITS" class="menu-button" />
                <ui:Button name="QuitButton" text="QUIT" class="menu-button" />
            </ui:VisualElement>
            
            <!-- Version info -->
            <ui:Label name="VersionLabel" text="v1.0.0" class="version-text" />
            
        </ui:VisualElement>
        
    </ui:VisualElement>
</ui:UXML>
```

**MainMenu.uss**
```css
/* ===== ROOT THEME ===== */
:root {
    --primary-purple: rgb(138, 43, 226);
    --secondary-cyan: rgb(0, 217, 255);
    --bg-dark: rgb(5, 5, 10);
    --bg-mid: rgb(15, 15, 25);
    
    --glow-purple: rgba(138, 43, 226, 0.5);
    --glow-cyan: rgba(0, 217, 255, 0.4);
    
    --transition-fast: 0.2s;
    --transition-normal: 0.3s;
}

/* ===== CONTAINERS ===== */
.menu-container {
    flex-grow: 1;
    width: 100%;
    height: 100%;
}

.menu-background {
    position: absolute;
    width: 100%;
    height: 100%;
    
    background-image: 
        radial-gradient(circle at 25% 35%, var(--glow-purple) 0%, transparent 55%),
        radial-gradient(circle at 75% 65%, var(--glow-cyan) 0%, transparent 55%),
        linear-gradient(180deg, var(--bg-mid) 0%, var(--bg-dark) 100%);
}

.content-container {
    flex-grow: 1;
    justify-content: center;
    align-items: center;
    padding: 40px;
}

/* ===== TYPOGRAPHY ===== */
.game-title {
    font-size: 96px;
    color: white;
    -unity-font-style: bold;
    -unity-text-align: middle-center;
    text-shadow: 0px 4px 30px var(--glow-cyan);
    margin-bottom: 60px;
    
    /* Letter spacing for modern look */
    letter-spacing: 8px;
}

.version-text {
    font-size: 14px;
    color: rgba(255, 255, 255, 0.4);
    -unity-text-align: middle-center;
    margin-top: 40px;
}

/* ===== BUTTONS ===== */
.button-container {
    align-items: center;
    gap: 16px;
}

.menu-button {
    min-width: 300px;
    min-height: 70px;
    
    /* Glass morphism */
    background-color: rgba(255, 255, 255, 0.03);
    border-width: 2px;
    border-color: rgba(255, 255, 255, 0.08);
    border-radius: 12px;
    
    /* Typography */
    font-size: 24px;
    color: white;
    -unity-font-style: bold;
    letter-spacing: 2px;
    
    /* Transitions */
    transition-property: background-color, border-color, scale, translate;
    transition-duration: var(--transition-normal);
}

.menu-button:hover {
    background-color: rgba(255, 255, 255, 0.1);
    border-color: var(--secondary-cyan);
    scale: 1.05 1.05;
    
    /* Subtle glow */
    box-shadow: 0px 0px 20px rgba(0, 217, 255, 0.3);
}

.menu-button:active {
    scale: 0.95 0.95;
    background-color: rgba(0, 217, 255, 0.15);
}

.menu-button:focus {
    border-color: var(--secondary-cyan);
    border-width: 3px;
}

/* Primary button variant */
.primary-button {
    background-color: var(--primary-purple);
    border-color: var(--primary-purple);
}

.primary-button:hover {
    background-color: rgba(138, 43, 226, 0.8);
    border-color: var(--secondary-cyan);
    box-shadow: 0px 0px 30px var(--glow-purple);
}
```

**MainMenuController.cs**
```csharp
using UnityEngine;
using UnityEngine.UIElements;
using UnityEngine.SceneManagement;

public class MainMenuController : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;
    [SerializeField] private AudioClip _hoverSound;
    [SerializeField] private AudioClip _clickSound;
    
    private VisualElement _root;
    private Button _playButton;
    private Button _optionsButton;
    private Button _creditsButton;
    private Button _quitButton;
    
    private void OnEnable()
    {
        _root = _uiDocument.rootVisualElement;
        
        // Query UI elements
        _playButton = _root.Q<Button>("PlayButton");
        _optionsButton = _root.Q<Button>("OptionsButton");
        _creditsButton = _root.Q<Button>("CreditsButton");
        _quitButton = _root.Q<Button>("QuitButton");
        
        // Register callbacks
        RegisterButtonEvents(_playButton, OnPlayClicked);
        RegisterButtonEvents(_optionsButton, OnOptionsClicked);
        RegisterButtonEvents(_creditsButton, OnCreditsClicked);
        RegisterButtonEvents(_quitButton, OnQuitClicked);
    }
    
    private void OnDisable()
    {
        // Unregister callbacks
        UnregisterButtonEvents(_playButton, OnPlayClicked);
        UnregisterButtonEvents(_optionsButton, OnOptionsClicked);
        UnregisterButtonEvents(_creditsButton, OnCreditsClicked);
        UnregisterButtonEvents(_quitButton, OnQuitClicked);
    }
    
    private void RegisterButtonEvents(Button button, System.Action callback)
    {
        if (button == null) return;
        
        button.clicked += callback;
        
        // Add hover sound
        button.RegisterCallback<PointerEnterEvent>(evt => PlaySound(_hoverSound));
    }
    
    private void UnregisterButtonEvents(Button button, System.Action callback)
    {
        if (button == null) return;
        
        button.clicked -= callback;
    }
    
    private void OnPlayClicked()
    {
        PlaySound(_clickSound);
        SceneManager.LoadScene("GameScene");
    }
    
    private void OnOptionsClicked()
    {
        PlaySound(_clickSound);
        Debug.Log("Options clicked - open settings panel");
    }
    
    private void OnCreditsClicked()
    {
        PlaySound(_clickSound);
        Debug.Log("Credits clicked - show credits");
    }
    
    private void OnQuitClicked()
    {
        PlaySound(_clickSound);
        
        #if UNITY_EDITOR
        UnityEditor.EditorApplication.isPlaying = false;
        #else
        Application.Quit();
        #endif
    }
    
    private void PlaySound(AudioClip clip)
    {
        if (clip != null)
        {
            AudioSource.PlayClipAtPoint(clip, Camera.main.transform.position, 0.5f);
        }
    }
}
```

### 9.2 HUD with Live Data Binding

**HUD.uxml**
```xml
<ui:UXML xmlns:ui="UnityEngine.UIElements">
    <Style src="HUD.uss" />
    
    <ui:VisualElement name="HUDRoot" class="hud-container">
        
        <!-- Top bar -->
        <ui:VisualElement name="TopBar" class="top-bar">
            
            <!-- Score -->
            <ui:VisualElement class="stat-group">
                <ui:Label text="SCORE" class="stat-label" />
                <ui:Label name="ScoreValue" 
                         binding-path="CurrentScore" 
                         class="stat-value" />
            </ui:VisualElement>
            
            <!-- Toppings -->
            <ui:VisualElement class="stat-group">
                <ui:Label text="TOPPINGS" class="stat-label" />
                <ui:Label name="ToppingsValue" 
                         binding-path="ToppingsRemaining" 
                         class="stat-value" />
            </ui:VisualElement>
            
            <!-- Timer -->
            <ui:VisualElement class="stat-group">
                <ui:Label text="TIME" class="stat-label" />
                <ui:Label name="TimeValue" 
                         binding-path="TimeRemaining" 
                         class="stat-value" />
            </ui:VisualElement>
            
        </ui:VisualElement>
        
        <!-- Health bar -->
        <ui:VisualElement name="HealthContainer" class="health-container">
            <ui:ProgressBar name="HealthBar" 
                           binding-path="Health"
                           low-value="0" 
                           high-value="100"
                           class="health-bar" />
        </ui:VisualElement>
        
    </ui:VisualElement>
</ui:UXML>
```

**HUD.uss**
```css
:root {
    --health-high: rgb(76, 217, 100);
    --health-medium: rgb(255, 204, 0);
    --health-low: rgb(255, 59, 48);
    --stat-text: rgb(0, 217, 255);
}

.hud-container {
    position: absolute;
    width: 100%;
    height: 100%;
    padding: 20px;
    pointer-events: none; /* Allow clicks to pass through */
}

.top-bar {
    flex-direction: row;
    justify-content: space-between;
    padding: 20px;
    
    background-color: rgba(0, 0, 0, 0.4);
    border-radius: 12px;
    border-width: 1px;
    border-color: rgba(255, 255, 255, 0.1);
}

.stat-group {
    align-items: center;
}

.stat-label {
    font-size: 14px;
    color: rgba(255, 255, 255, 0.6);
    margin-bottom: 4px;
    letter-spacing: 1px;
}

.stat-value {
    font-size: 32px;
    color: var(--stat-text);
    -unity-font-style: bold;
}

/* Health bar */
.health-container {
    position: absolute;
    bottom: 40px;
    left: 50%;
    translate: -50% 0;
    width: 400px;
}

.health-bar {
    height: 30px;
    border-radius: 15px;
    background-color: rgba(0, 0, 0, 0.5);
    border-width: 2px;
    border-color: rgba(255, 255, 255, 0.2);
}

.health-bar > #unity-progress-bar {
    background-color: var(--health-high);
    border-radius: 13px;
    margin: 2px;
}

/* Dynamic health color (set via C#) */
.health-bar.low > #unity-progress-bar {
    background-color: var(--health-low);
}

.health-bar.medium > #unity-progress-bar {
    background-color: var(--health-medium);
}
```

**HUDController.cs**
```csharp
using UnityEngine;
using UnityEngine.UIElements;

public class HUDController : MonoBehaviour
{
    [SerializeField] private UIDocument _uiDocument;
    [SerializeField] private GameData _gameData;
    
    private VisualElement _root;
    private Label _scoreValue;
    private Label _toppingsValue;
    private Label _timeValue;
    private ProgressBar _healthBar;
    
    private SerializedObject _serializedGameData;
    
    private void OnEnable()
    {
        _root = _uiDocument.rootVisualElement;
        
        // Query elements
        _scoreValue = _root.Q<Label>("ScoreValue");
        _toppingsValue = _root.Q<Label>("ToppingsValue");
        _timeValue = _root.Q<Label>("TimeValue");
        _healthBar = _root.Q<ProgressBar>("HealthBar");
        
        // Setup data binding
        SetupBinding();
        
        // Setup custom formatters
        SetupCustomFormatters();
    }
    
    private void SetupBinding()
    {
        _serializedGameData = new SerializedObject(_gameData);
        _root.Bind(_serializedGameData);
    }
    
    private void SetupCustomFormatters()
    {
        // Custom score formatter
        _root.TrackPropertyValue(
            _serializedGameData.FindProperty("_currentScore"),
            property => {
                _scoreValue.text = $"{_gameData.CurrentScore:N0}";
            }
        );
        
        // Custom time formatter (MM:SS)
        _root.TrackPropertyValue(
            _serializedGameData.FindProperty("_timeRemaining"),
            property => {
                int minutes = Mathf.FloorToInt(_gameData.TimeRemaining / 60f);
                int seconds = Mathf.FloorToInt(_gameData.TimeRemaining % 60f);
                _timeValue.text = $"{minutes:00}:{seconds:00}";
            }
        );
        
        // Health color update
        _root.TrackPropertyValue(
            _serializedGameData.FindProperty("_health"),
            property => {
                UpdateHealthColor(_gameData.Health);
            }
        );
    }
    
    private void UpdateHealthColor(float health)
    {
        // Remove all health classes
        _healthBar.RemoveFromClassList("low");
        _healthBar.RemoveFromClassList("medium");
        
        // Add appropriate class
        if (health < 30f)
            _healthBar.AddToClassList("low");
        else if (health < 60f)
            _healthBar.AddToClassList("medium");
    }
}
```

---

## 10. Quest 2 VR Adaptation

### Key Considerations for Quest 2

1. **Performance Budget**: Target 72-90 Hz
2. **Resolution**: 1832x1920 per eye
3. **GPU**: Mobile GPU - avoid heavy effects
4. **Input**: Controller rays + hand tracking

### Optimized VR UI Setup

**VRUIManager.cs**
```csharp
using UnityEngine;
using UnityEngine.UIElements;

public class VRUIManager : MonoBehaviour
{
    [Header("References")]
    [SerializeField] private UIDocument _uiDocument;
    [SerializeField] private Camera _centerEyeCamera;
    
    [Header("VR Settings")]
    [SerializeField] private float _uiDistance = 2.5f;
    [SerializeField] private float _pixelsPerMeter = 1000f;
    [SerializeField] private Vector2 _panelSize = new Vector2(1.5f, 1.0f); // meters
    
    [Header("Performance")]
    [SerializeField] private int _targetFrameRate = 90;
    [SerializeField] private bool _usePerformanceMode = true;
    
    private PanelSettings _panelSettings;
    private RenderTexture _uiRenderTexture;
    
    private void Awake()
    {
        // Set target frame rate for Quest 2
        Application.targetFrameRate = _targetFrameRate;
        
        // Setup VR UI
        SetupVRPanel();
        
        // Apply performance optimizations
        if (_usePerformanceMode)
        {
            ApplyPerformanceOptimizations();
        }
    }
    
    private void SetupVRPanel()
    {
        // Calculate resolution based on panel size and pixels per meter
        int width = Mathf.RoundToInt(_panelSize.x * _pixelsPerMeter);
        int height = Mathf.RoundToInt(_panelSize.y * _pixelsPerMeter);
        
        // Create render texture
        _uiRenderTexture = new RenderTexture(width, height, 0, RenderTextureFormat.ARGB32);
        _uiRenderTexture.Create();
        
        // Create panel settings
        _panelSettings = ScriptableObject.CreateInstance<PanelSettings>();
        _panelSettings.targetTexture = _uiRenderTexture;
        _panelSettings.clearColor = Color.clear;
        _panelSettings.scaleMode = PanelScaleMode.ScaleWithScreenSize;
        
        _uiDocument.panelSettings = _panelSettings;
    }
    
    private void ApplyPerformanceOptimizations()
    {
        var root = _uiDocument.rootVisualElement;
        
        // Add performance mode class to disable expensive effects
        root.AddToClassList("performance-mode");
        
        // Disable antialiasing for better performance
        _uiRenderTexture.antiAliasing = 1;
    }
}
```

### Performance-Optimized USS for Quest 2

```css
/* Quest 2 Performance Mode */
.performance-mode {
    /* Disable all transitions */
    * {
        transition-duration: 0s !important;
    }
}

/* Simplified gradients for mobile GPU */
.vr-background {
    background-image: linear-gradient(
        135deg,
        rgba(138, 43, 226, 0.3),
        rgba(0, 217, 255, 0.2)
    );
    /* Single gradient instead of multiple layered radials */
}

/* Reduce border complexity */
.vr-button {
    border-radius: 8px; /* Simpler than 12-16px */
    border-width: 2px;
}

/* No shadows in VR mode */
.vr-panel {
    /* box-shadow: none; */
}
```

---

## Summary & Best Practices

### Architecture Takeaways

1. **Separation of Concerns**:
   - UXML = Structure
   - USS = Styling
   - C# = Logic & Data

2. **Data Binding**: Use Unity 6's SerializedObject binding for automatic UI updates

3. **World Space UI**: Render to texture, display on quad in 3D space

4. **Themes**: CSS variables + runtime updates for dynamic theming

5. **Shader Graph**: Apply sparingly for premium effects (glass, glow)

### Quest 2 Optimization Checklist

- ✅ Target 90 Hz, accept 72 Hz minimum
- ✅ Use simple gradients (linear > radial)
- ✅ Disable transitions in performance mode
- ✅ Pool UI elements for dynamic content
- ✅ Batch data updates (10-30 Hz)
- ✅ Simple border-radius values
- ✅ No box-shadows
- ✅ Anti-aliasing: 1x or off

---

## Resources

- **Bagel Game Repo**: https://github.com/Unity-Technologies/BagelGame
- **UI Toolkit Manual**: https://docs.unity3d.com/Manual/UIElements.html
- **Shader Graph Samples**: Package Manager → Shader Graph → Samples
- **Unity Input System**: https://docs.unity3d.com/Packages/com.unity.inputsystem@latest

---

*This analysis provides a comprehensive foundation for building UI Toolkit systems inspired by Bagel Game patterns, optimized for Quest 2 VR deployment.*
