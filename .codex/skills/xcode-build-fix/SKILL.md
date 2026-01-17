---
name: xcode-build-fix
description: Build Xcode projects or workspaces from the CLI, diagnose compile-time/build-time failures, and iterate fixes until the app builds. Use when asked to run xcodebuild, compile iOS/macOS apps, resolve compiler/linker/signing issues, or unblock build errors in .xcodeproj/.xcworkspace repositories.
---

# Xcode Build Fix

## Overview

Build the current Xcode app from the CLI, collect compile-time/build-time errors, apply minimal fixes, and repeat until the build succeeds.

## Workflow

### 1) Locate the build entry point

- Prefer a `.xcworkspace` if present; otherwise use the `.xcodeproj`.
- List schemes to choose a build target.

```bash
ls *.xcworkspace *.xcodeproj 2>/dev/null
xcodebuild -list -workspace App.xcworkspace
# or
xcodebuild -list -project App.xcodeproj
```

### 2) Build with deterministic settings

- Use a simulator SDK to avoid device signing and provisioning.
- Set an explicit derived data path for easier cleanup.

```bash
DERIVED_DATA=.build/DerivedData
xcodebuild -workspace App.xcworkspace -scheme App \
  -configuration Debug -sdk iphonesimulator \
  -derivedDataPath "$DERIVED_DATA" build
```

If there is no workspace:

```bash
DERIVED_DATA=.build/DerivedData
xcodebuild -project App.xcodeproj -scheme App \
  -configuration Debug -sdk iphonesimulator \
  -derivedDataPath "$DERIVED_DATA" build
```

### 3) Capture and classify failures

Focus only on build-time signals:

- **Swift/Obj-C compile errors**: file/line diagnostics; fix code or imports.
- **Module not found**: missing package/module map; check build settings and package resolution.
- **Linker errors**: missing framework/library, duplicate symbols, or wrong target membership.
- **Package resolution errors**: run `xcodebuild -resolvePackageDependencies`.
- **Code signing errors**: keep simulator builds; if necessary, add `CODE_SIGNING_ALLOWED=NO`.
- **Resource or Info.plist errors**: validate file paths and target membership.

### 4) Apply minimal fixes and rebuild

- Make the smallest change that addresses the error.
- Re-run the same build command after each fix.
- Clean derived data only when errors suggest stale artifacts:

```bash
rm -rf .build/DerivedData
```

### 5) Stop criteria

- Build completes with `** BUILD SUCCEEDED **`.
- Summarize the final build command, fixes applied, and any remaining warnings.

## Error Playbook

Use these quick heuristics to keep fixes focused:

- **"No such module"**: verify package integration; ensure target includes the package; resolve packages.
- **"Undefined symbols"**: add missing framework/library to target; verify `OTHER_LDFLAGS`.
- **"Duplicate symbols"**: remove duplicate source files or conflicting libraries.
- **"Swift package resolved but module missing"**: delete derived data, resolve packages, rebuild.
- **"Code signing is required for product type"**: build for simulator or set `CODE_SIGNING_ALLOWED=NO` for the build command only.

## Output Expectations

- Report the exact build command used.
- Include the top 1–3 actionable error lines.
- List each fix with the file path and rationale.
- Confirm the successful build, or explain the blocker if unresolved.
