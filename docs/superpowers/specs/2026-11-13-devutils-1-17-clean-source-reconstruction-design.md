# DevUtils 1.17.0 Clean-Source Reconstruction Design

Date: 2026-11-13
Status: Approved direction; implementation design
Behavioral oracle: `DevUtils.app` version 1.17.0, build 142
Source baseline: public/open source DevUtils 1.6.1 lineage in `DevUtilsApp/`

## 1. Objective

Reconstruct DevUtils 1.17.0 as a fully editable, buildable source tree whose observable behavior matches the shipped 1.17.0 application.

This is an observable-contract restoration, not an attempt to reproduce the lost source text line-for-line. The shipped application defines the behavioral contract. IDA analysis, runtime metadata, bundle resources, and controlled differential tests provide evidence for that contract.

The completed project must not launch, inject into, load executable code from, or otherwise depend on the original `DevUtils.app`. The original application remains an external oracle and test fixture only.

## 2. Chosen approach

The implementation will be a fully editable clean-source reconstruction.

All first-party build inputs must be maintainable source artifacts:

- Swift or Objective-C source for application logic;
- editable XIB and storyboard XML for AppKit interfaces;
- editable asset catalogs or individual source image assets;
- editable JavaScript, CSS, HTML, JSON, templates, and scripts;
- source-built third-party dependencies through Swift Package Manager, local packages, or reproducible source builds.

The final application target will not consume recovered compiled nibs, storyboards, asset catalogs, or stale compiler-specific first-party frameworks. Such binary artifacts may be inspected during reconstruction and used in parity tests, but not shipped as the implementation.

Text resources present in the shipped bundle may be recovered as editable source when provenance is clear. Their behavior must still be covered by tests rather than assumed correct merely because the text was recoverable.

## 3. Evidence and reconstruction rules

### 3.1 Source-of-truth hierarchy

When evidence conflicts, use this order:

1. Observable behavior of the shipped 1.17.0 application.
2. IDA decompilation, disassembly, cross-references, type metadata, selectors, constants, and string references from the shipped executable.
3. Compiled NIB/storyboard archive metadata, bundle structure, Info.plist declarations, entitlements, and resource contents.
4. The 1.6.1 source tree, where behavior remains present or supplies architectural continuity.
5. Public DevUtilsApp dependency forks and standalone tools as secondary implementation references.
6. New implementation choices required where the preceding evidence is incomplete.

### 3.2 Uncertainty handling

Every inferred behavior that is not directly proven by static or dynamic evidence must be recorded in the reconstruction notes and covered by an oracle comparison before being treated as equivalent.

Unknown values must not be silently invented. Where exact recovery is impossible, choose a deterministic and isolated fallback, document it, and expose it through a replaceable boundary.

### 3.3 Stable runtime identity

Classes referenced by archived UI, Objective-C selectors, notifications, defaults, URL routes, or serialization must preserve their recovered runtime-visible names and signatures. Internal implementation may differ, but the observable contract must remain stable.

## 4. Scope

### 4.1 Included

- Exact 47-tool registry, stable IDs, default order, grouping, aliases, descriptions, default actions, trial flags, icons, controller classes, implementation classes, and settings controllers.
- Hierarchical sidebar model, serialization, groups, separators, hidden tools, user-selected tools, frequency shortcuts, search, custom ordering, and alphabetical ordering.
- Main window, outline/sidebar, detail pane, toolbar, status-bar UI, preferences, manage-tools UI, onboarding, licensing/trial UI, document windows, setup/check-connection flows, shell-debug UI, and all tool-specific views and settings.
- Tool execution behavior for all 47 registered tools.
- `devutils://` routing, macOS Services handling, global hotkey, clipboard activation, status-bar activation, document pop-outs, launch-at-login, theme, font and editor preferences, dock/status-item behavior, update integration, and generated integration artifacts exposed by 1.17.0.
- UserDefaults keys, persisted formats, migration from the old source baseline, malformed-state behavior, and reset behavior.
- Editable reconstruction of first-party UI and assets.
- Reproducible source dependency graph.
- Differential verification against the shipped app.

### 4.2 Excluded from behavioral equivalence claims unless separately verified

