# Factory → Builder → View → Host → Controller

> **Pattern**: Code-generated UITK panels for the Training App  
> **Examples**: Login, Settings (with tabs), Module instructions  
> **Note**: Code samples are demonstrative — not production-complete

---

## Layer Overview

```
UIFactory          creates individual elements with correct USS classes
PanelBuilder       assembles elements into layouts, returns cached refs
*PanelView         pure C#, calls builder, wires internal events, exposes them upward  
*PanelHost         MonoBehaviour, panel lifecycle, Unity API calls
AppController      MonoBehaviour, scene orchestration, navigation
```

Each layer has one job. The View never touches Unity APIs. The Host never touches raw `VisualElement` creation. The Controller never queries the UI directly.

---

## UIStyles — String Constants

Every USS class name lives here. Never use raw strings anywhere else in the codebase. If a class name changes in USS, there is one place to update it.

```csharp
public static class UIStyles
{
    // Layout
    public const string PanelRoot        = "panel-root";
    public const string PanelHeader      = "panel-header";
    public const string PanelBody        = "panel-body";
    public const string PanelFooter      = "panel-footer";
    public const string Row              = "row";
    public const string Column           = "column";
    public const string Spacer           = "spacer";

    // Typography
    public const string TitleLarge       = "title-large";
    public const string TitleMedium      = "title-medium";
    public const string BodyText         = "body-text";
    public const string LabelText        = "label-text";
    public const string CaptionText      = "caption-text";

    // Buttons
    public const string BtnPrimary       = "btn-primary";
    public const string BtnSecondary     = "btn-secondary";
    public const string BtnGhost         = "btn-ghost";
    public const string BtnTab           = "btn-tab";
    public const string BtnTabActive     = "btn-tab--active";
    public const string BtnIcon          = "btn-icon";
    public const string BtnDanger        = "btn-danger";

    // Inputs
    public const string InputGroup       = "input-group";
    public const string InputLabel       = "input-label";
    public const string InputField       = "input-field";
    public const string InputError       = "input-field--error";

    // Components
    public const string Card             = "card";
    public const string CardActive       = "card--active";
    public const string CardLocked       = "card--locked";
    public const string GlassPanel       = "glass-panel";
    public const string Divider          = "divider";
    public const string TabBar           = "tab-bar";
    public const string TabContent       = "tab-content";
    public const string ProgressTrack    = "progress-track";
    public const string ProgressFill     = "progress-fill";
    public const string ProgressLabel    = "progress-label";
    public const string StepItem         = "step-item";
    public const string StepItemComplete = "step-item--complete";
    public const string StepItemActive   = "step-item--active";

    // State
    public const string Hidden           = "hidden";
    public const string Disabled         = "disabled";
    public const string FadeIn           = "fade-in";
    public const string Loading          = "loading";
}
```

---

## UIFactory

Creates individual elements. No layout assembly here — just correctly classed elements ready to be placed by a builder.

Tuple returns are used throughout so callers get both the wrapper group and the interactive element without needing to query for either.

