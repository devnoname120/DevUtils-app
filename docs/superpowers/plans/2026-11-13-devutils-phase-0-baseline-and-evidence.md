# DevUtils Phase 0 Baseline and Evidence Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Produce a reproducible unsigned Debug build and test baseline on Xcode 27 while replacing the two compiler-incompatible Carthage dependencies with pinned source packages and capturing a deterministic contract manifest for the 1.17.0 oracle bundle.

**Architecture:** Keep the existing Xcode targets and source layout. Change only Highlightr and ShortcutRecorder from embedded Carthage frameworks to Swift Package products in this phase, add repository-local build/evidence scripts, and treat `DevUtils.app` as optional external evidence rather than a build input. The scripts fail loudly when project references or oracle identity drift.

**Tech Stack:** Xcode 27, Swift 6.4 compatibility mode, Swift Package Manager, zsh, Python 3, XCTest, macOS command-line tools (`plutil`, `otool`, `codesign`, `shasum`, `file`).

---

## File map

**Create**

- `DevUtilsApp/Scripts/Reconstruction/build-debug.sh` — canonical unsigned Debug build/test command.
- `DevUtilsApp/Scripts/Reconstruction/verify-source-dependencies.py` — rejects remaining linked/embedded Highlightr and ShortcutRecorder binaries and verifies pinned package references.
- `DevUtilsApp/Scripts/Reconstruction/capture-oracle-manifest.py` — generates deterministic metadata and resource counts from an optional 1.17.0 app bundle.
- `DevUtilsApp/Scripts/Reconstruction/verify-oracle-manifest.py` — validates the checked-in manifest and, when an oracle path is supplied, checks identity/hash drift.
- `DevUtilsApp/Reconstruction/README.md` — documents evidence provenance and commands.
- `DevUtilsApp/Reconstruction/OracleManifest.json` — generated contract manifest for the supplied 1.17.0 build.

**Modify**

- `DevUtilsApp/DevUtils.xcodeproj/project.pbxproj` — replace Highlightr/ShortcutRecorder framework links with source package products for direct, Setapp, and App Store targets.
- `DevUtilsApp/DevUtils.xcodeproj/project.xcworkspace/xcshareddata/swiftpm/Package.resolved` — record the resolved package revisions after migration.
- `.gitignore` — ignore reconstruction DerivedData and generated local evidence output, but not the checked-in manifest.

**Delete**

- `DevUtilsApp/Cartfile` — obsolete after both Carthage dependencies become pinned source packages.
- `DevUtilsApp/Cartfile.resolved` — obsolete after both Carthage dependencies become pinned source packages.

**Verify without modifying**

- `DevUtilsApp/TINOBHNYWE/Components/HighlightedTextView.swift`
- `DevUtilsApp/TINOBHNYWE/Components/JSONTextView.swift`
- `DevUtilsApp/TINOBHNYWE/Components/HotkeyRecorder.swift`
- `DevUtilsApp/TINOBHNYWE/Main/AppState.swift`
- `DevUtilsApp/TINOBHNYWE/Main/AppDelegate.swift`
- `DevUtilsApp/TINOBHNYWE/PreferencesWindow/PreferencesViewController.swift`

## Task 1: Add the canonical build wrapper

**Files:**
- Create: `DevUtilsApp/Scripts/Reconstruction/build-debug.sh`
- Modify: `.gitignore`

- [ ] **Step 1: Write the build wrapper**

Create `DevUtilsApp/Scripts/Reconstruction/build-debug.sh`:

```zsh
#!/bin/zsh
set -euo pipefail

script_dir=${0:A:h}
app_root=${script_dir:h:h}
repo_root=${app_root:h}
derived_data=${DEVUTILS_DERIVED_DATA:-"${repo_root}/build/reconstruction-derived-data"}
action=${1:-build}

case ${action} in
  build|test)
    ;;
  *)
    print -u2 "usage: ${0:t} [build|test]"
    exit 64
    ;;
esac

xcodebuild \
  -project "${app_root}/DevUtils.xcodeproj" \
  -scheme DevUtils \
  -configuration Debug \
  -derivedDataPath "${derived_data}" \
  -destination 'platform=macOS' \
  CODE_SIGNING_ALLOWED=NO \
  CODE_SIGNING_REQUIRED=NO \
  MACOSX_DEPLOYMENT_TARGET=12.0 \
  "${action}"
```

- [ ] **Step 2: Make it executable**