- Live availability or historical behavior of third-party network services, licensing backends, update feeds, analytics endpoints, or package registries outside the repository.
- macOS 10.13 runtime certification on the current machine, because the installed Xcode/SDK cannot produce or validate that deployment target.
- Pixel-perfect behavior caused solely by undocumented AppKit implementation differences between macOS releases.
- Recovery of original comments, local names, file boundaries, or implementation style that have no observable effect.

## 5. Compatibility policy

The shipped bundle declares macOS 10.13 as its minimum. The reconstruction will preserve compatibility-sensitive abstraction boundaries and avoid gratuitous use of newer APIs, but the reproducible build and automated verification lane will initially target macOS 12.0 or newer because that is the minimum accepted by the installed toolchain.

The project must distinguish:

- **declared historical compatibility:** 10.13 in the shipped oracle;
- **source-intended compatibility:** guarded code paths designed not to preclude 10.13;
- **verified compatibility:** macOS 12.0 or newer under the current Xcode;
- **future certification lane:** an older compatible Xcode/SDK used to build and run the reconstructed source on 10.13.

No release note or test result may claim verified 10.13 support until that lane exists and passes.

## 6. Architecture

### 6.1 Project layout

The existing Xcode project history and product identities remain the base. Reconstruction should move code toward these explicit modules or directory boundaries without unnecessary unrelated refactoring:

- `ApplicationCore`: app state, defaults, activation, routing, notifications, build variants.
- `ToolCatalog`: tool metadata, exact registry, groups, aliases, availability, icons.
- `SidebarModel`: outline items, serialization, migration, filtering, sorting, drag/drop, usage ranking.
- `ToolRuntime`: common execution protocol, cancellation, output model, errors, runtime discovery.
- `ToolImplementations`: native, scripted, web-backed, and external-runtime tools.
- `AppIntegrations`: Services, URL scheme, hotkey, clipboard, status item, login item, Sparkle, generated integrations.
- `UI`: editable AppKit storyboards/XIBs, controllers, cells, window controllers, and reusable views.
- `Resources`: editable assets, scripts, HTML, CSS, JSON, templates, documentation, and fixtures.
- `ParityTests`: registry/model/tool fixtures, UI smoke tests, integration tests, and oracle tooling.

These may initially remain Xcode groups within one application target. Separate Swift packages or frameworks are justified only when they improve testability without changing runtime identity or making the build graph harder to reproduce.

### 6.2 Core model

Reconstruct the following recovered types and their contracts first:

- `ToolRaw`
- `Tool`
- `ToolGroup`
- `ToolWithUsageCount`
- `ToolShortcut`
- `OutlineItem`
- `OutlineItemList`
- `OutlineItemSerializer`
- `OutlineListStateProvider`

Recovered model requirements include:

- `ToolRaw` carries a stable tool ID.
- `Tool` carries name, tool description, default action description, tags, image identity, view-controller type, implementation type, settings-controller type, free-trial availability, aliases, and default group.
- `ToolGroup` carries a name, tools, and smart-group state.
- `OutlineItem` represents a tool, group, or separator and supports the latest outline operations.
- Tool items serialize as stable IDs.
- User-created groups serialize as `|<base64 UTF-8 group name>:<semicolon-separated tool IDs>|`.
- Persisted outline items are comma-joined under `sidebar-items`.
- Separators, smart groups, and shortcut rows are excluded from persisted group serialization where the binary excludes them.
- Malformed group payloads follow the recovered `Unnamed Group` fallback instead of corrupting the entire layout.
- Unknown tool IDs remain recoverable placeholders until the catalog can resolve them.

### 6.3 Catalog

The catalog is declarative and testable. It must reproduce the exact 47-tool order and metadata recovered from the binary. Registration must not be scattered across controllers.

Catalog validation runs at startup in debug/test builds and rejects:

- duplicate IDs;
- missing controllers or implementations;
- duplicate aliases that create ambiguous routes;
- missing icons or settings resources;
- invalid default groups;
- registry count or order drift from the checked-in golden snapshot.

### 6.4 Tool runtime

All tools conform to a common runtime contract independent of AppKit:

