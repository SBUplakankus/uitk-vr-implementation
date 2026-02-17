# Controller → Host → View Architecture

> **Based on**: Actual implementation from the Avatar Academy UITK project  
> **Pattern**: Factory → View → Host → Controller  
> **Context**: Code-generated UITK panels with MonoBehaviour lifecycle management

---

## What the Architecture Is

Three layers with distinct responsibilities:

```
StartMenuController          (MonoBehaviour — scene orchestration)
    │
    └─► StartMenuPanelHost   (MonoBehaviour — panel lifecycle)
            │
            └─► StartMenuPanelView  (Pure C# — UI generation via Factory)
```

**Controller** — scene-level. Knows about multiple panels and how they relate. Handles navigation, application logic, and coordinates which panels are visible.

**Host** — panel-level. Owns a single panel's lifecycle: Generate(), Show(), Hide(), Dispose(). Acts as the boundary between Unity (MonoBehaviour, UIDocument) and the pure C# view. Exposes events upward to the Controller.

**View** — pure C#, no Unity dependencies. Uses the Factory to create elements, wires up internal events, and exposes them for the Host to subscribe to. Has no knowledge of scene state or other panels.

This split — scene-level vs panel-level — is the most useful structural decision in the architecture. It means a Controller can coordinate three panels without any of those panels knowing about each other.

---

## Where It Sits in MVVM Terms

| Your Layer | Nearest MVVM Equivalent                                |
| ---------- | ------------------------------------------------------ |
| Controller | Application Controller / Navigator                     |
| Host       | View Controller                                        |
| View       | View                                                   |
| *(absent)* | ViewModel — only needed when presentation logic exists |

The ViewModel layer is intentionally absent for simple panels and that's correct. A start menu with four buttons and no state has no presentation logic to encapsulate. Adding a ViewModel there would be structure for its own sake.

---

## Suggested Improvements to StartMenuController

These are based on partial code visibility — verify against what's actually there before applying.

### 1. Redundant SubscribeEvents() call

If `Generate()` in the Host already calls `SubscribeEvents()` internally, calling it again from `BindButtons()` in the Controller doubles up the subscription. If that's the case, remove the explicit call from `BindButtons()` and let Host manage its own internal wiring. The Controller should only be subscribing to events the Host exposes publicly.

### 2. Settings panel regenerating on every toggle

Calling `Generate()` each time settings are toggled recreates the entire UI tree. For a settings panel that doesn't change structure between openings, the better approach is to generate once at startup and then use `Show()` / `Hide()` for visibility toggling. The tradeoff is memory (the panel exists even when hidden) vs. CPU cost (rebuilding the tree on each open). For a settings panel, always-in-memory is the right call.

```csharp
private void OnEnable()
{
    _startMenuPanel.Generate();
    _settingsPanel.Generate();
    _settingsPanel.Hide();       // generated but hidden
    SubscribeToEvents();
}

private void ToggleSettings()
{
    _settingsActive = !_settingsActive;
    if (_settingsActive) _settingsPanel.Show();
    else _settingsPanel.Hide();
}
```

### 3. ShowPanel() helper for mutual exclusion

If you have more than two panels that need to be mutually exclusive, a helper that hides all and shows one is cleaner than managing show/hide pairs manually:

```csharp
private void ShowPanel(BasePanelHost target)
{
    _startMenuPanel.Hide();
    _settingsPanel.Hide();
    target.Show();
}
```

This scales without accumulating pairwise show/hide logic when you add a third or fourth panel.

---

## When to Add a ViewModel

The decision is straightforward: add a ViewModel when the View needs to display something other than raw data. If the panel contains formatting, validation, computed values, or state that affects how data is presented, that logic belongs in a ViewModel rather than the View.

| Panel type                   | Needs ViewModel? | Why                                       |
| ---------------------------- | ---------------- | ----------------------------------------- |
| Button-only menus            | No               | No data to transform                      |
| Pause / confirmation dialogs | No               | No state                                  |
| Settings with sliders        | Yes              | Volume needs formatting, validation       |
| Inventory                    | Yes              | Filtering, sorting, selection state       |
| Character sheet              | Yes              | Computed stats, percentage bars           |
| Live HUD                     | Maybe            | Depends on whether values need formatting |

When you do add a ViewModel, it slots between Host and View. The Host creates it, passes it to the View, and subscribes to its events for anything that requires Unity APIs:

```
Host creates ViewModel
Host passes ViewModel into View constructor
View binds UI elements to ViewModel properties
ViewModel raises events → Host handles Unity-specific side effects
    (e.g. AudioListener.volume, QualitySettings, Save())
```

The ViewModel itself stays pure C# and testable. The Host is the only place where ViewModel events result in Unity API calls.

---

## Directory Structure

```
Assets/Scripts/UI/
├── Controllers/          # Scene-level orchestration
├── Hosts/
│   └── Base/             # BasePanelHost
├── Views/
│   └── Base/             # BasePanelView
├── ViewModels/           # Add as panels require them
├── Models/               # Data structures passed to ViewModels
└── Common/               # Shared utilities (ObservableProperty, etc.)
```

`ViewModels/` starts empty and grows as panels accumulate presentation logic. Keeping it separate from Views makes the distinction visible — if you're writing formatting or validation logic in a View file, it's a signal it should move.

---

## What This Architecture Is Good At

**Lifecycle is explicit.** Generate, Show, Hide, and Dispose are named operations on the Host rather than implicit behaviour scattered across OnEnable/OnDisable. This makes panel state predictable.

**The Controller doesn't touch UITK.** All UIDocument references live in Hosts. The Controller only calls methods on Hosts and handles events from them. If the UI implementation changes (e.g. switching a panel from code-generated to UXML-driven), the Controller doesn't change.

**Views are testable without Unity.** Pure C# Views with no MonoBehaviour dependency can be instantiated in unit tests if needed. The Factory they call also has no Unity dependencies in principle.

---

## What to Watch as It Scales

**Host proliferation.** As the project grows, you'll have many Hosts. Consider whether BasePanelHost needs more shared behaviour (e.g. standard fade animations) or whether each Host reimplements the same Show/Hide pattern independently.

**Event chain length.** Currently: View fires event → Host rebroadcasts → Controller handles. For simple panels that's fine. If you find Controllers subscribing to events that tunnel through multiple Hosts, consider whether a shared event bus or direct reference is cleaner at that point.

**Factory as the single source of element creation.** The Factory approach means visual consistency is enforced at creation time rather than per-panel. As the style system grows (ScriptableObject themes, USS variable swapping), the Factory is the natural place to apply those — keep it the only place elements are instantiated.

---

*Architecture description reflects the actual implementation. Improvement suggestions are provisional — verify against the full codebase before applying.*
