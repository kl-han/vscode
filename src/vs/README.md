# VS Code source overview: layers, parts, and components

This document maps the components that make up VS Code — first the **architectural layers** the source tree is split into, then the **workbench structure**, and finally a **component map** that connects each user-facing feature (what you see in the product) to the code that implements it, with a note on why each component exists.

It complements, and links to, the canonical references:

- [`.github/copilot-instructions.md`](../../.github/copilot-instructions.md) — repository-wide architecture summary
- [`.github/instructions/source-code-organization.instructions.md`](../../.github/instructions/source-code-organization.instructions.md) — layering and organization rules in detail
- [Source Code Organization](https://github.com/microsoft/vscode/wiki/Source-Code-Organization) on the upstream wiki

## Layers

Code in `src/vs` is organized in layers. Each layer may only depend on the layers below it — the rules are enforced by ESLint (`local/code-import-patterns` and `local/code-layering` in [`eslint.config.js`](../../eslint.config.js)); run `npm run valid-layers-check` to validate.

| Layer | Path | Purpose | May import from |
|-------|------|---------|-----------------|
| **base** | [`src/vs/base`](./base) | General utilities and UI building blocks with no service dependencies: arrays, events, lifecycle/`Disposable`, async, DOM helpers; `base/parts` holds larger reusable pieces such as IPC. | base only |
| **platform** | [`src/vs/platform`](./platform) | Dependency-injection infrastructure (`instantiation`) plus ~100 shared services: files, configuration, keybinding, actions, commands, log, telemetry, extension management, and more. Services declare a decorator identifier and register via `registerSingleton`. | base |
| **editor** | [`src/vs/editor`](./editor) | The Monaco editor core: text model, view, languages, diffing. `editor/contrib` holds editor features; `editor/standalone` is the standalone Monaco distribution. Must stay free of `node`/`electron` dependencies so it can ship as Monaco. | base, platform |
| **workbench** | [`src/vs/workbench`](./workbench) | The full application frame: workbench parts (side bars, editor area, panel, status bar), core services, feature contributions (`contrib/`), and the extension-host API implementation (`api/`). | base, platform, editor |
| **code** | [`src/vs/code`](./code) | Desktop (Electron) entry points: the Electron main process, CLI process, shared/utility process, and the web workbench host page. | base, platform, editor (+ workbench entry bundles in the browser layer) |
| **server** | [`src/vs/server`](./server) | The remote development server: remote extension host agent, its CLI, remote file system and extension-management channels. | base, platform, workbench |
| **sessions** | [`src/vs/sessions`](./sessions) | The Agents Window — a simplified, sessions-first workbench for agent workflows, with its own entry points and `contrib/`/`services/` tree. `vs/workbench` must never import from `vs/sessions`. See [`src/vs/sessions/README.md`](./sessions/README.md). | workbench and below |

The standalone Rust crate in [`cli/`](../../cli) (the `code` launcher and tunnels binary) sits outside the TypeScript layering entirely.

### Target environments

Within every layer, code is split by runtime environment, and may only import sideways or down:

| Folder | Runs in | May import from |
|--------|---------|-----------------|
| `common` | Everywhere (pure JS) | `common` |
| `browser` | Browser / renderer (DOM APIs) | `common`, `browser` |
| `node` | Node.js | `common`, `node` |
| `electron-browser` | Electron renderer with limited IPC | `common`, `browser`, `electron-browser` |
| `electron-utility` | Electron utility process | `common`, `node` |
| `electron-main` | Electron main process | `common`, `node`, `electron-utility` |

A feature that works in the web build must live in `common`/`browser`; anything importing `node` or `electron-*` is desktop- or server-only.

## Workbench structure

[`src/vs/workbench`](./workbench) is where the product comes together:

- [`browser/parts`](./workbench/browser/parts) — the window chrome, one folder per part: `titlebar`, `activitybar`, `sidebar`, `auxiliarybar`, `editor`, `panel`, `statusbar`, `notifications`, `banner`, `dialogs`, plus the shared composite-bar machinery. [`browser/layout.ts`](./workbench/browser/layout.ts) arranges them into the grid.
- [`services/`](./workbench/services) — core workbench services not tied to one feature (text files, keybinding, themes, lifecycle, history, layout).
- [`contrib/`](./workbench/contrib) — self-contained feature contributions, one folder per feature (~100 of them: `terminal`, `debug`, `search`, `scm`, `chat`, …). A contribution has a single `*.contribution.ts` entry point, exposes at most one common API file to other contribs, and no code outside `contrib/` may depend on it.
- [`api/`](./workbench/api) — the extension host: the implementation of `vscode.d.ts` (`extHost*.ts`) and the `mainThread*` counterparts.

**Entry points are import lists.** [`workbench.common.main.ts`](./workbench/workbench.common.main.ts) imports every browser-safe contribution; [`workbench.desktop.main.ts`](./workbench/workbench.desktop.main.ts) adds Electron-only ones; [`workbench.web.main.ts`](./workbench/workbench.web.main.ts) is the web equivalent. Only code reachable from an entry point ships in a build.

### How a feature registers itself

Features self-register by side effect when their `*.contribution.ts` is imported from an entry point. The common registration styles, all rooted in the generic [`platform/registry`](./platform/registry):

```ts
// workbench lifecycle hook (phases control startup timing)
registerWorkbenchContribution2(MyFeature.ID, MyFeature, WorkbenchPhase.Eventually);

// a service
registerSingleton(IMyService, MyService, InstantiationType.Delayed);

// a command / menu action with keybinding
registerAction2(class extends Action2 { /* id, title, keybinding, menu */ });

// a view container (Activity Bar icon) and its views
Registry.as<IViewContainersRegistry>(ViewContainerExtensions.ViewContainersRegistry)
    .registerViewContainer({ ... });
Registry.as<IViewsRegistry>(ViewContainerExtensions.ViewsRegistry).registerViews([...]);
```

A small real example is [`contrib/emergencyAlert/electron-browser/emergencyAlert.contribution.ts`](./workbench/contrib/emergencyAlert/electron-browser/emergencyAlert.contribution.ts); a richer one showing services, views, and actions together is [`contrib/output/browser/output.contribution.ts`](./workbench/contrib/output/browser/output.contribution.ts).

## Component map

Where to find the code behind each user-facing component. Components are grouped the way users encounter them; the *why* column states the need the component answers, because it usually explains the design of the code too.

### Workbench shell

The shell exists to keep code central while tools stay one keystroke away. Everything here lives under [`workbench/browser`](./workbench/browser).

| Component | Why it exists | Source |
|-----------|---------------|--------|
| Activity Bar | One always-visible icon per tool surface, with badges | [`browser/parts/activitybar`](./workbench/browser/parts/activitybar), [`browser/parts/globalCompositeBar.ts`](./workbench/browser/parts/globalCompositeBar.ts) (Accounts + Manage) |
| Primary Side Bar | Persistent tool views beside the code | [`browser/parts/sidebar`](./workbench/browser/parts/sidebar) |
| Secondary Side Bar | A second dock so two tool surfaces coexist (default home of Chat) | [`browser/parts/auxiliarybar`](./workbench/browser/parts/auxiliarybar) |
| Editor area | Grid of editor groups with tabs, splitting, floating windows | [`browser/parts/editor`](./workbench/browser/parts/editor) (`editorPart.ts`, `editorGroupView.ts`, tab controls) |
| Panel | Wide, log-like views (terminal, problems) below the code | [`browser/parts/panel`](./workbench/browser/parts/panel) |
| Status Bar | Glanceable, clickable ambient state | [`browser/parts/statusbar`](./workbench/browser/parts/statusbar), editor items in [`browser/parts/editor/editorStatus.ts`](./workbench/browser/parts/editor/editorStatus.ts) |
| Title Bar / Command Center | Cross-platform window chrome hosting global controls | [`browser/parts/titlebar`](./workbench/browser/parts/titlebar) |
| Notifications | Non-blocking progress/error reporting with an archive | [`browser/parts/notifications`](./workbench/browser/parts/notifications) |
| Breadcrumbs | Path + symbol trail for orientation and nearby navigation | [`browser/parts/editor/breadcrumbsControl.ts`](./workbench/browser/parts/editor/breadcrumbsControl.ts) |
| Command Palette / Quick Open | One searchable entry point for every command and file | [`contrib/quickaccess`](./workbench/contrib/quickaccess), "anything" provider in [`contrib/search/browser/anythingQuickAccess.ts`](./workbench/contrib/search/browser/anythingQuickAccess.ts), widget in [`platform/quickinput`](./platform/quickinput) |
| Zen Mode / layout control | Focus mode; every layout knob in one live picker | [`browser/workbench.zenMode.contribution.ts`](./workbench/browser/workbench.zenMode.contribution.ts), [`browser/actions/layoutActions.ts`](./workbench/browser/actions/layoutActions.ts) |

### Code editor

Editor features keep the developer in flow: information renders at the cursor, edits become commands. Core lives in [`editor/contrib`](./editor/contrib) (usable by Monaco), with workbench glue in [`workbench/contrib/codeEditor`](./workbench/contrib/codeEditor) and friends.

| Component | Why it exists | Source |
|-----------|---------------|--------|
| IntelliSense & parameter hints | API discovery without leaving the editor | [`editor/contrib/suggest`](./editor/contrib/suggest), [`editor/contrib/parameterHints`](./editor/contrib/parameterHints) |
| Hover | Types, docs, and diagnostics in place | [`editor/contrib/hover`](./editor/contrib/hover) |
| Go to Definition / References / Peek | Navigation is how unfamiliar code gets read | [`editor/contrib/gotoSymbol`](./editor/contrib/gotoSymbol), [`editor/contrib/peekView`](./editor/contrib/peekView); hierarchy views in [`contrib/callHierarchy`](./workbench/contrib/callHierarchy), [`contrib/typeHierarchy`](./workbench/contrib/typeHierarchy) |
| Rename / linked editing | Safe, atomic cross-workspace renames | [`editor/contrib/rename`](./editor/contrib/rename), [`editor/contrib/linkedEditing`](./editor/contrib/linkedEditing) |
| Code Actions | Diagnostics become one-keystroke repairs | [`editor/contrib/codeAction`](./editor/contrib/codeAction), [`contrib/codeActions`](./workbench/contrib/codeActions) |
| Formatting | Style delegated to the language's formatter | [`editor/contrib/format`](./editor/contrib/format), [`contrib/format`](./workbench/contrib/format) |
| Find & Replace (in file) | The highest-frequency editing operation | [`editor/contrib/find`](./editor/contrib/find) |
| Multi-cursor | Parallel edits beat serial edits | [`editor/contrib/multicursor`](./editor/contrib/multicursor) |
| Folding, minimap, sticky scroll | Structure over detail in large files | [`editor/contrib/folding`](./editor/contrib/folding), [`editor/browser/viewParts/minimap`](./editor/browser/viewParts/minimap), [`editor/contrib/stickyScroll`](./editor/contrib/stickyScroll) |
| CodeLens, inlay hints, color picker | The compiler's knowledge rendered inline | [`editor/contrib/codelens`](./editor/contrib/codelens), [`editor/contrib/inlayHints`](./editor/contrib/inlayHints), [`editor/contrib/colorPicker`](./editor/contrib/colorPicker) |
| Inline suggestions (ghost text) | Multi-token predictions with single-keystroke acceptance | [`editor/contrib/inlineCompletions`](./editor/contrib/inlineCompletions), glue in [`contrib/inlineCompletions`](./workbench/contrib/inlineCompletions) |
| Snippets & Emmet | Parameterized boilerplate expansion | [`editor/contrib/snippet`](./editor/contrib/snippet), [`contrib/snippets`](./workbench/contrib/snippets), [`contrib/emmet`](./workbench/contrib/emmet) + [`extensions/emmet`](../../extensions/emmet) |
| Bracket colorization, unicode highlighting | Structure made visible; invisible characters made visible | [`editor/common/model/bracketPairsTextModelPart`](./editor/common/model/bracketPairsTextModelPart), [`editor/contrib/unicodeHighlighter`](./editor/contrib/unicodeHighlighter) |

### Navigation, search, and workspace

| Component | Why it exists | Source |
|-----------|---------------|--------|
| Explorer & Open Editors | A spatial map of the project on disk | [`contrib/files`](./workbench/contrib/files) |
| Search view & Quick Search | ripgrep-backed workspace search/replace | [`contrib/search`](./workbench/contrib/search), service in [`services/search`](./workbench/services/search) |
| Search Editor | Search results as a durable, saveable document | [`contrib/searchEditor`](./workbench/contrib/searchEditor) |
| Outline | A live table of contents beside the code | [`contrib/outline`](./workbench/contrib/outline) |
| Timeline & Local History | Per-file chronology; a safety net independent of VCS | [`contrib/timeline`](./workbench/contrib/timeline), [`contrib/localHistory`](./workbench/contrib/localHistory) |
| Problems panel | All diagnostics, one filterable list | [`contrib/markers`](./workbench/contrib/markers), navigation in [`editor/contrib/gotoError`](./editor/contrib/gotoError) |
| Navigation history | Every jump is reversible | [`services/history`](./workbench/services/history) |
| Workspaces & Workspace Trust | Multi-root workspaces; a security gate on untrusted folders | [`contrib/workspaces`](./workbench/contrib/workspaces), [`contrib/workspace`](./workbench/contrib/workspace) |

### Run, debug, test, terminal

| Component | Why it exists | Source |
|-----------|---------------|--------|
| Run and Debug view, breakpoints, REPL | Launch under a debugger; inspect paused state | [`contrib/debug`](./workbench/contrib/debug) (views, `debugToolBar.ts`, `repl.ts`, breakpoint widgets) |
| Testing view | Framework-agnostic test discovery, runs, coverage | [`contrib/testing`](./workbench/contrib/testing) |
| Tasks | Builds/scripts with output mapped back to source via problem matchers | [`contrib/tasks`](./workbench/contrib/tasks) |
| Integrated terminal | Shell work that interoperates with open files | [`contrib/terminal`](./workbench/contrib/terminal); isolated features (find, quick fix, suggest, sticky scroll, shell integration…) each in [`contrib/terminalContrib/*`](./workbench/contrib/terminalContrib) |
| Output panel | Named, switchable log channels | [`contrib/output`](./workbench/contrib/output) |

### Source control

| Component | Why it exists | Source |
|-----------|---------------|--------|
| Source Control view & SCM Graph | Provider-agnostic staging, committing, history graph | [`contrib/scm`](./workbench/contrib/scm) (`scmHistoryViewPane.ts` for the graph) |
| Git integration | VCS as a built-in extension over the public SCM API | [`extensions/git`](../../extensions/git), [`extensions/git-base`](../../extensions/git-base), [`extensions/github`](../../extensions/github) |
| Diff editor & quick diff | Reviewing change is the core VCS task | [`editor/browser/widget/diffEditor`](./editor/browser/widget/diffEditor), gutter decorations in [`contrib/scm/browser/quickDiffDecorator.ts`](./workbench/contrib/scm/browser/quickDiffDecorator.ts) |
| Merge editor | Structured 3-way conflict resolution | [`contrib/mergeEditor`](./workbench/contrib/mergeEditor); inline conflict CodeLens in [`extensions/merge-conflict`](../../extensions/merge-conflict) |
| Multi-file diff | PR-style review of a whole changeset | [`contrib/multiDiffEditor`](./workbench/contrib/multiDiffEditor) |
| Comments | Threaded review anchored to editor ranges | [`contrib/comments`](./workbench/contrib/comments) |

### AI assistance

The chat UI, agent runtime, tool framework, and MCP support are core platform code; language models and completion providers come from provider extensions.

| Component | Why it exists | Source |
|-----------|---------------|--------|
| Chat view (Ask/Edit/Agent) & Quick Chat | Conversation to delegation in one surface | [`contrib/chat`](./workbench/contrib/chat) |
| Inline chat | AI edits at the exact code location | [`contrib/inlineChat`](./workbench/contrib/inlineChat) |
| MCP support | Standard, extension-free tool plug-in point for agents | [`contrib/mcp`](./workbench/contrib/mcp) |
| Agents Window (this fork) | Sessions-first workbench for agent workflows | [`src/vs/sessions`](./sessions) — see its [README](./sessions/README.md) and [LAYERS.md](./sessions/LAYERS.md) |

### Remote development

| Component | Why it exists | Source |
|-----------|---------------|--------|
| Remote indicator, Remote Explorer, Ports | Local UI, remote workspace — context always visible | [`contrib/remote`](./workbench/contrib/remote) (`remoteIndicator.ts`, `tunnelView.ts`) |
| Remote server | Runs extensions and serves the workbench remotely | [`src/vs/server`](./server), packages in [`remote/`](../../remote) |
| Remote Tunnels | Reach machines with no inbound network path | [`contrib/remoteTunnel`](./workbench/contrib/remoteTunnel), [`platform/tunnel`](./platform/tunnel), Rust CLI in [`cli/`](../../cli) |
| Continue Working On / Cloud Changes | Uncommitted work follows you across environments | [`contrib/editSessions`](./workbench/contrib/editSessions) |

### Personalization and extensibility

| Component | Why it exists | Source |
|-----------|---------------|--------|
| Extensions view & Marketplace | In-product discovery/install/management of extensions | [`contrib/extensions`](./workbench/contrib/extensions) |
| Settings editor & Keyboard Shortcuts editor | GUI over the configuration and keybinding models | [`contrib/preferences`](./workbench/contrib/preferences), services in [`services/preferences`](./workbench/services/preferences) |
| Themes | Color, file-icon, and product-icon theming with live preview | [`contrib/themes`](./workbench/contrib/themes), engine in [`services/themes`](./workbench/services/themes) |
| Profiles | Named bundles of settings/extensions/UI state | [`contrib/userDataProfile`](./workbench/contrib/userDataProfile), [`platform/userDataProfile`](./platform/userDataProfile) |
| Settings Sync | Your setup on every machine | [`contrib/userDataSync`](./workbench/contrib/userDataSync), engine in [`platform/userDataSync`](./platform/userDataSync) |
| Welcome & walkthroughs | First-run guidance; extension feature discovery | [`contrib/welcomeGettingStarted`](./workbench/contrib/welcomeGettingStarted) |
| Accessibility | Accessible View, help dialogs, signals | [`contrib/accessibility`](./workbench/contrib/accessibility), [`contrib/accessibilitySignals`](./workbench/contrib/accessibilitySignals) |

### Rich content surfaces

| Component | Why it exists | Source |
|-----------|---------------|--------|
| Notebook editor | Code + results + prose as a first-class editor | [`contrib/notebook`](./workbench/contrib/notebook), serialization in [`extensions/ipynb`](../../extensions/ipynb) |
| Webviews & custom editors | Sandboxed HTML canvases; visual editors per file type | [`contrib/webview`](./workbench/contrib/webview), [`contrib/webviewPanel`](./workbench/contrib/webviewPanel), [`contrib/customEditor`](./workbench/contrib/customEditor) |
| Markdown preview | Live, scroll-synced rendered docs | [`extensions/markdown-language-features`](../../extensions/markdown-language-features), glue in [`contrib/markdown`](./workbench/contrib/markdown) |
| Media preview & Simple Browser | Default viewers for images/audio/video; in-editor browser | [`extensions/media-preview`](../../extensions/media-preview), [`extensions/simple-browser`](../../extensions/simple-browser) |

## Built-in extensions

[`extensions/`](../../extensions) holds first-party extensions that ship inside the product. Unlike `workbench/contrib` features (core code using internal APIs), these use only the public `vscode` API — Git, Emmet, Markdown, the language grammars, default themes, `merge-conflict`, `references-view`, `simple-browser`, and the language-features extensions for TypeScript/HTML/CSS/JSON. They prove the extension API and keep the core VCS- and language-agnostic. See [`extensions/CONTRIBUTING.md`](../../extensions/CONTRIBUTING.md).

## Finding your way

- To find the code for a UI element, start from its **command ID** (visible in Keyboard Shortcuts editor → right-click → *Copy Command ID*) and search for it under `src/vs`.
- To find where a **view** lives, search for its view ID (for example `workbench.view.explorer`) — it leads to the `registerViewContainer`/`registerViews` call in the owning contribution.
- To trace a **setting**, search for its ID (for example `editor.minimap.enabled`) — configuration schemas are registered next to the feature.
- User-facing documentation for these components lives in the companion [vscode-docs](https://github.com/microsoft/vscode-docs) repository.
