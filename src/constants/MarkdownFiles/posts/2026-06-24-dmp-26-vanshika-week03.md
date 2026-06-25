---
title: "DMP '26 Week 03 Update by Vanshika Pahal"
excerpt: "Week 03: Modularizing UI and Controllers. Deconstructing grid rendering, plugin lifecycles, execution controllers, alerts, and toolbars into separate widgets and controller layers in Music Blocks v3."
category: "DEVELOPER NEWS"
date: "2026-06-24"
slug: "2026-06-24-dmp-26-vanshika-week03"
author: "@/constants/MarkdownFiles/authors/vanshika2720.md"
tags: "dmp26,sugarlabs,musicblocks,refactoring,week03,modularization"
image: "assets/Images/dmp_c4gt.logo.png"
---

<!-- markdownlint-disable -->

# Week 03 Progress Report by Vanshika Pahal

**Project:** [Music Blocks v3 — Test Coverage, Refactoring & Dependency Updates](https://github.com/sugarlabs/musicblocks)  
**Mentors:** [Walter Bender](https://github.com/walterbender), [Sumit Srivastava](https://github.com/sum2it)  
**Assisting Mentors:** [Devin Ulibarri](https://github.com/pikurasa), [Om Santosh Suneri](https://github.com/omsuneri)  
**Organization:** [Sugar Labs](https://sugarlabs.org)  
**Week:** UI vs. Controller Separation & Module Shimming  
**Reporting Period:** 2026-06-18 – 2026-06-24

---

## Overview

Following up on the initial refactoring of self-contained subsystems in Week 2, **Week 3 focused on separating UI interactions and rendering concerns from underlying business and state execution logic**. 

The main monolithic structure `activity.js` was further stripped of direct DOM manipulations, EaselJS rendering pathways, and controller states. Over the course of this week, I modularized grid overlays, plugin lifecycle and dialog systems, runtime toolbar controls, alert renderers, and the visual toolbar representation. By utilizing the **RequireJS compatibility shim pattern**, I isolated visual components (widgets) and logic handlers (controllers) without introducing breaking changes or violating the existing API surface.

Across these changes, I worked on **6 pull requests**—5 of which are already merged into the sugarlabs master, with 1 currently open under active review—resulting in **over 6,800 additions and 3,800 deletions** across the codebase.

---

## Modularization & UI Separation at a Glance

Here is a summary of the architectural changes and pull requests handled this week:

| Pull Request | Subsystem / Component | Target File(s) | Impact & Code Changes | Status |
| :--- | :--- | :--- | :--- | :---: |
| **[PR #7572](https://github.com/sugarlabs/musicblocks/pull/7572)** | Grid Rendering Module | `js/activity/grid-renderer.js` | Extracted grid rendering methods (showing/hiding overlays) from `activity.js`. | **Merged** |
| **[PR #7581](https://github.com/sugarlabs/musicblocks/pull/7581)** | Plugin Controller | `js/activity/plugin-controller.js` | Extracted plugin lifecycle and persistence logic from `activity.js`. | **Merged** |
| **[PR #7584](https://github.com/sugarlabs/musicblocks/pull/7584)** | Plugin Dialog UI Widget | `js/widgets/plugin-dialog.js` | Decoupled plugin-related UI / file chooser dialogs from Activity. | **Merged** |
| **[PR #7622](https://github.com/sugarlabs/musicblocks/pull/7622)** | Toolbar Controller | `js/activity/toolbar-controller.js` | Extracted execution-state & Logo runtime controls (Fast, Slow, Step, Hard Stop). | **Merged** |
| **[PR #7639](https://github.com/sugarlabs/musicblocks/pull/7639)** | Alert Renderer & Controller | `js/activity/alert-renderer.js` | Decoupled alert state management from visual rendering (EaselJS/DOM msg container). | **Merged** |
| **[PR #7628](https://github.com/sugarlabs/musicblocks/pull/7628)** | Visual Toolbar UI Widget | `js/widgets/toolbar-ui.js` | Extracted visual toolbar (DOM elements, colors, icons, focus cycling) into a widget. | **Open** |

*Total changes: **+6,837 additions** and **-3,836 deletions** across these PRs, with all Jest test suites passing.*

---

## Detailed Breakdown of Extracted Components

### 1. Grid Renderer (PR #7572)

To clean up visual overlays (Cartesian, Polar, and music staff grids: Treble, Grand, Soprano, Alto, Tenor, Bass, and Accidentals), the rendering methods were removed from `activity.js` and consolidated.

* **Changes:** Created `js/activity/grid-renderer.js` which registers `setupGridRenderer()`. This wires show/hide drawing methods back onto the existing Activity context.
* **Preservation:** Grid states, canvas bitmap references, and the bitmap initialization helper `_createGrid()` were intentionally kept in `activity.js` to ensure the core canvas drawing contexts remain intact.
* **Compatibility:** Because methods are wired back dynamically during startup, `grid-controller.js` still references methods like `this.activity._showCartesian()` and `this.activity._hideCartesian()` identically, ensuring backward compatibility.

### 2. Plugin Lifecycle & Dialog UI (PR #7581 & #7584)

The plugin system in Music Blocks previously had state management, file-loading listeners, and browser modal popup dialogs intertwined within `activity.js`. I split this subsystem into a dedicated logic controller and a separate UI widget.

* **Plugin Controller (PR #7581):** Extracted plugin state initialization, loading of built-in/stored/local plugins, registration, and local storage persistence into `js/activity/plugin-controller.js`. I added automated tests (`js/activity/__tests__/plugin-controller.test.js`) covering these lifecycle actions.
* **Plugin Dialog (PR #7584):** Decoupled the visual prompts, file chooser interactions, and event listeners into a dedicated widget in `js/widgets/plugin-dialog.js`.
* **Activity Integration:** The Activity orchestrator now acts as a high-level router: loading state triggers, cursor updates, and palette refreshes remain in the activity while delegating logic to the controller and prompts to the widget via callbacks.

```javascript
// PluginDialog decoupling using option callbacks
const dialog = new PluginDialog({
    onLoadBuiltin: (name) => this._loadBuiltInPlugin(name),
    onLoadFile: (file) => this._loadLocalPluginFile(file)
});
```

### 3. Toolbar Controller & Visual Toolbar UI (PR #7622 & #7628)

The main toolbar is the primary user interface for running Logo programs (Fast Run, Slow Run, Step Run, Hard Stop). Just like plugins, the execution states and visual representations were completely entangled.

* **Toolbar Controller (PR #7622):** Extracted execution state flags, run-mode transitions, turtle speed configuration, and logo runtime execution hooks to `js/activity/toolbar-controller.js`. Added a full test suite in `js/activity/__tests__/toolbar-controller.test.js` to cover state transitions.
* **Visual Toolbar UI (PR #7628):** Moved all DOM calculations, toolbar rendering, button highlight states, button colors, and keyboard focus cycling from `js/toolbar.js` to `js/widgets/toolbar-ui.js`.
* **The Shim Pattern:** To support existing code files that depend on RequireJS imports of `activity/toolbar`, I turned the original `js/toolbar.js` into a thin compatibility wrapper that imports and returns the new `widgets/toolbar-ui.js` module. Added tests in `js/widgets/__tests__/toolbar-ui.test.js` using Jest fake timers to test button highlights and dim-restore cycles.

### 4. Alert Renderer & Controller (PR #7639)

Alerts in Music Blocks display simple notification banners or highly styled EaselJS/DOM visual graphics when compiler errors happen. 

* **Changes:** Separated alert lifecycle management from raw visual presentation. 
* **Alert Controller:** Retains logic for timers, message queuing, and timeout scheduling.
* **Alert Renderer:** Created `js/activity/alert-renderer.js` to manage all visual elements, including the drawing of the error artwork canvas, message containers, and target arrow indicators. Added unit tests in `js/activity/__tests__/alert-renderer.test.js` verifying DOM layout changes and dismissal behaviors.

---

## Architectural Impact: Separation of Concerns

Our ongoing modularization campaign has successfully changed how components interact:

1. **Logical vs. Presentation Decoupling:** Modules are no longer mixed. Controllers handle state transitions and business logic; Widgets own DOM manipulations and canvas rendering.
2. **Explicit Interfaces:** Replaced implicit visual mutations with clear event callbacks.
3. **No-Regression Shimming:** Using thin AMD compatibility wrappers avoids the need for massive refactoring of third-party plugins or scripts that rely on the old paths, meaning we maintain a stable runtime environment throughout the process.

---

## Key Learnings

1. **Fake Timers for UI Assertions:** When refactoring UI classes like `ToolbarUI` or `AlertRenderer` that rely on timeouts (e.g., dimming a button and restoring it after 200ms), using Jest's `jest.useFakeTimers()` is crucial to run deterministic assertions without slowing down tests.
2. **Dynamic Wiring vs. Clean Shims:** While dynamic wiring allows backward-compatible delegation, using formal RequireJS path registrations and compatibility shims keeps dependency management clear and readable.

---

## Roadmap for Week 04

For the upcoming week, the focus will shift to search indexing and the core interpreter environment:

### 1. Search Subsystem Extraction
I will extract the search mechanism inside `activity.js` into two modules:
* **Search Controller (`js/activity/search-controller.js`):** Responsible for search indexing, filtering logic, and result generation.
* **Search Widget (`js/widgets/search-widget.js`):** Responsible for DOM rendering, click event handlers, dropdown animations, and visual search states.

### 2. Auditing Logo Globals (`js/logo.js`)
* **The Problem:** The Logo interpreter (`js/logo.js` - 2,699 lines) reads directly from unscoped globals (`Singer`, `Turtle`, `blockList`, `boxes`) during code execution. Writing unit tests for any interpreter path currently requires setting up a fragile, 200+ line global mock block.
* **The Plan:** Audit these implicit reads and begin designing a `LogoDependencies` injection system to pass scopes explicitly, simplifying the testing setup.

---

## Acknowledgements

I want to extend my gratitude to my mentor **Walter Bender** for checking the structural designs, merging these pull requests, and validating the compatibility behaviors. Thanks as well to the GSoC/DMP peers at Sugar Labs for review discussions and suggestions on Codecov.