```csharp
public static class UIFactory
{
    // ── Containers ──────────────────────────────────────────────────────────

    public static VisualElement CreateContainer(params string[] classes)
    {
        var el = new VisualElement();
        foreach (var c in classes) el.AddToClassList(c);
        return el;
    }

    public static VisualElement CreateRow(params string[] extraClasses)
    {
        var el = new VisualElement();
        el.AddToClassList(UIStyles.Row);
        foreach (var c in extraClasses) el.AddToClassList(c);
        return el;
    }

    public static VisualElement CreateColumn(params string[] extraClasses)
    {
        var el = new VisualElement();
        el.AddToClassList(UIStyles.Column);
        foreach (var c in extraClasses) el.AddToClassList(c);
        return el;
    }

    public static VisualElement CreateDivider()
    {
        var el = new VisualElement();
        el.AddToClassList(UIStyles.Divider);
        return el;
    }

    public static VisualElement CreateGlassPanel()
    {
        var el = new VisualElement();
        el.AddToClassList(UIStyles.GlassPanel);
        return el;
    }

    // ── Text ────────────────────────────────────────────────────────────────

    public static Label CreateLabel(string text, string styleClass = UIStyles.BodyText)
    {
        var label = new Label(text);
        label.AddToClassList(styleClass);
        return label;
    }

    public static Label CreateTitle(string text, bool large = true)
        => CreateLabel(text, large ? UIStyles.TitleLarge : UIStyles.TitleMedium);

    // ── Buttons ─────────────────────────────────────────────────────────────

    public static Button CreateButton(string text, string styleClass = UIStyles.BtnPrimary)
    {
        var btn = new Button { text = text };
        btn.AddToClassList(styleClass);
        return btn;
    }

    public static Button CreateTabButton(string text)
    {
        var btn = new Button { text = text };
        btn.AddToClassList(UIStyles.BtnTab);
        return btn;
    }

    public static Button CreateIconButton(Sprite icon, string tooltip = "")
    {
        var btn = new Button();
        btn.AddToClassList(UIStyles.BtnIcon);
        btn.tooltip = tooltip;
        if (icon != null)
            btn.style.backgroundImage = new StyleBackground(icon);
        return btn;
    }

    // ── Inputs ──────────────────────────────────────────────────────────────

    public static (VisualElement group, TextField field) CreateTextField(
        string labelText, string placeholder = "")
    {
        var group = new VisualElement();
        group.AddToClassList(UIStyles.InputGroup);

        var label = CreateLabel(labelText, UIStyles.InputLabel);
        var field = new TextField { value = placeholder };
        field.AddToClassList(UIStyles.InputField);

        group.Add(label);
        group.Add(field);

        return (group, field);
    }

    public static (VisualElement group, TextField field) CreatePasswordField(string labelText)
    {
        var (group, field) = CreateTextField(labelText);
        field.isPasswordField = true;
        return (group, field);
    }

    public static (VisualElement group, Slider slider, Label valueLabel) CreateSlider(
        string labelText, float min, float max, float initial)
    {
        var group = new VisualElement();
        group.AddToClassList(UIStyles.InputGroup);

        var headerRow = CreateRow();
        headerRow.Add(CreateLabel(labelText, UIStyles.InputLabel));

        var valueLabel = CreateLabel(initial.ToString("0%"), UIStyles.CaptionText);
        headerRow.Add(valueLabel);

        var slider = new Slider(min, max) { value = initial };
        slider.AddToClassList(UIStyles.InputField);

        group.Add(headerRow);
        group.Add(slider);

        return (group, slider, valueLabel);
    }

    public static (VisualElement group, Toggle toggle) CreateToggle(
        string labelText, bool initial = false)
    {
        var group = new VisualElement();
        group.AddToClassList(UIStyles.InputGroup);

        var toggle = new Toggle(labelText) { value = initial };
        group.Add(toggle);

        return (group, toggle);
    }

    public static (VisualElement group, DropdownField dropdown) CreateDropdown(
        string labelText, List<string> choices, int defaultIndex = 0)
    {
        var group = new VisualElement();
        group.AddToClassList(UIStyles.InputGroup);

        group.Add(CreateLabel(labelText, UIStyles.InputLabel));

        var dropdown = new DropdownField(choices, defaultIndex);
        dropdown.AddToClassList(UIStyles.InputField);
        group.Add(dropdown);

        return (group, dropdown);
    }

    // ── Composite ───────────────────────────────────────────────────────────

    public static (VisualElement track, VisualElement fill, Label label) CreateProgressBar(
        string labelText = "")
    {
        var track = new VisualElement();
        track.AddToClassList(UIStyles.ProgressTrack);

        var fill = new VisualElement();
        fill.AddToClassList(UIStyles.ProgressFill);
        track.Add(fill);

        var label = CreateLabel(labelText, UIStyles.ProgressLabel);

        return (track, fill, label);
    }

    public static (VisualElement card, Label title, Label body) CreateCard(
        string title, string body)
    {
        var card = new VisualElement();
        card.AddToClassList(UIStyles.Card);

        var titleLabel = CreateLabel(title, UIStyles.TitleMedium);
        var bodyLabel  = CreateLabel(body, UIStyles.BodyText);

        card.Add(titleLabel);
        card.Add(bodyLabel);

        return (card, titleLabel, bodyLabel);
    }
}
```

---

## PanelBuilder

The base builder handles the panel skeleton. Typed subclasses add panel-specific elements and return typed element results — a simple record of every element the View will need at runtime.

The View holds this result and never runs `Q<>()` after build time. If a builder's internals change, the compiler tells you what broke in the View.

### Base Builder