- input value and input metadata;
- typed options;
- primary output and optional secondary outputs;
- output copy/default action semantics;
- progress and cancellation where relevant;
- deterministic structured errors;
- optional runtime/setup requirements;
- fixtures suitable for oracle comparison.

Adapters are divided by execution model:

1. **Native tools:** Swift/Foundation implementations for UUID/ULID, Unix time, JWT, hashes, color conversion, number bases, string inspection, random generation, line operations, and similar deterministic logic.
2. **Scripted tools:** JavaScriptCore-backed formatter/converter engines using editable JavaScript and JSON resources.
3. **Web-backed tools:** WebKit/CodeMirror preview and editor behavior using editable HTML, CSS, and JavaScript resources.
4. **External-runtime tools:** PHP, OpenSSL, QuickType, and other command-backed flows with explicit executable discovery, setup state, argument construction, bounded execution, cancellation, stdout/stderr capture, and recovered error mapping.

External commands must be invoked without shell interpolation unless the shipped behavior explicitly requires a shell. Paths and user input must remain separate process arguments whenever possible.

### 6.5 UI reconstruction

Every latest UI archive is recreated as editable XIB/storyboard source.

The reconstruction process for each UI unit is:

1. Recover runtime class names, outlets, actions, selectors, bindings, and object relationships from IDA and NIBArchive metadata.
2. Observe the shipped screen in controlled states and record hierarchy, sizing, constraints, control types, labels, menus, keyboard equivalents, accessibility attributes, focus order, and state transitions.
3. Recreate the scene in Interface Builder XML or programmatic AppKit only where a programmatic view is materially easier to verify and maintain.
4. Add a smoke test that instantiates the scene and validates all expected outlets, actions, selectors, and top-level objects.
5. Add screenshot or geometry assertions only for stable layout properties; avoid brittle whole-screen pixel tests as the sole proof.
6. Compare interactions against the oracle and record deviations before moving the scene to parity-complete status.

The reconstruction order is dependency-driven:

1. shared controls and text editors;
2. main window and split/outline/detail navigation;
3. preferences and manage-tools flows;
4. status-bar and document windows;
5. onboarding, licensing, runtime setup, and connection/debug windows;
6. per-tool views and settings.

Compiled UI resources from the shipped app may remain in an external evidence directory during development, but must not be referenced by the application target or copied into the final product.

### 6.6 Application integrations

Restore integrations behind focused services:

- URL scheme parsing and tool activation;
- macOS Services pasteboard input;
- global hotkey registration;
- clipboard capture policy by activation source;
- status-item menu/popover and frequently used tools;
- pop-out document windows and selection synchronization;
- launch-at-login;
- theme, editor, font, line wrapping, and line-number preferences;
- dock and status-item visibility;
- Sparkle update behavior by build variant;
- Alfred/Raycast or other generated integration artifacts present in the oracle;
- connection checks, shell-debug flows, and runtime setup.

`AppDelegate` coordinates lifecycle only. Integration-specific state and side effects belong in injectable services so tests can observe requests without modifying the host system.

### 6.7 Licensing and trial behavior

Licensing and trial behavior are part of parity and must not be bypassed, weakened, or replaced with an always-enabled state.

Reconstruction includes:

- recovered product/trial state transitions;
- UI availability and tool gating;
- request construction and endpoint selection where statically recoverable;
- offline and error behavior;
- secure storage boundaries;
- App Store, Setapp, and direct-distribution variant behavior where evidence supports it.

Secrets, signing keys, or backend state that cannot be recovered are not fabricated. Network clients are isolated behind protocols so tests can reproduce all recovered responses and state transitions. Live-backend equivalence is claimed only when exercised against an authorized functioning service.

### 6.8 Persistence and migration

Preserve exact recovered keys and formats, including at least:

- `sidebar-items`
- `user-hidden-tools`
- `tools-top-frequently-used`
- `tools-sort-order`
- `custom-tools-ordered-names`
- existing activation, clipboard, theme, editor, hotkey, dock/status, and tool-specific settings keys

Migration must be:

- idempotent;
- non-destructive;
- testable from clean install, 1.6.1-era defaults, representative 1.17.0 defaults, and malformed states;
- reversible only where the shipped behavior is reversible;
- explicit about dropped or renamed keys.