Run:

```bash
chmod +x DevUtilsApp/Scripts/Reconstruction/build-debug.sh
```

Expected: `test -x DevUtilsApp/Scripts/Reconstruction/build-debug.sh` exits 0.

- [ ] **Step 3: Extend `.gitignore`**

Append exactly:

```gitignore

# Reconstruction-local outputs
DevUtilsApp/Reconstruction/Local/
build/reconstruction-derived-data/
```

Do not ignore `DevUtilsApp/Reconstruction/OracleManifest.json`.

- [ ] **Step 4: Run the wrapper to preserve the pre-migration failure evidence**

Run:

```bash
DevUtilsApp/Scripts/Reconstruction/build-debug.sh build 2>&1 | tee /tmp/devutils-phase0-pre-migration.log
```

Expected before Task 2: nonzero exit with the app target failing at the stale Highlightr binary/module compatibility boundary. Package dependencies should resolve before that failure.

- [ ] **Step 5: Commit the wrapper**

```bash
git add .gitignore DevUtilsApp/Scripts/Reconstruction/build-debug.sh
git commit -m "build: add reproducible reconstruction build command"
```

## Task 2: Add a failing source-dependency verifier

**Files:**
- Create: `DevUtilsApp/Scripts/Reconstruction/verify-source-dependencies.py`

- [ ] **Step 1: Write the verifier before changing the project**

Create `DevUtilsApp/Scripts/Reconstruction/verify-source-dependencies.py`:

```python
#!/usr/bin/env python3
from __future__ import annotations

import argparse
from pathlib import Path

HIGH_LIGHTR_URL = "https://github.com/DevUtilsApp/Highlightr.git"
HIGH_LIGHTR_REVISION = "93199b9e434f04bda956a613af8f571933f9f037"
SHORTCUT_RECORDER_URL = "https://github.com/Kentzo/ShortcutRecorder.git"
SHORTCUT_RECORDER_VERSION = "3.4.0"

FORBIDDEN_LINKED_REFERENCES = (
    "Highlightr.framework in Frameworks",
    "Highlightr.framework in Embed Frameworks",
    "ShortcutRecorder.framework in Frameworks",
    "ShortcutRecorder.framework in Embed Frameworks",
    "$(PROJECT_DIR)/Carthage/Build/Mac",
)

REQUIRED_SNIPPETS = (
    HIGH_LIGHTR_URL,
    HIGH_LIGHTR_REVISION,
    SHORTCUT_RECORDER_URL,
    f"version = {SHORTCUT_RECORDER_VERSION};",
    "productName = Highlightr;",
    "productName = ShortcutRecorder;",
)


def verify(project_file: Path) -> list[str]:
    contents = project_file.read_text(encoding="utf-8")
    failures = [
        f"linked binary reference remains: {reference}"
        for reference in FORBIDDEN_LINKED_REFERENCES
        if reference in contents
    ]
    failures.extend(
        f"required source-package contract missing: {snippet}"
        for snippet in REQUIRED_SNIPPETS
        if snippet not in contents
    )
    app_root = project_file.parents[1]
    failures.extend(
        f"obsolete Carthage manifest remains: {path.name}"
        for path in (app_root / "Cartfile", app_root / "Cartfile.resolved")
        if path.exists()
    )
    return failures


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "project_file",
        nargs="?",
        type=Path,
        default=Path(__file__).resolve().parents[2]
        / "DevUtils.xcodeproj"
        / "project.pbxproj",
    )
    args = parser.parse_args()

    failures = verify(args.project_file)
    if failures:
        for failure in failures:
            print(f"ERROR: {failure}")
        return 1

    print("source dependency contract verified")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

- [ ] **Step 2: Make it executable and run it**

Run:

```bash
chmod +x DevUtilsApp/Scripts/Reconstruction/verify-source-dependencies.py
DevUtilsApp/Scripts/Reconstruction/verify-source-dependencies.py
```

Expected: FAIL. It must report all four categories of linked binary reference and missing package-contract snippets.

- [ ] **Step 3: Commit the failing verifier**

```bash
git add DevUtilsApp/Scripts/Reconstruction/verify-source-dependencies.py
git commit -m "test: define source dependency contract"
```

## Task 3: Replace Highlightr and ShortcutRecorder binaries with source packages

**Files:**
- Modify: `DevUtilsApp/DevUtils.xcodeproj/project.pbxproj`
- Modify: `DevUtilsApp/DevUtils.xcodeproj/project.xcworkspace/xcshareddata/swiftpm/Package.resolved`

Use these package contracts:

```text
Highlightr
  URL: https://github.com/DevUtilsApp/Highlightr.git
  requirement: exact revision 93199b9e434f04bda956a613af8f571933f9f037
  product: Highlightr