```csharp
public class PanelBuilder
{
    protected readonly VisualElement _root;
    protected VisualElement _body;
    protected VisualElement _footer;

    public PanelBuilder(VisualElement root)
    {
        _root = root;
        _root.AddToClassList(UIStyles.PanelRoot);
    }

    public PanelBuilder AddHeader(string title)
    {
        var header = new VisualElement();
        header.AddToClassList(UIStyles.PanelHeader);
        header.Add(UIFactory.CreateTitle(title));
        _root.Add(header);
        return this;
    }

    // Variant for panels that need a back button in the header
    public PanelBuilder AddHeaderWithBack(string title, out Button backButton)
    {
        var header = new VisualElement();
        header.AddToClassList(UIStyles.PanelHeader);

        var row = UIFactory.CreateRow();
        backButton = UIFactory.CreateButton("←", UIStyles.BtnGhost);
        row.Add(backButton);
        row.Add(UIFactory.CreateTitle(title, large: false));
        header.Add(row);
        _root.Add(header);
        return this;
    }

    // Variant for headers that need a right-side action button
    public PanelBuilder AddHeaderWithAction(string title, out Label titleLabel,
        out Button actionButton, string actionText = "⚙")
    {
        var header = new VisualElement();
        header.AddToClassList(UIStyles.PanelHeader);

        var row = UIFactory.CreateRow();
        titleLabel   = UIFactory.CreateTitle(title, large: false);
        actionButton = UIFactory.CreateButton(actionText, UIStyles.BtnGhost);

        row.Add(titleLabel);
        row.Add(actionButton);
        header.Add(row);
        _root.Add(header);
        return this;
    }

    protected VisualElement EnsureBody()
    {
        if (_body != null) return _body;
        _body = new VisualElement();
        _body.AddToClassList(UIStyles.PanelBody);
        _root.Add(_body);
        return _body;
    }

    protected VisualElement EnsureFooter()
    {
        if (_footer != null) return _footer;
        _footer = new VisualElement();
        _footer.AddToClassList(UIStyles.PanelFooter);
        _root.Add(_footer);
        return _footer;
    }
}
```

### Typed Element Results

Each builder returns one of these. The View holds the result and accesses elements by name — no string queries after build time.

```csharp
public class LoginElements
{
    public TextField     UsernameField { get; init; }
    public TextField     PasswordField { get; init; }
    public Button        LoginButton   { get; init; }
    public Label         ErrorLabel    { get; init; }
}

public class DashboardElements
{
    public Label         WelcomeLabel   { get; init; }
    public VisualElement ProgressFill   { get; init; }
    public Label         ProgressLabel  { get; init; }
    public VisualElement ModuleList     { get; init; }
    public Button        SettingsButton { get; init; }
}

public class SettingsElements
{
    public Button        VisualTab          { get; init; }
    public Button        AudioTab           { get; init; }
    public VisualElement VisualContent      { get; init; }
    public VisualElement AudioContent       { get; init; }
    public DropdownField ThemeDropdown      { get; init; }
    public Toggle        HighContrastToggle { get; init; }
    public Slider        MasterSlider       { get; init; }
    public Label         MasterValueLabel   { get; init; }
    public Slider        MusicSlider        { get; init; }
    public Label         MusicValueLabel    { get; init; }
    public Slider        SFXSlider          { get; init; }
    public Label         SFXValueLabel      { get; init; }
    public Button        BackButton         { get; init; }
}

public class ModuleElements
{
    public Label         ModuleTitleLabel  { get; init; }
    public Label         InstructionsLabel { get; init; }
    public VisualElement MediaContainer    { get; init; }
    public VisualElement StepList         { get; init; }
    public Button        PreviousButton   { get; init; }
    public Button        NextButton       { get; init; }
    public Label         StepCountLabel   { get; init; }
}
```

### Login Builder

```csharp
public class LoginBuilder : PanelBuilder
{
    public LoginBuilder(VisualElement root) : base(root) { }

    public LoginElements Build()
    {
        AddHeader("Welcome Back");
        var body = EnsureBody();

        var (userGroup, usernameField) = UIFactory.CreateTextField("Username");
        var (passGroup, passwordField) = UIFactory.CreatePasswordField("Password");

        // Error label starts hidden — shown by the View on validation or auth failure
        var errorLabel = UIFactory.CreateLabel("", UIStyles.CaptionText);
        errorLabel.AddToClassList(UIStyles.Hidden);

        body.Add(userGroup);
        body.Add(passGroup);
        body.Add(errorLabel);

        var footer      = EnsureFooter();
        var loginButton = UIFactory.CreateButton("Sign In");
        footer.Add(loginButton);

        return new LoginElements
        {
            UsernameField = usernameField,
            PasswordField = passwordField,
            LoginButton   = loginButton,
            ErrorLabel    = errorLabel
        };
    }
}
```

### Dashboard Builder