Factory reset must match the oracle's scope and user confirmation behavior.

## 7. Dependency and build design

### 7.1 Dependency policy

- Replace stale compiler-specific Carthage frameworks with source-built dependencies.
- Prefer pinned Swift Package Manager revisions or local source packages.
- Use DevUtilsApp-maintained forks when they correspond to the shipped behavior and are available.
- Preserve behavior-compatible versions before attempting upgrades.
- Keep direct, App Store, and Setapp variants explicit rather than hiding them behind ad hoc build settings.
- Do not introduce a new dependency when a small, deterministic implementation is easier to audit and test.

The checked-in `Highlightr.framework` is incompatible with the current Swift compiler and must be rebuilt or replaced from source before the application target can establish a modern baseline.

### 7.2 Reproducibility

A clean checkout must be able to resolve dependencies, build, and run tests without the original app bundle. The oracle bundle may be supplied separately for differential tests, which must skip with a clear reason when it is absent.

The build records:

- Xcode and Swift version;
- resolved dependency revisions;
- deployment target used for verification;
- build variant;
- resource manifest and hashes;
- generated-file provenance.

## 8. Verification design

### 8.1 Static contract tests

- Exact 47-tool registry count, IDs, order, metadata, aliases, groups, trial flags, controller types, and implementation types.
- Bundle identifier, URL scheme, Services declarations, storyboard names, menu entries, Sparkle metadata, and required resources.
- Runtime-visible class and selector inventory for archived/UI-facing types.
- UserDefaults key manifest and serialization format.

### 8.2 Model golden tests

- Tool and group serialization/deserialization.
- Unicode and delimiter handling in group names.
- Malformed payloads and unknown IDs.
- Hidden tools, smart groups, separators, shortcuts, usage counts, ordering, and search.
- Migration from old and latest representative persisted states.

### 8.3 Tool fixture tests

Each of the 47 tools receives fixtures covering, as applicable:

- empty input;
- representative valid input;
- invalid input;
- Unicode and combining characters;
- binary or non-UTF-8 data where supported;
- large input;
- option combinations;
- missing external runtime;
- cancellation and timeout;
- stdout/stderr and error mapping.

Deterministic output is compared directly with the shipped app. Nondeterministic output is compared by structure and invariants using controlled clocks, random generators, or normalized fields.

### 8.4 UI verification

- Instantiate every storyboard, XIB, controller, cell, and reusable view.
- Assert outlet/action/selector completeness.
- Exercise focus order, keyboard equivalents, menus, validation, bindings, resizing, split positions, and state restoration.
- Capture stable reference screenshots for major scenes and compare geometry/semantics with tolerances appropriate to AppKit rendering differences.
- Maintain a manual checklist only for behavior that cannot be automated reliably.

### 8.5 Integration verification

- `devutils://` parsing and routing.
- Services input and error handling.
- hotkey and clipboard activation policies.
- status menu and frequency shortcuts.
- search, drag/drop, grouping, hiding, and ordering.
- document-window creation, synchronization, and teardown.
- launch-at-login requests without mutating the test host.
- preference propagation and theme changes.
- runtime discovery/setup and external process failures.
- licensing/trial matrices using deterministic mock responses.
- update request construction and offline behavior.

### 8.6 Differential oracle harness

The parity harness treats the shipped app as an independently launched black-box oracle. It must not inject reconstructed code into the oracle or rely on undocumented state that cannot be reproduced.

For each comparable case it records:

- input and options;
- app/build identity;
- output, error, and side effects;
- normalized UI state or screenshot where relevant;
- pass/fail and any accepted platform-specific difference.

A mismatch becomes a tracked reconstruction defect, not an undocumented exception.

## 9. Implementation sequence

### Phase 0: Baseline and evidence

- Make the old source build with the current toolchain by replacing incompatible binary dependencies.
- Check in an evidence manifest for binary identity, registry snapshot, class/selector inventories, bundle resources, and recovered constants.
- Establish test infrastructure and an optional oracle path.

### Phase 1: Core models and catalog