ShortcutRecorder
  URL: https://github.com/Kentzo/ShortcutRecorder.git
  requirement: exact version 3.4.0
  product: ShortcutRecorder
```

The Highlightr fork revision is deliberately after tag 2.1.1 because it adds source-package resource loading while retaining the 2.1.1 lineage. ShortcutRecorder 3.4.0 is the version already recorded in `Cartfile.resolved` and has a package manifest.

- [ ] **Step 1: Add six package-product build-file objects**

In `PBXBuildFile`, add one Highlightr and one ShortcutRecorder product per app target:

```text
D11700010000000000000001 /* Highlightr in Frameworks */ = {isa = PBXBuildFile; productRef = D11700100000000000000001 /* Highlightr */; };
D11700020000000000000001 /* ShortcutRecorder in Frameworks */ = {isa = PBXBuildFile; productRef = D11700110000000000000001 /* ShortcutRecorder */; };
D11700030000000000000001 /* Highlightr in Frameworks */ = {isa = PBXBuildFile; productRef = D11700120000000000000001 /* Highlightr */; };
D11700040000000000000001 /* ShortcutRecorder in Frameworks */ = {isa = PBXBuildFile; productRef = D11700130000000000000001 /* ShortcutRecorder */; };
D11700050000000000000001 /* Highlightr in Frameworks */ = {isa = PBXBuildFile; productRef = D11700140000000000000001 /* Highlightr */; };
D11700060000000000000001 /* ShortcutRecorder in Frameworks */ = {isa = PBXBuildFile; productRef = D11700150000000000000001 /* ShortcutRecorder */; };
```

Mapping:

- `D1170001`/`D1170002`: DevUtils direct target.
- `D1170003`/`D1170004`: DevUtils Setapp target.
- `D1170005`/`D1170006`: DevUtils AppStore target.

- [ ] **Step 2: Replace linked framework build-file IDs in the three Frameworks phases**

Replace only these entries:

```text
A5114FDA24821504001A8DB5 -> D11700010000000000000001
A513045124758D4B0023D0CF -> D11700020000000000000001
A5D7360C2555802A00F3C28D -> D11700030000000000000001
A5D736072555802A00F3C28D -> D11700040000000000000001
A5D7369625565D1E00F3C28D -> D11700050000000000000001
A5D7369125565D1E00F3C28D -> D11700060000000000000001
```

Delete the six superseded `PBXBuildFile` objects rather than leaving dead references.

- [ ] **Step 3: Remove the six embed-framework build files**

Delete these entries from the three Embed Frameworks phases and delete their `PBXBuildFile` objects:

```text
A5114FDB24821504001A8DB5
A513045224758D4B0023D0CF
A5D736312555802A00F3C28D
A5D736332555802A00F3C28D
A5D736BB25565D1E00F3C28D
A5D736BD25565D1E00F3C28D
```

Swift package products are linked by Xcode and must not be copied by the old Carthage embed phases.

- [ ] **Step 4: Remove stale file references, search paths, and Carthage manifests**

Delete these `PBXFileReference` objects and their references from the Frameworks group:

```text
A5114FD7248214F6001A8DB5 /* Highlightr.framework */
A5130415247422AB0023D0CF /* ShortcutRecorder.framework */
```

Do not remove `DiffMatchPatch.framework` or `Sparkle.framework` in Phase 0; they are separate source-migration work and are not the current Swift compiler blocker.

Remove every `"$(PROJECT_DIR)/Carthage/Build/Mac",` entry from the direct, Setapp, and App Store build configurations. Delete `DevUtilsApp/Cartfile` and `DevUtilsApp/Cartfile.resolved`; their exact historical versions are preserved in Git history and in this plan's package contracts.

- [ ] **Step 5: Add package references to the project**

Add to `PBXProject.packageReferences`:

```text
D11700200000000000000001 /* XCRemoteSwiftPackageReference "Highlightr" */,
D11700210000000000000001 /* XCRemoteSwiftPackageReference "ShortcutRecorder" */,
```

Add to `XCRemoteSwiftPackageReference`:

```text
D11700200000000000000001 /* XCRemoteSwiftPackageReference "Highlightr" */ = {
  isa = XCRemoteSwiftPackageReference;
  repositoryURL = "https://github.com/DevUtilsApp/Highlightr.git";
  requirement = {
    kind = revision;
    revision = 93199b9e434f04bda956a613af8f571933f9f037;
  };
};
D11700210000000000000001 /* XCRemoteSwiftPackageReference "ShortcutRecorder" */ = {
  isa = XCRemoteSwiftPackageReference;
  repositoryURL = "https://github.com/Kentzo/ShortcutRecorder.git";
  requirement = {
    kind = exactVersion;
    version = 3.4.0;
  };
};
```

- [ ] **Step 6: Add product dependencies**

Add to `XCSwiftPackageProductDependency`:

```text
D11700100000000000000001 /* Highlightr */ = {
  isa = XCSwiftPackageProductDependency;
  package = D11700200000000000000001 /* XCRemoteSwiftPackageReference "Highlightr" */;
  productName = Highlightr;
};
D11700110000000000000001 /* ShortcutRecorder */ = {
  isa = XCSwiftPackageProductDependency;
  package = D11700210000000000000001 /* XCRemoteSwiftPackageReference "ShortcutRecorder" */;
  productName = ShortcutRecorder;
};
D11700120000000000000001 /* Highlightr */ = {
  isa = XCSwiftPackageProductDependency;
  package = D11700200000000000000001 /* XCRemoteSwiftPackageReference "Highlightr" */;
  productName = Highlightr;
};
D11700130000000000000001 /* ShortcutRecorder */ = {
  isa = XCSwiftPackageProductDependency;
  package = D11700210000000000000001 /* XCRemoteSwiftPackageReference "ShortcutRecorder" */;
  productName = ShortcutRecorder;
};
D11700140000000000000001 /* Highlightr */ = {
  isa = XCSwiftPackageProductDependency;
  package = D11700200000000000000001 /* XCRemoteSwiftPackageReference "Highlightr" */;
  productName = Highlightr;
};
D11700150000000000000001 /* ShortcutRecorder */ = {
  isa = XCSwiftPackageProductDependency;
  package = D11700210000000000000001 /* XCRemoteSwiftPackageReference "ShortcutRecorder" */;
  productName = ShortcutRecorder;
};
```

Append direct products `D1170010`/`D1170011`, Setapp products `D1170012`/`D1170013`, and AppStore products `D1170014`/`D1170015` to the corresponding `packageProductDependencies` arrays.

- [ ] **Step 7: Resolve packages**

Run:

```bash
xcodebuild \
  -resolvePackageDependencies \
  -project DevUtilsApp/DevUtils.xcodeproj \
  -scheme DevUtils