```csharp
public class DashboardBuilder : PanelBuilder
{
    public DashboardBuilder(VisualElement root) : base(root) { }

    public DashboardElements Build()
    {
        // Header with welcome text and settings button in the corner
        AddHeaderWithAction("Hello", out var welcomeLabel, out var settingsButton);

        var body = EnsureBody();

        // Overall progress section
        var (progressTrack, progressFill, progressLabel) =
            UIFactory.CreateProgressBar("Overall Progress");

        body.Add(progressLabel);
        body.Add(progressTrack);
        body.Add(UIFactory.CreateDivider());

        // Module list — View calls PopulateModules() to fill this at runtime
        body.Add(UIFactory.CreateLabel("Modules", UIStyles.TitleMedium));
        var moduleList = UIFactory.CreateColumn();
        body.Add(moduleList);

        return new DashboardElements
        {
            WelcomeLabel   = welcomeLabel,
            ProgressFill   = progressFill,
            ProgressLabel  = progressLabel,
            ModuleList     = moduleList,
            SettingsButton = settingsButton
        };
    }
}
```

### Settings Builder — Tab Pattern

```csharp
public class SettingsBuilder : PanelBuilder
{
    private static readonly List<string> ThemeChoices =
        new() { "Default", "Dark", "Light", "High Contrast" };

    public SettingsBuilder(VisualElement root) : base(root) { }

    public SettingsElements Build()
    {
        AddHeaderWithBack("Settings", out var backButton);
        var body = EnsureBody();

        // ── Tab bar ─────────────────────────────────────────────────────────
        var tabBar   = UIFactory.CreateRow(UIStyles.TabBar);
        var visualTab = UIFactory.CreateTabButton("Visual");
        var audioTab  = UIFactory.CreateTabButton("Audio");
        visualTab.AddToClassList(UIStyles.BtnTabActive); // default active tab
        tabBar.Add(visualTab);
        tabBar.Add(audioTab);
        body.Add(tabBar);

        // ── Visual tab content ───────────────────────────────────────────────
        var visualContent = UIFactory.CreateColumn(UIStyles.TabContent);

        var (themeGroup, themeDropdown)     = UIFactory.CreateDropdown("Theme", ThemeChoices);
        var (contrastGroup, contrastToggle) = UIFactory.CreateToggle("High Contrast Mode");

        visualContent.Add(themeGroup);
        visualContent.Add(contrastGroup);
        body.Add(visualContent);

        // ── Audio tab content (hidden by default) ────────────────────────────
        var audioContent = UIFactory.CreateColumn(UIStyles.TabContent);
        audioContent.AddToClassList(UIStyles.Hidden);

        var (masterGroup, masterSlider, masterLabel) =
            UIFactory.CreateSlider("Master Volume", 0f, 1f, 0.8f);
        var (musicGroup, musicSlider, musicLabel) =
            UIFactory.CreateSlider("Music Volume", 0f, 1f, 0.5f);
        var (sfxGroup, sfxSlider, sfxLabel) =
            UIFactory.CreateSlider("SFX Volume", 0f, 1f, 0.7f);

        audioContent.Add(masterGroup);
        audioContent.Add(UIFactory.CreateDivider());
        audioContent.Add(musicGroup);
        audioContent.Add(sfxGroup);
        body.Add(audioContent);

        return new SettingsElements
        {
            VisualTab          = visualTab,
            AudioTab           = audioTab,
            VisualContent      = visualContent,
            AudioContent       = audioContent,
            ThemeDropdown      = themeDropdown,
            HighContrastToggle = contrastToggle,
            MasterSlider       = masterSlider,
            MasterValueLabel   = masterLabel,
            MusicSlider        = musicSlider,
            MusicValueLabel    = musicLabel,
            SFXSlider          = sfxSlider,
            SFXValueLabel      = sfxLabel,
            BackButton         = backButton
        };
    }
}
```

### Module Builder

