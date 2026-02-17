# MVVM in UI Toolkit — Implementation Reference

> **Purpose**: Comparison point against the Controller → Host → View architecture  
> **Context**: Unity 6.3 UITK with runtime data binding  
> **Honest framing**: MVVM is not universally better — it solves different problems

---

## What MVVM Actually Is

Model-View-ViewModel is a pattern built around **data binding as the primary communication mechanism** between layers. The ViewModel exposes observable state; the View binds to it. Neither layer calls the other directly.

```
Model  ←→  ViewModel  ←→  View
             ↑                ↑
         pure C#          UXML/USS/C#
         no Unity         no business logic
```

The key distinction from MVC or MVP: the ViewModel doesn't hold a reference to the View. The View holds a reference to the ViewModel and observes it. This inversion is what makes ViewModels unit-testable without instantiating any UI.

---

## The Three Layers

### Model

Raw data and business logic. No Unity dependencies, no presentation concerns. A `PlayerStats` ScriptableObject or a plain C# class representing inventory state. Models don't know they're being displayed.

### ViewModel

Transforms Model data into something the View can display directly. Contains:

- Observable properties (wrapped values that notify on change)
- Formatted strings (`$"{health}/{maxHealth}"`, percentages, timestamps)
- Commands (encapsulated actions the View can invoke)
- Validation logic
- State derived from multiple Model properties

Does **not** contain: Unity API calls, scene references, MonoBehaviour inheritance.

### View

Binds to ViewModel properties and calls ViewModel commands. In UITK this means registering callbacks on ViewModel observable properties and updating elements in response. Contains no logic — only wiring.

---

## Observable Property — The Foundation

Everything in MVVM depends on change notification. The simplest implementation:

```csharp
public class Observable<T>
{
    private T _value;

    public event Action<T> Changed;

    public T Value
    {
        get => _value;
        set
        {
            if (EqualityComparer<T>.Default.Equals(_value, value)) return;
            _value = value;
            Changed?.Invoke(_value);
        }
    }

    public Observable(T initial = default) => _value = initial;
}
```

The equality check matters — without it every assignment fires a notification regardless of whether the value changed, which on a frequently-updated HUD will cause unnecessary UI updates.

Unity 6 native binding via `SerializedObject` is an alternative for ScriptableObject-backed Models, but `Observable<T>` works for any C# class and doesn't require Unity's serialization pipeline.

---

## ViewModel Implementation

```csharp
// No MonoBehaviour, no Unity using directives needed
public class HealthViewModel
{
    // Observable state the View binds to
    public Observable<string> HealthText { get; } = new();
    public Observable<float> HealthFraction { get; } = new();
    public Observable<bool> IsCritical { get; } = new();

    // Command the View can invoke
    public Action OnHealRequested { get; set; }

    private readonly PlayerStats _stats;

    public HealthViewModel(PlayerStats stats)
    {
        _stats = stats;
        _stats.HealthChanged += Refresh;
        Refresh();
    }

    private void Refresh()
    {
        float fraction = (float)_stats.Health / _stats.MaxHealth;
        HealthText.Value = $"{_stats.Health} / {_stats.MaxHealth}";
        HealthFraction.Value = fraction;
        IsCritical.Value = fraction < 0.25f;
    }

    public void Dispose()
    {
        _stats.HealthChanged -= Refresh;
    }
}
```

Notice what's not here: no `label.text =`, no `AddToClassList`, no `UIDocument`. The ViewModel has no idea what kind of UI is observing it. You could bind this to a UITK panel, a uGUI canvas, or a unit test assertion — the ViewModel doesn't change.

---

## View Implementation (UITK)

The View's only job is to wire ViewModel observables to UI elements:

```csharp
public class HealthView
{
    private readonly HealthViewModel _vm;
    private readonly Label _healthLabel;
    private readonly ProgressBar _healthBar;
    private readonly Button _healButton;
    private readonly VisualElement _root;

    public HealthView(VisualElement root, HealthViewModel vm)
    {
        _root = root;
        _vm = vm;

        // Query elements — cache on construction, never query in callbacks
        _healthLabel = root.Q<Label>("health-label");
        _healthBar = root.Q<ProgressBar>("health-bar");
        _healButton = root.Q<Button>("heal-button");

        Bind();
    }

    private void Bind()
    {
        // Observable → UI element
        _vm.HealthText.Changed += text => _healthLabel.text = text;
        _vm.HealthFraction.Changed += f => _healthBar.value = f * 100f;
        _vm.IsCritical.Changed += critical =>
        {
            _root.EnableInClassList("health--critical", critical);
            _root.EnableInClassList("health--normal", !critical);
        };

        // UI element → ViewModel command
        _healButton.clicked += () => _vm.OnHealRequested?.Invoke();

        // Initialise with current values without waiting for a change
        _healthLabel.text = _vm.HealthText.Value;
        _healthBar.value = _vm.HealthFraction.Value * 100f;
    }

    public void Dispose()
    {
        // Unsubscribe to prevent memory leaks
        _vm.HealthText.Changed -= text => _healthLabel.text = text;
        _vm.HealthFraction.Changed -= f => _healthBar.value = f * 100f;
        // etc.
    }
}
```