```

Expected: exit 0; resolved output includes Highlightr at revision `93199b9e434f04bda956a613af8f571933f9f037` and ShortcutRecorder 3.4.0.

- [ ] **Step 8: Run the dependency verifier**

Run:

```bash
DevUtilsApp/Scripts/Reconstruction/verify-source-dependencies.py
```

Expected: `source dependency contract verified`.

- [ ] **Step 9: Build the direct target**

Run:

```bash
DevUtilsApp/Scripts/Reconstruction/build-debug.sh build
```

Expected: either `** BUILD SUCCEEDED **` or a new source-level/current-toolchain error unrelated to loading Highlightr/ShortcutRecorder prebuilt Swift modules. If a new error occurs, stop Task 3 and use systematic debugging before changing behavior.

- [ ] **Step 10: Commit the dependency migration**

```bash
git add \
  DevUtilsApp/Cartfile \
  DevUtilsApp/Cartfile.resolved \
  DevUtilsApp/DevUtils.xcodeproj/project.pbxproj \
  DevUtilsApp/DevUtils.xcodeproj/project.xcworkspace/xcshareddata/swiftpm/Package.resolved
git commit -m "build: source Highlightr and ShortcutRecorder with SwiftPM"
```

## Task 4: Capture the oracle bundle contract

**Files:**
- Create: `DevUtilsApp/Scripts/Reconstruction/capture-oracle-manifest.py`
- Create: `DevUtilsApp/Reconstruction/OracleManifest.json`
- Create: `DevUtilsApp/Reconstruction/README.md`

- [ ] **Step 1: Write the manifest generator**

Create `DevUtilsApp/Scripts/Reconstruction/capture-oracle-manifest.py`:

```python
#!/usr/bin/env python3
from __future__ import annotations