```csharp
public class ModuleBuilder : PanelBuilder
{
    public ModuleBuilder(VisualElement root) : base(root) { }

    public ModuleElements Build()
    {
        var body = EnsureBody();

        // Title and instructions at the top
        var moduleTitleLabel  = UIFactory.CreateTitle("", large: false);
        var instructionsLabel = UIFactory.CreateLabel("", UIStyles.BodyText);

        body.Add(moduleTitleLabel);
        body.Add(instructionsLabel);
        body.Add(UIFactory.CreateDivider());

        // Media placeholder — hidden until the Host sets content
        var mediaContainer = UIFactory.CreateContainer(UIStyles.Card);
        mediaContainer.AddToClassList(UIStyles.Hidden);
        body.Add(mediaContainer);

        // Step checklist — rebuilt by View on each step change
        var stepList = UIFactory.CreateColumn();
        body.Add(stepList);

        // Footer: previous / step counter / next
        var footer    = EnsureFooter();
        var footerRow = UIFactory.CreateRow();

        var prevButton     = UIFactory.CreateButton("← Previous", UIStyles.BtnSecondary);
        var stepCountLabel = UIFactory.CreateLabel("Step 1 of 1", UIStyles.CaptionText);
        var nextButton     = UIFactory.CreateButton("Next →", UIStyles.BtnPrimary);

        footerRow.Add(prevButton);
        footerRow.Add(stepCountLabel);
        footerRow.Add(nextButton);
        footer.Add(footerRow);

        return new ModuleElements
        {
            ModuleTitleLabel  = moduleTitleLabel,
            InstructionsLabel = instructionsLabel,
            MediaContainer    = mediaContainer,
            StepList          = stepList,
            PreviousButton    = prevButton,
            NextButton        = nextButton,
            StepCountLabel    = stepCountLabel
        };
    }
}
```

---

## Views

Pure C#. Calls the builder, holds the element result, wires internal events, exposes them upward. No Unity API calls anywhere in this layer — if something requires `AudioListener`, `QualitySettings`, or a service call, it belongs in the Host.

### Login View

```csharp
public class LoginPanelView
{
    public event Action<string, string> OnLoginSubmitted;

    private readonly LoginElements _els;

    public LoginPanelView(VisualElement root)
    {
        _els = new LoginBuilder(root).Build();
        Wire();
    }

    private void Wire()
    {
        _els.LoginButton.clicked += OnLoginClicked;
    }

    private void OnLoginClicked()
    {
        ClearError();

        var username = _els.UsernameField.value.Trim();
        var password = _els.PasswordField.value;

        // Basic client-side validation before raising the event
        if (string.IsNullOrEmpty(username))
        {
            ShowError("Please enter your username.");
            return;
        }

        if (string.IsNullOrEmpty(password))
        {
            ShowError("Please enter your password.");
            return;
        }

        SetInteractable(false);
        OnLoginSubmitted?.Invoke(username, password);
    }

    // Called by Host on auth failure
    public void ShowError(string message)
    {
        _els.ErrorLabel.text = message;
        _els.ErrorLabel.RemoveFromClassList(UIStyles.Hidden);
        _els.UsernameField.AddToClassList(UIStyles.InputError);
    }

    public void ClearError()
    {
        _els.ErrorLabel.AddToClassList(UIStyles.Hidden);
        _els.UsernameField.RemoveFromClassList(UIStyles.InputError);
    }

    // Disabled while auth request is in flight, re-enabled on failure
    public void SetInteractable(bool on)
    {
        _els.LoginButton.SetEnabled(on);
        _els.UsernameField.SetEnabled(on);
        _els.PasswordField.SetEnabled(on);
        _els.LoginButton.text = on ? "Sign In" : "Signing in...";
    }

    // VR poke entry point — mirrors the click path exactly
    public void HandlePoke(string pokeId)
    {
        if (pokeId == PokeIds.Login) OnLoginClicked();
    }
}
```

### Settings View — Tab Switching

Tab state lives entirely in the View. The Host and Controller have no knowledge of which tab is active.