- Reconstruct the model types, exact serializer, defaults facade, migration, usage counting, and exact 47-tool catalog.
- Add golden tests before integrating UI.

### Phase 2: Shared runtime and shared UI

- Define the tool runtime and error model.
- Reconstruct shared editor, input/output, settings, table/cell, split-view, and window components as editable source.

### Phase 3: Navigation and app shell

- Reconstruct main window, hierarchical outline, search, ordering, drag/drop, hiding, manage-tools, preferences, status UI, and document windows.
- Restore activation paths and persistence.

### Phase 4: Deterministic native tools

- Implement and oracle-test native tools with no external runtime or complex web editor requirement.

### Phase 5: Scripted and web-backed tools

- Recover editable text resources and recreate JavaScriptCore/WebKit/CodeMirror integration.
- Reconstruct formatter, preview, conversion, and code-generation views.

### Phase 6: External-runtime tools and integrations

- Restore PHP/OpenSSL/QuickType and other runtime-backed flows.
- Restore setup/check-connection/shell-debug interfaces and robust process control.

### Phase 7: Onboarding, licensing, distribution variants

- Reconstruct onboarding, trial/license UI and state, update behavior, App Store/Setapp/direct variants, and generated integrations.

### Phase 8: Parity closure

- Run the complete static, model, tool, UI, integration, and manual oracle matrix.
- Resolve or explicitly document every remaining deviation.
- Remove any temporary dependency on recovered compiled first-party artifacts.

## 10. Acceptance criteria

The reconstruction is complete when all of the following are true:

1. A clean checkout builds from editable source with the documented current toolchain.
2. The application does not load or launch the original `DevUtils.app`.
3. No first-party compiled nib/storyboard or stale compiler-specific framework is required by the product target.
4. The exact 47-tool catalog and metadata snapshot pass.
5. Every registered tool has a working source implementation and parity fixtures.
6. Every latest scene is represented by editable XIB/storyboard or justified programmatic AppKit source and passes instantiation/wiring tests.
7. Persisted layout and settings semantics match the recovered contract, including migration and malformed-state handling.
8. URL scheme, Services, hotkey, clipboard, status menu, ordering/search/grouping, document windows, preferences, runtime setup, update, and licensing/trial flows pass their verification matrices to the extent the corresponding external services are available.
9. The shipped app and reconstructed build agree on deterministic tool outputs and observable state transitions for the checked-in oracle corpus.
10. Residual differences are enumerated with evidence and are not hidden behind broad equivalence claims.
11. The repository contains build, test, architecture, evidence, and provenance documentation sufficient for another engineer to continue without the lost source.

## 11. Principal risks and mitigations

### UI archive reconstruction risk

Compiled NIB archives do not preserve a direct editable representation of every Interface Builder property.

Mitigation: recover metadata, observe live hierarchy and behavior, rebuild incrementally, add wiring/geometry tests, and close each scene against the oracle before moving on.

### Compiler and framework drift

The old source depends on stale binary frameworks and historical Swift behavior.

Mitigation: source-build dependencies, pin revisions, isolate compatibility adapters, and establish a working baseline before feature reconstruction.

### Incomplete decompilation

Optimized Swift can obscure source-level intent and generic types.

Mitigation: triangulate decompilation with symbols, metadata, xrefs, strings, runtime observation, and old-source behavior. Test observable contracts rather than copying uncertain pseudocode mechanically.

### External-service drift

Historical licensing, update, or generation services may no longer return the same responses.

Mitigation: isolate clients, preserve recovered request contracts, test state machines with recorded or synthetic authorized responses, and distinguish live-service availability from application parity.

### Scope and sequencing risk

Rebuilding 142 missing types and 28 additional tools in one undifferentiated change would make regressions difficult to localize.

Mitigation: phase by architectural dependency, require tests and oracle evidence per batch, and keep the app runnable at each integration checkpoint.

## 12. Decision record

The user selected the fully editable clean-source reconstruction over the recommended hybrid-resource approach. Therefore exact compiled UI reuse is not an acceptable final implementation shortcut. The design prioritizes maintainability and complete source editability while retaining the shipped 1.17.0 binary as the behavioral oracle throughout reconstruction.