import argparse
import hashlib
import json
import plistlib
import subprocess
from collections import Counter
from pathlib import Path
from typing import Any


def sha256(path: Path) -> str:
    digest = hashlib.sha256()
    with path.open("rb") as handle:
        for chunk in iter(lambda: handle.read(1024 * 1024), b""):
            digest.update(chunk)
    return digest.hexdigest()


def run(*arguments: str) -> str:
    return subprocess.run(
        arguments,
        check=True,
        capture_output=True,
        text=True,
    ).stdout.strip()


def resource_summary(resources: Path) -> dict[str, Any]:
    files = sorted(path for path in resources.rglob("*") if path.is_file())
    suffixes = Counter(path.suffix.lower() or "<none>" for path in files)
    top_level = Counter(path.relative_to(resources).parts[0] for path in files)
    return {
        "fileCount": len(files),
        "suffixCounts": dict(sorted(suffixes.items())),
        "topLevelCounts": dict(sorted(top_level.items())),
    }


def manifest(app: Path) -> dict[str, Any]:
    contents = app / "Contents"
    info_path = contents / "Info.plist"
    executable_path = contents / "MacOS" / "DevUtils"
    resources_path = contents / "Resources"

    with info_path.open("rb") as handle:
        info = plistlib.load(handle)

    return {
        "schemaVersion": 1,
        "bundle": {
            "identifier": info["CFBundleIdentifier"],
            "shortVersion": info["CFBundleShortVersionString"],
            "buildVersion": info["CFBundleVersion"],
            "minimumSystemVersion": info.get("LSMinimumSystemVersion"),
            "principalClass": info.get("NSPrincipalClass"),
            "mainStoryboard": info.get("NSMainStoryboardFile"),
            "urlTypes": info.get("CFBundleURLTypes", []),
            "services": info.get("NSServices", []),
            "sparkleFeedURL": info.get("SUFeedURL"),
        },
        "executable": {
            "relativePath": "Contents/MacOS/DevUtils",
            "sha256": sha256(executable_path),
            "size": executable_path.stat().st_size,
            "fileDescription": run("file", "-b", str(executable_path)),
            "linkedLibraries": run("otool", "-L", str(executable_path)).splitlines()[1:],
        },
        "resources": resource_summary(resources_path),
    }


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("app", type=Path)
    parser.add_argument("output", type=Path)
    args = parser.parse_args()

    result = manifest(args.app.resolve())
    args.output.parent.mkdir(parents=True, exist_ok=True)
    args.output.write_text(
        json.dumps(result, indent=2, sort_keys=True) + "\n",
        encoding="utf-8",
    )
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

- [ ] **Step 2: Generate the checked-in manifest**

Run:

```bash
chmod +x DevUtilsApp/Scripts/Reconstruction/capture-oracle-manifest.py
DevUtilsApp/Scripts/Reconstruction/capture-oracle-manifest.py \
  DevUtils.app \
  DevUtilsApp/Reconstruction/OracleManifest.json
```

Expected manifest identity:

```json
{
  "bundle": {
    "identifier": "tonyapp.devutils",
    "shortVersion": "1.17.0",
    "buildVersion": "142"
  }
}
```

The actual file contains the complete sorted contract, not this abbreviated excerpt.

- [ ] **Step 3: Document provenance and usage**

Create `DevUtilsApp/Reconstruction/README.md`:

```markdown
# Reconstruction evidence

`OracleManifest.json` records deterministic identity and bundle-contract data from
DevUtils 1.17.0 build 142. The original app bundle is not a build dependency and
is intentionally not committed by this work.

Regenerate from an authorized local oracle bundle:

```sh
DEVUTILS_ORACLE_APP=/absolute/location/DevUtils.app
DevUtilsApp/Scripts/Reconstruction/capture-oracle-manifest.py \
  "${DEVUTILS_ORACLE_APP}" \
  DevUtilsApp/Reconstruction/OracleManifest.json
```

The executable hash and bundle identity must change only when the behavioral
oracle changes deliberately. Resource counts are contract evidence, not a grant
to ship compiled recovered first-party resources.
```

- [ ] **Step 4: Verify deterministic regeneration**

Run:

```bash
cp DevUtilsApp/Reconstruction/OracleManifest.json /tmp/oracle-manifest.before.json
DevUtilsApp/Scripts/Reconstruction/capture-oracle-manifest.py \
  DevUtils.app \
  /tmp/oracle-manifest.after.json
cmp /tmp/oracle-manifest.before.json /tmp/oracle-manifest.after.json
```

Expected: `cmp` exits 0.

- [ ] **Step 5: Commit oracle evidence**

```bash
git add \
  DevUtilsApp/Scripts/Reconstruction/capture-oracle-manifest.py \
  DevUtilsApp/Reconstruction/OracleManifest.json \
  DevUtilsApp/Reconstruction/README.md
git commit -m "docs: capture DevUtils 1.17 oracle contract"
```

## Task 5: Add manifest verification

**Files:**
- Create: `DevUtilsApp/Scripts/Reconstruction/verify-oracle-manifest.py`

- [ ] **Step 1: Write the verifier**

Create `DevUtilsApp/Scripts/Reconstruction/verify-oracle-manifest.py`:

```python
#!/usr/bin/env python3
from __future__ import annotations

import argparse
import hashlib
import json
from pathlib import Path
from typing import Any

EXPECTED_IDENTITY = {
    "identifier": "tonyapp.devutils",
    "shortVersion": "1.17.0",
    "buildVersion": "142",
}
EXPECTED_EXECUTABLE_SHA256 = (
    "0cd93e970da57dfa3102d1cc209e433a83d9f9842a9db1b1a696d6d7d63589ef"
)


def sha256(path: Path) -> str:
    digest = hashlib.sha256()
    with path.open("rb") as handle:
        for chunk in iter(lambda: handle.read(1024 * 1024), b""):
            digest.update(chunk)
    return digest.hexdigest()


def load(path: Path) -> dict[str, Any]:
    return json.loads(path.read_text(encoding="utf-8"))


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "--manifest",
        type=Path,
        default=Path(__file__).resolve().parents[2]
        / "Reconstruction"
        / "OracleManifest.json",
    )
    parser.add_argument("--oracle", type=Path)
    args = parser.parse_args()

    data = load(args.manifest)
    failures: list[str] = []

    if data.get("schemaVersion") != 1:
        failures.append("schemaVersion must be 1")

    bundle = data.get("bundle", {})
    for key, expected in EXPECTED_IDENTITY.items():
        if bundle.get(key) != expected:
            failures.append(f"bundle.{key}: expected {expected!r}, got {bundle.get(key)!r}")

    recorded_hash = data.get("executable", {}).get("sha256")
    if recorded_hash != EXPECTED_EXECUTABLE_SHA256:
        failures.append(
            f"executable.sha256: expected {EXPECTED_EXECUTABLE_SHA256}, got {recorded_hash}"
        )

    if args.oracle:
        executable = args.oracle / "Contents" / "MacOS" / "DevUtils"
        actual_hash = sha256(executable)
        if actual_hash != recorded_hash:
            failures.append(
                f"oracle executable hash: expected {recorded_hash}, got {actual_hash}"
            )

    if failures:
        for failure in failures:
            print(f"ERROR: {failure}")
        return 1

    print("oracle manifest verified")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

- [ ] **Step 2: Make it executable and verify both modes**

Run:

```bash
chmod +x DevUtilsApp/Scripts/Reconstruction/verify-oracle-manifest.py
DevUtilsApp/Scripts/Reconstruction/verify-oracle-manifest.py
DevUtilsApp/Scripts/Reconstruction/verify-oracle-manifest.py --oracle DevUtils.app
```

Expected for both: `oracle manifest verified`.

- [ ] **Step 3: Verify the negative path**

Run:

```bash
python3 - <<'PY'
import json
from pathlib import Path
source = Path('DevUtilsApp/Reconstruction/OracleManifest.json')
data = json.loads(source.read_text())
data['bundle']['buildVersion'] = '141'
Path('/tmp/devutils-invalid-manifest.json').write_text(json.dumps(data))
PY
DevUtilsApp/Scripts/Reconstruction/verify-oracle-manifest.py \
  --manifest /tmp/devutils-invalid-manifest.json