```csharp
public class SettingsPanelView
{
    public event Action         OnBackClicked;
    public event Action<string> OnThemeChanged;
    public event Action<bool>   OnHighContrastChanged;
    public event Action<float>  OnMasterVolumeChanged;
    public event Action<float>  OnMusicVolumeChanged;
    public event Action<float>  OnSFXVolumeChanged;

    private readonly SettingsElements _els;

    public SettingsPanelView(VisualElement root)
    {
        _els = new SettingsBuilder(root).Build();
        Wire();
    }

    private void Wire()
    {
        _els.BackButton.clicked += () => OnBackClicked?.Invoke();

        _els.VisualTab.clicked += () => SwitchTab(audio: false);
        _els.AudioTab.clicked  += () => SwitchTab(audio: true);

        _els.ThemeDropdown.RegisterValueChangedCallback(
            e => OnThemeChanged?.Invoke(e.newValue));

        _els.HighContrastToggle.RegisterValueChangedCallback(
            e => OnHighContrastChanged?.Invoke(e.newValue));

        RegisterVolumeSlider(_els.MasterSlider, _els.MasterValueLabel,
            v => OnMasterVolumeChanged?.Invoke(v));

        RegisterVolumeSlider(_els.MusicSlider, _els.MusicValueLabel,
            v => OnMusicVolumeChanged?.Invoke(v));

        RegisterVolumeSlider(_els.SFXSlider, _els.SFXValueLabel,
            v => OnSFXVolumeChanged?.Invoke(v));
    }

    // Shared helper avoids repeating the same three-line pattern per slider
    private void RegisterVolumeSlider(Slider slider, Label label, Action<float> onChange)
    {
        slider.RegisterValueChangedCallback(e =>
        {
            label.text = e.newValue.ToString("0%");
            onChange?.Invoke(e.newValue);
        });
    }

    private void SwitchTab(bool audio)
    {
        _els.VisualTab.EnableInClassList(UIStyles.BtnTabActive, !audio);
        _els.AudioTab.EnableInClassList(UIStyles.BtnTabActive,   audio);
        _els.VisualContent.EnableInClassList(UIStyles.Hidden,    audio);
        _els.AudioContent.EnableInClassList(UIStyles.Hidden,    !audio);
    }

    // Called by Host.Show() to sync sliders with saved values before panel opens
    public void SetValues(float master, float music, float sfx,
        string theme, bool highContrast)
    {
        // SetValueWithoutNotify prevents firing change events during initialisation
        _els.MasterSlider.SetValueWithoutNotify(master);
        _els.MasterValueLabel.text = master.ToString("0%");

        _els.MusicSlider.SetValueWithoutNotify(music);
        _els.MusicValueLabel.text = music.ToString("0%");

        _els.SFXSlider.SetValueWithoutNotify(sfx);
        _els.SFXValueLabel.text = sfx.ToString("0%");

        _els.ThemeDropdown.SetValueWithoutNotify(theme);
        _els.HighContrastToggle.SetValueWithoutNotify(highContrast);
    }

    public void HandlePoke(string pokeId)
    {
        switch (pokeId)
        {
            case PokeIds.Back:      OnBackClicked?.Invoke(); break;
            case PokeIds.TabVisual: SwitchTab(false);        break;
            case PokeIds.TabAudio:  SwitchTab(true);         break;
        }
    }
}
```

### Module View — Dynamic Content Per Step

```csharp
public class ModulePanelView
{
    public event Action OnPreviousClicked;
    public event Action OnNextClicked;

    private readonly ModuleElements _els;

    public ModulePanelView(VisualElement root)
    {
        _els = new ModuleBuilder(root).Build();
        Wire();
    }

    private void Wire()
    {
        _els.PreviousButton.clicked += () => OnPreviousClicked?.Invoke();
        _els.NextButton.clicked     += () => OnNextClicked?.Invoke();
    }

    // Called by Host each time the step index changes
    public void Populate(ModuleStepData step, int current, int total)
    {
        _els.ModuleTitleLabel.text  = step.ModuleTitle;
        _els.InstructionsLabel.text = step.Instructions;
        _els.StepCountLabel.text    = $"Step {current} of {total}";

        _els.PreviousButton.SetEnabled(current > 1);

        // Last step: change button label to signal completion
        _els.NextButton.text = current == total ? "Complete ✓" : "Next →";

        // Show media only if this step has an image assigned
        if (step.Media != null)
        {
            _els.MediaContainer.style.backgroundImage =
                new StyleBackground(step.Media);
            _els.MediaContainer.RemoveFromClassList(UIStyles.Hidden);
        }
        else
        {
            _els.MediaContainer.AddToClassList(UIStyles.Hidden);
        }

        RebuildStepList(step.Steps, current);
    }

    private void RebuildStepList(IList<string> steps, int activeIndex)
    {
        _els.StepList.Clear();

        for (int i = 0; i < steps.Count; i++)
        {
            var item  = UIFactory.CreateContainer(UIStyles.StepItem);
            var label = UIFactory.CreateLabel(steps[i], UIStyles.BodyText);
            item.Add(label);

            if (i < activeIndex - 1)
                item.AddToClassList(UIStyles.StepItemComplete);
            else if (i == activeIndex - 1)
                item.AddToClassList(UIStyles.StepItemActive);

            _els.StepList.Add(item);
        }
    }

    public void HandlePoke(string pokeId)
    {
        switch (pokeId)
        {
            case PokeIds.Next:     OnNextClicked?.Invoke();     break;
            case PokeIds.Previous: OnPreviousClicked?.Invoke(); break;
        }
    }
}
```

---

## VR Interactable Handshake

```csharp
public interface IPokeTarget
{
    void OnPoked(string buttonId);
}

public static class PokeIds
{
    public const string Login     = "login";
    public const string Back      = "back";
    public const string Next      = "next";
    public const string Previous  = "previous";
    public const string TabVisual = "tab-visual";
    public const string TabAudio  = "tab-audio";
}
```

