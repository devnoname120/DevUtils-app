# DevUtils 1.17.0 Reconstruction Roadmap

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reconstruct DevUtils 1.17.0 as fully editable source while preserving its 47-tool catalog, persisted behavior, AppKit interfaces, integrations, and licensing/trial semantics.

**Architecture:** Implement the approved design as independently verifiable phases. Each phase starts from a buildable application, adds one architectural layer, and closes with oracle-backed tests before the next phase begins. The shipped application remains an external behavioral oracle and never becomes a product dependency.

**Tech Stack:** Swift 6.4 compatibility mode, AppKit, XIB/storyboard XML, Swift Package Manager, XCTest, JavaScriptCore, WebKit, Foundation `Process`, Python 3 evidence tooling, IDA MCP, Xcode 27.

---

## Plan set

- [ ] **Phase 0 — Baseline and evidence**

  Execute `docs/superpowers/plans/2026-11-13-devutils-phase-0-baseline-and-evidence.md`.

  Deliverable: the old source builds and tests on the current toolchain without the incompatible Highlightr or ShortcutRecorder binaries; the oracle identity and bundle contract are captured reproducibly.

- [ ] **Phase 1 — Core models, persistence, and exact catalog**

  Create and execute `docs/superpowers/plans/2026-11-13-devutils-phase-1-models-catalog-persistence.md` after Phase 0 passes.

  Deliverable: `ToolRaw`, `Tool`, `ToolGroup`, `ToolShortcut`, `ToolWithUsageCount`, `OutlineItem`, `OutlineItemList`, serializer/state provider, exact defaults keys, migration, and a golden 47-tool catalog snapshot.

- [ ] **Phase 2 — Shared runtime and editable shared UI**

  Create and execute `docs/superpowers/plans/2026-11-13-devutils-phase-2-runtime-shared-ui.md` after Phase 1 passes.

  Deliverable: AppKit-independent tool execution protocol, output/error/cancellation model, shared editor/input/output/settings controls, editable XIBs, and UI instantiation tests.

- [ ] **Phase 3 — Application shell and navigation**

  Create and execute `docs/superpowers/plans/2026-11-13-devutils-phase-3-app-shell-navigation.md` after Phase 2 passes.

  Deliverable: editable main-window UI, hierarchical outline, search, grouping, drag/drop, hidden/custom/alphabetical ordering, manage-tools/preferences, status UI, document windows, and activation synchronization.

- [ ] **Phase 4 — Deterministic native tools**

  Create and execute `docs/superpowers/plans/2026-11-13-devutils-phase-4-native-tools.md` after Phase 3 passes.

  Deliverable: source implementations and differential fixtures for deterministic tools that need neither WebKit nor external executables.

- [ ] **Phase 5 — Scripted and web-backed tools**

  Create and execute `docs/superpowers/plans/2026-11-13-devutils-phase-5-scripted-web-tools.md` after Phase 4 passes.

  Deliverable: editable JavaScript/HTML/CSS resources, JavaScriptCore and WebKit adapters, CodeMirror-equivalent editable UI, formatter/converter/preview/code-generation parity.

- [ ] **Phase 6 — External runtime tools and setup flows**

  Create and execute `docs/superpowers/plans/2026-11-13-devutils-phase-6-external-runtimes.md` after Phase 5 passes.

  Deliverable: PHP/OpenSSL/QuickType and other process-backed tools, executable discovery, bounded process execution, cancellation, setup/check-connection UI, shell-debug UI, and deterministic error mapping.

- [ ] **Phase 7 — Onboarding, licensing, distribution, and integrations**

  Create and execute `docs/superpowers/plans/2026-11-13-devutils-phase-7-product-integrations.md` after Phase 6 passes.

  Deliverable: onboarding, trial/license state and UI without bypasses, direct/App Store/Setapp behavior, Sparkle, `devutils://`, Services, hotkey, clipboard, status item, login item, generated integrations, and offline/error matrices.

- [ ] **Phase 8 — Parity closure and release evidence**

  Create and execute `docs/superpowers/plans/2026-11-13-devutils-phase-8-parity-closure.md` after Phase 7 passes.

  Deliverable: complete registry/model/tool/UI/integration matrix, residual-deviation ledger, clean-checkout build proof, bundle-contract proof, and confirmation that no original-app or compiled first-party resource dependency remains.

## Global gates

Every phase must satisfy all of these gates before the next phase starts:

- [ ] `git diff --check` reports no whitespace errors.
- [ ] The unsigned Debug build succeeds with the documented command.
- [ ] All tests available at that phase pass.
- [ ] New behavior has a failing test or oracle fixture before implementation.
- [ ] No pre-existing user deletion or the untracked oracle bundle is staged accidentally.
- [ ] The phase commit contains only the intended source, tests, documentation, and project-file changes.
- [ ] The rollout journal records evidence, residual uncertainty, and the next phase.