```

Expected: nonzero exit containing `bundle.buildVersion: expected '142', got '141'`.

- [ ] **Step 4: Commit the verifier**

```bash
git add DevUtilsApp/Scripts/Reconstruction/verify-oracle-manifest.py
git commit -m "test: verify DevUtils oracle identity"
```

## Task 6: Close the current-toolchain baseline

**Files:**
- Modify only files required by concrete compiler errors found in this task.

- [ ] **Step 1: Verify project syntax and dependencies**

Run:

```bash
plutil -lint DevUtilsApp/DevUtils.xcodeproj/project.pbxproj
DevUtilsApp/Scripts/Reconstruction/verify-source-dependencies.py
DevUtilsApp/Scripts/Reconstruction/verify-oracle-manifest.py --oracle DevUtils.app
```

Expected: all exit 0.

- [ ] **Step 2: Build**

Run:

```bash
DevUtilsApp/Scripts/Reconstruction/build-debug.sh build
```

Expected: `** BUILD SUCCEEDED **`.

If source compiler errors remain, fix one root cause at a time using `superpowers:systematic-debugging`. Do not suppress errors by removing tool behavior. Every compatibility change must be the smallest behavior-preserving adaptation and receive a focused test where observable behavior could change.

- [ ] **Step 3: Run tests**

Run:

```bash
DevUtilsApp/Scripts/Reconstruction/build-debug.sh test
```

Expected: `** TEST SUCCEEDED **`; the existing UUID tests pass.

- [ ] **Step 4: Verify package resolution and product contents**

Run:

```bash
xcodebuild \
  -project DevUtilsApp/DevUtils.xcodeproj \
  -scheme DevUtils \
  -showBuildSettings \
  | grep -E 'PRODUCT_BUNDLE_IDENTIFIER|MACOSX_DEPLOYMENT_TARGET|SWIFT_VERSION'

find build/reconstruction-derived-data/Build/Products/Debug/DevUtils.app/Contents/Frameworks \
  -maxdepth 1 -type d -print 2>/dev/null | sort
```

Expected:

- product bundle identifier remains `tonyapp.devutils`;
- effective verification deployment target is 12.0;
- no `Highlightr.framework` or `ShortcutRecorder.framework` is copied from `DevUtilsApp/Carthage/Build/Mac`;
- source package products are resolved in DerivedData.

- [ ] **Step 5: Run repository checks**

```bash
git diff --check
git status --short
```

Expected: no whitespace errors. The four pre-existing deleted documentation files and untracked `DevUtils.app/` remain untouched and unstaged.

- [ ] **Step 6: Handle unexpected source compiler failures without speculative edits**

If Step 2 fails after the package migration, preserve the complete log and stop Phase 0 execution at that failure:

```bash
DevUtilsApp/Scripts/Reconstruction/build-debug.sh build \
  2>&1 | tee /tmp/devutils-phase0-current-toolchain-failure.log
```

Then invoke `superpowers:systematic-debugging`, identify one root cause, and amend this plan with the exact affected paths, failing test, minimal compatibility change, verification command, and commit command before editing source. If Step 2 succeeds, this step is complete with no source change.

## Task 7: Phase verification and handoff

**Files:**
- Modify: `docs/superpowers/plans/2026-11-13-devutils-1-17-reconstruction-roadmap.md`
- Modify: `docs/superpowers/plans/2026-11-13-devutils-phase-0-baseline-and-evidence.md`

- [ ] **Step 1: Run the complete Phase 0 gate**

```bash
DevUtilsApp/Scripts/Reconstruction/verify-source-dependencies.py
DevUtilsApp/Scripts/Reconstruction/verify-oracle-manifest.py --oracle DevUtils.app
DevUtilsApp/Scripts/Reconstruction/build-debug.sh build
DevUtilsApp/Scripts/Reconstruction/build-debug.sh test
git diff --check
```

Expected: every command exits 0.

- [ ] **Step 2: Mark completed checkboxes accurately**

Change only checkboxes whose command and expected evidence actually passed. Do not mark the phase complete based on intent.

- [ ] **Step 3: Commit the plan state**

```bash
git add \
  docs/superpowers/plans/2026-11-13-devutils-1-17-reconstruction-roadmap.md \
  docs/superpowers/plans/2026-11-13-devutils-phase-0-baseline-and-evidence.md
git commit -m "docs: record DevUtils reconstruction phase zero"
```

- [ ] **Step 4: Start Phase 1 planning**

Read the approved design and the Phase 0 evidence, then create `docs/superpowers/plans/2026-11-13-devutils-phase-1-models-catalog-persistence.md` using the writing-plans skill. Phase 1 must define the exact recovered type signatures, serializer golden cases, defaults migration fixtures, and 47-tool metadata snapshot before any model implementation begins.