The uGUI button sitting above the UITK panel calls `host.OnPoked(PokeIds.Next)` in its `onClick`. The Host passes it to `view.HandlePoke()`, which runs the same code path as a pointer click. Poke and pointer events are in sync with no duplication.

---

## Hosts

MonoBehaviour. Owns the panel lifecycle and handles everything that requires Unity APIs — audio, quality settings, scene loading, save calls. Views never do this.

### Base Host

```csharp
public abstract class BasePanelHost : MonoBehaviour
{
    [SerializeField] protected UIDocument uiDocument;
    [SerializeField] protected StyleSheet styleSheet;

    protected bool IsGenerated { get; private set; }
    protected VisualElement Root => uiDocument.rootVisualElement;

    public virtual void Generate()
    {
        if (IsGenerated) Dispose();
        if (styleSheet != null)
            Root.styleSheets.Add(styleSheet);
        IsGenerated = true;
    }

    public virtual void Show()
    {
        Root.RemoveFromClassList(UIStyles.Hidden);
        Root.AddToClassList(UIStyles.FadeIn);
    }

    public virtual void Hide()
    {
        Root.AddToClassList(UIStyles.Hidden);
        Root.RemoveFromClassList(UIStyles.FadeIn);
    }

    public virtual void Dispose()
    {
        Root.Clear();
        Root.styleSheets.Clear();
        IsGenerated = false;
    }
}
```

### Login Host

```csharp
public class LoginPanelHost : BasePanelHost, IPokeTarget
{
    public event Action<string, string> OnLoginSubmitted;

    private LoginPanelView _view;

    public override void Generate()
    {
        base.Generate();
        _view = new LoginPanelView(Root);
        _view.OnLoginSubmitted += (user, pass) => OnLoginSubmitted?.Invoke(user, pass);
        Show();
    }

    // Called by Controller if the auth service returns a failure
    public void NotifyLoginFailed(string reason)
    {
        _view?.ShowError(reason);
        _view?.SetInteractable(true);
    }

    public void OnPoked(string buttonId) => _view?.HandlePoke(buttonId);

    public override void Dispose()
    {
        _view = null;
        base.Dispose();
    }
}
```

### Settings Host

```csharp
public class SettingsPanelHost : BasePanelHost, IPokeTarget
{
    [SerializeField] private AppSettings _appSettings;

    public event Action OnBackRequested;

    private SettingsPanelView _view;

    public override void Generate()
    {
        base.Generate();
        _view = new SettingsPanelView(Root);

        _view.OnBackClicked         += () => OnBackRequested?.Invoke();
        _view.OnThemeChanged        += name  => ThemeManager.Instance.Apply(name);
        _view.OnHighContrastChanged += on    => ThemeManager.Instance.SetHighContrast(on);
        _view.OnMasterVolumeChanged += value =>
        {
            _appSettings.MasterVolume = value;
            AudioManager.SetMaster(value);
        };
        _view.OnMusicVolumeChanged += value =>
        {
            _appSettings.MusicVolume = value;
            AudioManager.SetMusic(value);
        };
        _view.OnSFXVolumeChanged += value =>
        {
            _appSettings.SFXVolume = value;
            AudioManager.SetSFX(value);
        };

        Hide(); // Settings starts hidden — shown by Controller on demand
    }

    public override void Show()
    {
        // Sync sliders with current saved values every time the panel opens
        _view?.SetValues(
            _appSettings.MasterVolume,
            _appSettings.MusicVolume,
            _appSettings.SFXVolume,
            _appSettings.Theme,
            _appSettings.HighContrast
        );
        base.Show();
    }

    public void OnPoked(string buttonId) => _view?.HandlePoke(buttonId);

    public override void Dispose()
    {
        _view = null;
        base.Dispose();
    }
}
```

### Module Host

```csharp
public class ModulePanelHost : BasePanelHost, IPokeTarget
{
    public event Action OnModuleCompleted;

    private ModulePanelView      _view;
    private List<ModuleStepData> _steps;
    private int                  _currentStep;

    public override void Generate()
    {
        base.Generate();
        _view = new ModulePanelView(Root);
        _view.OnNextClicked     += HandleNext;
        _view.OnPreviousClicked += HandlePrevious;
        Hide();
    }

    public void LoadModule(List<ModuleStepData> steps)
    {
        _steps       = steps;
        _currentStep = 1;
        RefreshView();
        Show();
    }

    private void HandleNext()
    {
        if (_currentStep >= _steps.Count)
        {
            OnModuleCompleted?.Invoke();
            return;
        }
        _currentStep++;
        RefreshView();
    }

    private void HandlePrevious()
    {
        if (_currentStep <= 1) return;
        _currentStep--;
        RefreshView();
    }

    private void RefreshView()
        => _view?.Populate(_steps[_currentStep - 1], _currentStep, _steps.Count);

    public void OnPoked(string buttonId) => _view?.HandlePoke(buttonId);

    public override void Dispose()
    {
        _view  = null;
        _steps = null;
        base.Dispose();
    }
}
```