**A note on disposal**: the lambda unsubscription pattern above doesn't actually work — you can't unsubscribe a lambda that wasn't stored. In practice, store callbacks as named fields or use a disposable subscription list. This is a common MVVM footgun in C#.

Better disposal pattern:

```csharp
private readonly List<Action> _unbinders = new();

private void Bind()
{
    Register(_vm.HealthText.Changed, text => _healthLabel.text = text);
    // ...
}

private void Register<T>(ref event Action<T> evt, Action<T> handler)
{
    evt += handler;
    _unbinders.Add(() => evt -= handler);
}

public void Dispose()
{
    foreach (var unbind in _unbinders) unbind();
    _unbinders.Clear();
}
```

Or use a disposable wrapper pattern. Either way, disposal is where MVVM implementations most commonly leak.

---

## Where MVVM Fits in the Host → View Architecture

MVVM isn't a replacement for your Controller → Host → View architecture. The ViewModel slots **between Host and View** for panels that have presentation logic:

```
Controller   (scene orchestration — unchanged)
    │
    ▼
Host         (lifecycle, Unity APIs, creates ViewModel)
    │
    ▼
ViewModel    (presentation logic, observable state)
    │
    ▼
View         (binds to ViewModel, no logic)
```

For panels without presentation logic (a four-button start menu), ViewModel adds nothing — the Host can pass data directly to the View. The test for whether a ViewModel is justified: **does the View need to display anything other than the raw value it received?** If yes, ViewModel. If no, skip it.

---

## Unit Testing the ViewModel

The core payoff of the pattern:

```csharp
[Test]
public void HealthText_FormatsCorrectly_WhenHealthChanges()
{
    var stats = new PlayerStats { Health = 75, MaxHealth = 100 };
    var vm = new HealthViewModel(stats);

    // Capture the observable output
    string captured = null;
    vm.HealthText.Changed += text => captured = text;

    stats.Health = 50;
    stats.NotifyHealthChanged();

    Assert.AreEqual("50 / 100", captured);
}

[Test]
public void IsCritical_True_WhenHealthBelowThreshold()
{
    var stats = new PlayerStats { Health = 100, MaxHealth = 100 };
    var vm = new HealthViewModel(stats);

    stats.Health = 20;
    stats.NotifyHealthChanged();

    Assert.IsTrue(vm.IsCritical.Value);
}
```

No scene, no UIDocument, no MonoBehaviour. This runs in Unity's test runner without entering play mode. The value scales with panel complexity — a character sheet ViewModel with twelve computed properties and three validation rules is genuinely worth testing this way.

---

## Where MVVM Creates Problems

**Binding disposal is error-prone.** Every subscription needs a corresponding unsubscription. Forget one and you have a reference keeping a disposed View alive. This is less of an issue in your Host-managed lifecycle because the Host's `Dispose()` is explicit and called in a known place — but it still requires discipline.

**Observable chains get complex.** A ViewModel property that depends on three Model properties and a user preference requires careful ordering of notifications. If the wrong thing fires first, the View gets an intermediate invalid state. `HealthFraction` that depends on both `Health` and `MaxHealth` needs both to be set before notifying, or you get a flash of incorrect values.

**It's boilerplate-heavy for simple cases.** A settings slider that maps directly to a float value needs an `Observable<float>`, a subscription in the View, and a disposal registration — for something a direct `Bind()` call on a SerializedObject would handle in one line. Use Unity's native binding for the simple cases.

**Testing the View itself is still hard.** The ViewModel being testable doesn't mean the visual result is verified. You can confirm that `HealthText.Value` is `"50 / 100"` without ever confirming that the label actually shows that text in the panel. Integration testing of the rendered output is still the unsolved problem.

---

## MVVM vs Controller → Host → View: The Honest Comparison

| Concern           | Controller → Host → View                   | With MVVM (Host + ViewModel + View)                |
| ----------------- | ------------------------------------------ | -------------------------------------------------- |
| Simple panels     | Clean, minimal                             | Overengineered                                     |
| Complex panels    | Logic leaks into Host or View              | Contained in testable ViewModel                    |
| Designer-editable | Host is MonoBehaviour — inspector-friendly | ViewModel is pure C# — no inspector                |
| Unit testing      | Host has Unity deps, hard to test          | ViewModel has no Unity deps, easy to test          |
| Boilerplate       | Low                                        | Higher — Observable<T> wiring throughout           |
| Lifecycle clarity | Explicit (Generate/Show/Hide/Dispose)      | Depends on Host — same lifecycle, just more layers |
| Learning curve    | Low for Unity devs                         | Moderate — subscription patterns require care      |

Neither is universally better. The architecture you have is the right default; MVVM is the right addition for panels with non-trivial presentation logic.

---

## Practical Decision

Before writing a ViewModel, answer these:

1. Does the panel display formatted or computed values? (percentages, timestamps, derived stats)
2. Does the panel have validation? (form inputs, error states)
3. Does the panel have state that affects display but isn't in the Model? (selected item, active filter)
4. Would you want to unit-test this panel's logic?

If one or more is yes, write the ViewModel. If all are no, the Host-to-View direct approach is sufficient and cleaner.

---

*Observable<T> implementation shown is illustrative — validate against your project's existing event patterns before adopting. Disposal pattern requires more robust implementation than the naive lambda approach shown.*