---

## AppController

Single MonoBehaviour for the screen set. Owns navigation only — no UI element references, no builder calls, no direct View interaction.

```csharp
public class AppController : MonoBehaviour
{
    [Header("Hosts")]
    [SerializeField] private LoginPanelHost     _loginHost;
    [SerializeField] private DashboardPanelHost _dashboardHost;
    [SerializeField] private SettingsPanelHost  _settingsHost;
    [SerializeField] private ModulePanelHost    _moduleHost;

    // ── Lifecycle ────────────────────────────────────────────────────────────

    private void OnEnable()
    {
        GenerateAll();
        SubscribeAll();
        ShowOnly(_loginHost);
    }

    private void OnDisable()
    {
        UnsubscribeAll();
    }

    private void GenerateAll()
    {
        _loginHost.Generate();
        _dashboardHost.Generate();
        _settingsHost.Generate();
        _moduleHost.Generate();
    }

    // ── Subscriptions ────────────────────────────────────────────────────────

    private void SubscribeAll()
    {
        _loginHost.OnLoginSubmitted        += HandleLogin;
        _dashboardHost.OnSettingsRequested += () => ShowOnly(_settingsHost);
        _dashboardHost.OnModuleSelected    += HandleModuleSelected;
        _settingsHost.OnBackRequested      += () => ShowOnly(_dashboardHost);
        _moduleHost.OnModuleCompleted      += HandleModuleCompleted;
    }

    private void UnsubscribeAll()
    {
        _loginHost.OnLoginSubmitted        -= HandleLogin;
        _dashboardHost.OnSettingsRequested -= () => ShowOnly(_settingsHost);
        _dashboardHost.OnModuleSelected    -= HandleModuleSelected;
        _settingsHost.OnBackRequested      -= () => ShowOnly(_dashboardHost);
        _moduleHost.OnModuleCompleted      -= HandleModuleCompleted;
    }

    // ── Navigation ───────────────────────────────────────────────────────────

    private void ShowOnly(BasePanelHost target)
    {
        _loginHost.Hide();
        _dashboardHost.Hide();
        _settingsHost.Hide();
        _moduleHost.Hide();
        target.Show();
    }

    // ── Handlers ─────────────────────────────────────────────────────────────

    private void HandleLogin(string username, string password)
    {
        // Replace with real auth service call
        // On success:
        _dashboardHost.SetUserData(username, progress: 0.4f);
        _dashboardHost.SetModules(ModuleService.GetAll());
        ShowOnly(_dashboardHost);

        // On failure:
        // _loginHost.NotifyLoginFailed("Invalid credentials.");
    }

    private void HandleModuleSelected(string moduleId)
    {
        var steps = ModuleService.GetSteps(moduleId);
        _moduleHost.LoadModule(steps);
        ShowOnly(_moduleHost);
    }

    private void HandleModuleCompleted()
    {
        // Update progress data, unlock next module, return to dashboard
        ShowOnly(_dashboardHost);
    }
}
```

---

## File Structure

```
Assets/Scripts/UI/
├── Common/
│   ├── UIStyles.cs
│   └── IPokeTarget.cs
├── Factory/
│   └── UIFactory.cs
├── Builder/
│   ├── PanelBuilder.cs
│   ├── LoginBuilder.cs
│   ├── DashboardBuilder.cs
│   ├── SettingsBuilder.cs
│   └── ModuleBuilder.cs
├── Views/
│   ├── LoginPanelView.cs
│   ├── DashboardPanelView.cs
│   ├── SettingsPanelView.cs
│   └── ModulePanelView.cs
├── Hosts/
│   ├── BasePanelHost.cs
│   ├── LoginPanelHost.cs
│   ├── DashboardPanelHost.cs
│   ├── SettingsPanelHost.cs
│   └── ModulePanelHost.cs
├── Controllers/
│   └── AppController.cs
└── Models/
    ├── ModuleData.cs
    └── ModuleStepData.cs
```
