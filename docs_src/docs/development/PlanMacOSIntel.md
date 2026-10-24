---
id: PlanMacOSIntel
title: Plan — macOS Intel (x86-64) Package for Katalog 3
description: Implementation plan for a separate Katalog 3 macOS package that runs on Intel Macs, built on a GitHub Intel runner with the official Qt binaries
---

# PLAN — macOS INTEL (x86-64) PACKAGE FOR KATALOG 3

![Status](https://img.shields.io/badge/Status-Draft-yellow) ![Implementation](https://img.shields.io/badge/Implementation-planned-yellow) ![K3](https://img.shields.io/badge/K3-3.0-blue)

## Context

Katalog 3 ships one macOS package, built by `.github/workflows/Katalog_3_Build_macOS.yml`
on the GitHub runner `macos-15` (Apple Silicon). The binary is `arm64` only: it
does not start on an Intel Mac, and the user documentation says so
(`Overview.md`, *macOS Compatibility*: "Intel Macs are **not** supported").
Rosetta 2 translates Intel software for Apple Silicon, never the other way
round, so there is no way to run the current package on Intel hardware.

This page plans a **separate Intel (x86-64) package**. It is a plan, not a
specification: nothing here is authorised until `SpecK3Deployment.md` is
amended (Phase 0). Decisions already taken with the maintainer of this fork on
2026-10-24:

- **K3 only.** K2 is in maintenance mode; a K2 Intel package can be cloned from
  the K3 workflow later if wanted.
- **Separate Intel-only DMG**, not a universal binary. Native x86-64 build and
  native launch test on an Intel runner; the Apple Silicon workflow is not
  modified.
- **Minimum macOS 15 (Sequoia)** for the Intel package in the first iteration,
  for parity with the Apple Silicon package. Lowering it is a listed follow-up.

**Non-goals:** K2 Intel package; universal (fat) binary; notarization or a
Developer ID signature; macOS below 15; any change under `core/`, `qt_quick/`
or `packaging/`; any user-visible string.

### Why the Apple Silicon recipe cannot simply be re-run on an Intel runner

Research of 2026-10-24:

- **Homebrew has stopped building Intel macOS bottles.** Homebrew's *Support
  Tiers* page places Intel x86_64 in Tier 3 for macOS 11 through 26, states
  "Homebrew has stopped building new bottles for Intel systems", and announces
  the removal of Intel support "in or after September 2027". The `qtbase` 6.12.0
  and `qtdeclarative` formulas carry only `arm64_golden_gate`, `arm64_tahoe`,
  `arm64_sequoia`, `arm64_linux` and `x86_64_linux` bottles. On an Intel runner
  `brew install qtbase` compiles Qt from source: hours against the 6-hour job
  limit, and exactly the defect class `DPL-C8` forbids (a runner-built Qt gave
  3.0.beta3 a libpng header/library mismatch and no icons).
- **Intel runners exist.** `actions/runner-images` lists `macos-15-intel`
  (also `macos-15-large`) and `macos-26-intel`; `macos-14-large` is deprecated.
  The macOS 15 Intel image ships Xcode 16.4 (default), Homebrew under
  `/usr/local`, CMake 4.4.2, Ninja 1.13.2, Python 3.14 and jq.
- **The official Qt binaries cover x86-64 macOS.** The Qt online repository
  (`mac_x64/desktop/qt6_6113/qt6_6113/Updates.xml`) publishes
  `qt.qt6.6113.clang_64` and the add-ons `qt5compat`, `qtcharts`,
  `qtimageformats` and `qtshadertools` — everything Katalog links
  (root `CMakeLists.txt`: Core, Widgets, Gui, Sql, Network, Charts,
  LinguistTools; `qt_quick/CMakeLists.txt`: Quick, QuickControls2, Sql, Charts,
  Qt ≥ 6.5). Qt 6.11 targets macOS 13 or later on `x86_64`, `x86_64h` and
  `arm64`; Qt 6.12 targets macOS 14.4 or later. They install in CI with
  `jurplel/install-qt-action@v4` (`host: mac`, `arch: clang_64`, `modules:`,
  `cache: true`), which sets `QT_ROOT_DIR`, `QT_PLUGIN_PATH`, `QML2_IMPORT_PATH`
  and prepends `$QT_ROOT_DIR/bin` to `PATH`. The official macOS frameworks are
  **universal**, so `CMAKE_OSX_ARCHITECTURES=x86_64` must be forced and the
  resulting architecture enforced, not assumed.
- **KF6 v6.22.0**, the tag the Apple Silicon workflow already builds from
  source as a fallback, requires Qt ≥ 6.8.0, ECM 6.22.0 and CMake ≥ 3.16 —
  compatible with Qt 6.11.3. `ki18n` needs LibIntl (gettext) and Python 3;
  `kiconthemes` keeps the existing BreezeIcons patch.

### Four facts of this repository that shape the workflow

1. **The Linux AppImage workflow already does "official Qt + KF6 from source
   into a dedicated prefix".** `.github/workflows/Katalog_3_Build_linux_qt6_appimage.yml`
   (lines 62-214) uses `jurplel/install-qt-action@v4` with `cache: true`, writes a
   `Qt6GuiPrivateConfig.cmake` stub into `$QT_ROOT_DIR/lib/cmake/Qt6GuiPrivate`
   (the official binaries lack it; `kguiaddons` and `kcolorscheme` link
   `Qt6::GuiPrivate`), prepends `find_package(Qt6GuiPrivate QUIET CONFIG)` to each
   framework's `src/CMakeLists.txt`, blanks `python/` so
   `ECMGeneratePythonBindings` never runs, caches the KF6 prefix with
   `actions/cache@v4` keyed by KF6 and Qt version, and configures with
   `-DCMAKE_PREFIX_PATH="${KF6_PREFIX};${QT_ROOT_DIR}"`. The Intel macOS workflow
   borrows these blocks. The Apple Silicon workflow never needed them because
   Homebrew's `qtbase` is a full build installed into the same prefix as KF6.
2. **Kirigami's QML module lands outside Qt's QML directory when KF6 has its own
   prefix.** On Apple Silicon, KF6 installs into `$(brew --prefix)`, which is also
   Qt's prefix, so ECM's `KDE_INSTALL_USE_QT_SYS_PATHS` puts Kirigami where
   `macdeployqt` scans. With `$HOME/kf6-install` it lands in
   `$KF6_PREFIX/lib/qml/org/kde/kirigami`, and the existing Theme.qml step would
   fail with "`$KIRIGAMI_DIR` is missing". Fix: `macdeployqt … -qmlimport="$KF6_PREFIX/lib/qml"`.
3. **`install-qt-action` exports `QT_PLUGIN_PATH` and `QML2_IMPORT_PATH`.** Left
   set, the launch test would resolve plugins and QML from the Qt install instead
   of the bundle. The app is launched with
   `env -u QT_PLUGIN_PATH -u QML2_IMPORT_PATH -u QML_IMPORT_PATH`, in addition to
   hiding `$QT_ROOT_DIR`.
4. **The Apple Silicon KF6 loop's error detection is broken and must not be copied
   verbatim.** `Katalog_3_Build_macOS.yml` lines 160-161:
   `if cmake .. "${CMAKE_ARGS[@]}" 2>&1 | tail -5; then` tests the exit code of
   `tail`, not of `cmake` (no `pipefail`), so a failing framework prints
   "Built and installed" and the job only dies later at Katalog's configure step.
   On Intel there is no Homebrew fallback, so the loop uses `set -o pipefail` and
   exits on the first clone, configure, build or install failure.

### Chosen approach

A **new workflow** `.github/workflows/Katalog_3_Build_macOS_intel.yml` on
`macos-15-intel`, **Qt 6.11.3 from the official binaries**, **KF6 6.22.0 built
from source** into a dedicated prefix, **`x86_64` enforced**, delivering
`Katalog_macos_intel_v<version>.dmg` next to the existing
`Katalog_macos_v<version>.dmg`.

---

## Phase 0 — Specification amendment (the gate)

Nothing in Phase 1 starts before this phase is approved and written. Per the
project rules the order is: `specifications` agent verdict → maintainer approval
→ `Spec*.md` edit → implementation. Requirement IDs are permanent: never
renumbered, never reused.

**File `docs_src/docs/development/SpecK3Deployment.md`**

| Row | Change |
|-----|--------|
| `DPL-C7` | Scope the first sentence to "The K3 macOS **Apple Silicon** package is built on …". Replace the last sentence *"No Intel or universal build is authorised."* with: *"No **universal** build is authorised. (Amended 2026-10-24: this row and `DPL-C8` govern the Apple Silicon workflow only. The Intel package is a separate build with its own rules, `DPL-F8` / `DPL-C10` / `DPL-C11`; it cannot take this row's Homebrew route because Homebrew publishes no Intel macOS bottles of Qt.)"* |
| `DPL-F6` | *"Minimum macOS for the K3 package is **macOS 15 (Sequoia)**, Apple Silicon only."* → *"Minimum macOS for both K3 packages — Apple Silicon (`DPL-C7`) and Intel (`DPL-F8`, `DPL-C10`) — is **macOS 15 (Sequoia)**. (Amended 2026-10-24: "Apple Silicon only" removed; the Intel package is added by `DPL-F8`.)"* |
| `DPL-C5` | *"Likewise for `DPL-F6` / `DPL-C7`–`C9`"* → *"Likewise for `DPL-F6` / `DPL-F8` / `DPL-C7`–`C11`"*. |
| **`DPL-F8`** (new, `[Planned]`) | A K3 macOS **Intel (x86-64)** package exists as a separate download, `Katalog_macos_intel_v<version>.dmg`, alongside the Apple Silicon `Katalog_macos_v<version>.dmg`. On an Intel Mac running macOS 15 or later it starts with the documented right-click-Open workaround, shows the same working features as the Apple Silicon package (`DPL-O1`), follows light/dark (`DPL-F5`), loads its images (`DPL-F6`), and its About page reports `Architecture: x86_64`. It is not a universal binary; Apple Silicon users take the arm64 package. Running it under Rosetta 2 on Apple Silicon is a verification aid, not a supported configuration. |
| **`DPL-C10`** (new, `[Planned]`) | The Intel package is built by a **separate** workflow, `.github/workflows/Katalog_3_Build_macOS_intel.yml`, on the GitHub runner `macos-15-intel`. Qt comes from the **official Qt binaries** (`jurplel/install-qt-action`, `host: mac`, `arch: clang_64`) at a **pinned** release named once in the workflow (6.11.3 at the time of writing) — never from Homebrew, never built on the runner (the `DPL-C8` defect class). KF6 is built from source at the same version as the Apple Silicon workflow (6.22.0), against that Qt, into a dedicated prefix. KF6 and Katalog are both configured with `-DCMAKE_OSX_ARCHITECTURES=x86_64`, and the workflow MUST fail unless `lipo -archs` of the app binary is exactly `x86_64` (the official Qt binaries are universal, so x86_64-only is enforced, not assumed). Minimum macOS is 15 in this iteration: the Homebrew-built `libintl` that KI18n requires targets the runner's macOS; lowering it is a separate change amending this row and `DPL-F6`. Moving the runner (`macos-26-intel`, or when GitHub retires Intel runners) or changing the Qt pin is a deliberate change recorded here. The Apple Silicon workflow is not changed. |
| **`DPL-C11`** (new, `[Planned]`) | The Intel workflow MUST perform the Apple Silicon workflow's shippability checks, adapted to its layout: (a) the Qt actually configured (`Qt6_DIR` in Katalog's CMake cache) resolves under the install-qt-action directory and its `QtCore.framework` carries an `x86_64` slice; (b) `macdeployqt` from that Qt runs with `-qmldir` **and** `-qmlimport` on the KF6 prefix's QML directory, so Kirigami is bundled; `packaging/Theme.qml` is copied per `DPL-C6`; `libqcocoa` and `libqsqlite` are present; (c) the otool sweep rejects any dependency under `/usr/local` (Homebrew on Intel), `/opt/homebrew`, `/opt/local` or `/Users/` (the Qt and KF6 install directories), and every bundled Mach-O carries `x86_64`; (d) the launch test runs with the Qt directory, the KF6 prefix and Homebrew's `Cellar`/`opt` hidden and `QT_PLUGIN_PATH`/`QML2_IMPORT_PATH` unset, failing on dyld / plugin-not-found lines and on `libpng warning: Application built with` (`DPL-C9` applies to this package too); (e) an ad-hoc signature is applied and verified. |
| *Scope at a glance* | In scope: append "— and, added 2026-10-24 at the maintainer's direction, a separate **Intel (x86-64) macOS package** (`DPL-F8`, `DPL-C10`, `DPL-C11`)". Out of scope: extend the exception list to `DPL-F8` / `DPL-C10` / `DPL-C11`; add "A universal (fat) macOS binary. A K2 Intel package. A macOS minimum below 15 for the Intel package." |
| *Known limitations* | Row *"macOS 14 (Sonoma) and Intel Macs not supported"* → item *"macOS 14 (Sonoma) not supported; Intel Macs: separate package"*. Keep the `arm64_sonoma` cause, then add: *"(Amended 2026-10-24) Intel Macs are served by a **separate** Intel package (`DPL-F8`); there is no universal binary. Its macOS 15 floor comes from Homebrew-built `libintl` targeting the runner's macOS (`DPL-C10`)."* |
| *Manual test charter* | Amend the `DPL-C7` line to name the Apple Silicon workflow. Add `DPL-F8`, `DPL-C10` and `DPL-C11` lines (text in Phase 3 below). |

**File `docs_src/docs/development/SpecK2Deployment.md`**

| Row | Change |
|-----|--------|
| `K2D-C6` | *"No Intel or universal build is authorised. Mirrors `DPL-C7`"* → *"No universal build is authorised, and no **K2** Intel build is authorised in this round — the K3 Intel package (`DPL-F8` / `DPL-C10`, SpecK3Deployment) is a K3-only decision. Mirrors `DPL-C7`"*. |
| *Known limitations* | Row *"macOS 14 (Sonoma) and Intel Macs not supported"*: append *"The K3 Intel package does not extend to K2."* |

`Specifications.md` needs no new row: both pages are already indexed.

---

## Phase 1 — New workflow `Katalog_3_Build_macOS_intel.yml`

Header: `name: Katalog 3 - Build macOS Intel`; comment
`#v3.0 run1 - Intel x86-64, official Qt binaries (SpecK3Deployment DPL-C10/C11)`;
`workflow_dispatch` with input `version_tag` (default `v3.0-macos-intel`), the
same shape as the other workflows. Job `build-macos-k3-intel`,
`runs-on: macos-15-intel`, job-level `env: QT_VERSION: "6.11.3"` and
`KF6_VERSION: "v6.22.0"` so each pin appears exactly once. Every `run:` block is
bash 3.2 (no `${var^}`); `set -o pipefail` wherever a pipe is used.

Origin of each step: **verbatim** = copied from `Katalog_3_Build_macOS.yml`
(line numbers refer to it); **Linux** = copied from
`Katalog_3_Build_linux_qt6_appimage.yml`; **new** = written for this workflow.

| # | Step | Origin | Content |
|---|------|--------|---------|
| 1 | Checkout | verbatim (L22-25) | `actions/checkout@v4`, `submodules: recursive`. |
| 2 | Install Qt (official binaries) | new; pattern Linux L62-70 | `jurplel/install-qt-action@v4` with `version: ${{ env.QT_VERSION }}`, `host: mac`, `target: desktop`, `arch: clang_64`, `modules: 'qtcharts qt5compat qtimageformats qtshadertools'`, `cache: true`, `cache-key-prefix: qt-mac-intel`. Default install dir `/Users/runner/work/Katalog/Qt` — under `/Users/`, which the otool sweep already rejects. |
| 3 | Assert Qt is the official x86_64-capable binary | new; replaces "poured from bottle" (L52-66) — `DPL-C10` | `test -d "$QT_ROOT_DIR/lib/QtCore.framework"`; `qmake -query QT_VERSION` equals `$QT_VERSION`; `realpath "$(qmake -query QT_INSTALL_PREFIX)"` equals `realpath "$QT_ROOT_DIR"`; `lipo -archs "$QT_ROOT_DIR/lib/QtCore.framework/Versions/A/QtCore"` contains `x86_64`; `! brew list --formula \| grep -qE '^qt(base)?$'` (no Homebrew Qt to shadow it). Any failure exits 1. Summary: `- **Qt**: 6.11.3 official binaries (x86_64 slice), not built on the runner`. |
| 4 | Install build dependencies | differs (L27-86) | Keep the `command -v cmake` / `ninja` guards (preinstalled). `brew install gettext` for KI18n's LibIntl — on Tier 3 it may compile from source (minutes); `poured_from_bottle` is recorded as information, not enforced. `brew install create-dmg \|\| true`. **Dropped:** every Qt formula, `extra-cmake-modules` (ECM is built from source in the KF6 loop), the `kde-mac/kde` tap and `kf6-*` formulas (they would link Homebrew Qt into `/usr/local`), `imagemagick` (unused). |
| 5 | Set environment | differs (L88-93) | `KF6_PREFIX=$HOME/kf6-install`, `GETTEXT_PREFIX=$(brew --prefix gettext)`, `CMAKE_BUILD_PARALLEL_LEVEL=$(sysctl -n hw.ncpu)` → `$GITHUB_ENV`. Do **not** set `CMAKE_PREFIX_PATH=$(brew --prefix)`. |
| 6 | Patch Qt for the GuiPrivate target | Linux L72-100 | Same `Qt6GuiPrivateConfig.cmake` stub, `6.11.2` → `${QT_VERSION}`, written only when absent. Print `ls "$QT_ROOT_DIR/include/QtGui/"` first: on the framework layout `include/QtGui` points to `QtGui.framework/Headers`, with private headers under `6.11.3/QtGui/private` — confirm in run 1 and adjust the two include paths if needed. |
| 7 | Restore KF6 cache | Linux L102-110 | `actions/cache@v4`, `id: kf6-cache`, `path: ~/kf6-install`, `key: kf6-${{ env.KF6_VERSION }}-qt${{ env.QT_VERSION }}-macos-15-intel-x86_64-v1`. |
| 8 | Build KF6 from source | adapted (L95-171) | `if: steps.kf6-cache.outputs.cache-hit != 'true'`. Same component list and order (`extra-cmake-modules kcoreaddons ki18n kconfig kwidgetsaddons kcodecs kguiaddons kcolorscheme kitemviews karchive kconfigwidgets kiconthemes kfilemetadata kirigami`), same clone-by-tag with fallback, same `kiconthemes` BreezeIcons `sed` patch. **Differs:** `-GNinja -DCMAKE_PREFIX_PATH="$KF6_PREFIX;$QT_ROOT_DIR;$GETTEXT_PREFIX" -DCMAKE_INSTALL_PREFIX=$KF6_PREFIX -DCMAKE_OSX_ARCHITECTURES=x86_64 -DCMAKE_OSX_DEPLOYMENT_TARGET=11.0 -DBUILD_TESTING=OFF -DBUILD_QCH=OFF -DQT_MAJOR_VERSION=6 -DBUILD_PYTHON_BINDINGS=OFF`; prepend `find_package(Qt6GuiPrivate QUIET CONFIG)` to `src/CMakeLists.txt` and blank `python/` (Linux L150-160); **remove** the `pkg-config --exists KF6…` skip (no Homebrew KF6 — the cache covers reruns); **fail fast** with `set -o pipefail` and `exit 1` on the first failure (fact 4). |
| 9 | Verify KF6, locate the Kirigami QML directory | new; runs on cache hit too | Assert `$KF6_PREFIX/lib/cmake/KF6CoreAddons`, `KF6Config`, `KF6FileMetaData`, `KF6I18n`, `KF6Archive` exist. `find "$KF6_PREFIX" -path '*/org/kde/kirigami/qmldir'` → derive `KF6_QML_DIR` (expected `$KF6_PREFIX/lib/qml`) → `$GITHUB_ENV`; fail if empty. Summary: `- **KF6**: 6.22.0 built from source (x86_64)`. |
| 10 | Set version information | adapted (L173-184) | Same grep of `KATALOG_VERSION_STRING` in `qt_quick/CMakeLists.txt`; `ARTIFACT_BASE=Katalog_macos_intel_v${KATALOG_VERSION}`. |
| 11 | Build Katalog 3 | adapted (L186-237) | `cmake .. -GNinja -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH="$KF6_PREFIX;$QT_ROOT_DIR;$GETTEXT_PREFIX" -DQt6_DIR="$QT_ROOT_DIR/lib/cmake/Qt6" -DCMAKE_OSX_ARCHITECTURES=x86_64 -DCMAKE_OSX_DEPLOYMENT_TARGET=11.0 -DBUILD_TESTS=OFF -DBUILD_QT_WIDGETS=OFF -DBUILD_QT_QUICK=ON -DCMAKE_INSTALL_PREFIX=/Applications`. After configure, `grep '^Qt6_DIR' CMakeCache.txt` must point under `$QT_ROOT_DIR` (`DPL-C11` a). Translations and `ninja` as before. **Architecture enforced:** `ARCHS=$(lipo -archs qt_quick/Katalog.app/Contents/MacOS/Katalog)`; `[ "$ARCHS" = "x86_64" ]` or write `- **Architecture**: $ARCHS - build rejected` to the summary and exit 1. Summary: `### macOS Intel build`, `- **Architecture**: x86_64 (Intel)`, `- Apple Silicon Macs: use the arm64 package`. Rewrite the "Recorded rather than enforced" comment accordingly. |
| 12 | Deploy Qt6 dependencies | adapted (L239-314) | `QT_DEPLOY_TOOL="$QT_ROOT_DIR/bin/macdeployqt"`; missing → exit 1 (a wrong Qt install, not a soft condition). `"$QT_DEPLOY_TOOL" qt_quick/Katalog.app -qmldir=../qt_quick -qmlimport="$KF6_QML_DIR" -verbose=1` (fact 2). QML-directory check, `Theme.qml` copy (`DPL-C6`), SQLite and Cocoa plugin checks verbatim. |
| 13 | Thin universal binaries to x86_64 (optional) | new | Before signing: for each regular-file Mach-O in the bundle whose `lipo -archs` lists more than one architecture, `lipo "$f" -thin x86_64 -output "$f.thin" && mv "$f.thin" "$f"`. Symlinks untouched. Report `du -sh` before and after in the summary. The Qt share of the DMG roughly halves. |
| 14 | Verify the bundle is self-contained | verbatim regex (L316-371) + new asserts | The prefix regex is unchanged (`/usr/local` is Homebrew on Intel; `/Users/` covers the Qt and KF6 directories) — update the comment. **Add:** per-file `lipo -archs "$f" \| grep -qw x86_64` or fail; positive asserts that `Contents/Frameworks/QtCore.framework` and a `libKF6CoreAddons*.dylib` are present. |
| 15 | Sign the bundle (ad-hoc) and verify | verbatim (L373-407) | Comment tweak: on Intel a signature is not required to execute, but a broken or absent one still yields Gatekeeper's "damaged" message. |
| 16 | Smoke test — start with Qt, KF6 and Homebrew hidden | adapted (L409-482) | `/usr/local` cannot be moved wholesale. Hide: `mv "$QT_ROOT_DIR" "$QT_ROOT_DIR.hidden"`, `mv "$KF6_PREFIX" "$KF6_PREFIX.hidden"`, `sudo mv /usr/local/Cellar /usr/local/Cellar.hidden`, `sudo mv /usr/local/opt /usr/local/opt.hidden` (Homebrew's `/usr/local/lib` symlinks then dangle, so nothing resolves). Launch: `env -u QT_PLUGIN_PATH -u QML2_IMPORT_PATH -u QML_IMPORT_PATH "$APP/Contents/MacOS/Katalog" > launch.log 2>&1 &` (fact 3); wait 20 s; restore all four **before** any check. Greps verbatim: `libpng warning: Application built with` is fatal (`DPL-C9`); dyld / `plugin … not found` / `no Qt platform plugin` lines are fatal. Summary label: `Starts without Qt, KF6 or Homebrew`. |
| 17 | Create DMG | adapted (L484-510) | `--volname "Katalog $KATALOG_VERSION (Intel)"`; `hdiutil` fallback unchanged; file `Katalog_macos_intel_v3.0.dmg`. |
| 18-19 | Upload artifacts | verbatim templates | `${{ env.ARTIFACT_BASE }}_dmg` (30 days), `${{ env.ARTIFACT_BASE }}_bundle` (7 days). |
| 20 | Summary | adapted (L526-531) | `## Katalog 3 macOS Intel Build`: Version, File, Runner, Qt pin and origin, KF6 version, Architecture, Minimum macOS 15, Status. |

**Not touched:** `Katalog_3_Build_macOS.yml`, `Katalog_2_Build_macOS.yml`,
`CMakeLists.txt` at any level, anything under `core/`, `qt_quick/` or
`packaging/`, any `tr()` / `qsTr()` string (`DPL-C5` holds). The About page
already prints `Architecture: x86_64` / `arm64` from the running process, which
is what support needs to tell the two packages apart.

---

## Phase 2 — Documentation (same change as Phase 1, after a green run)

Sources live in `docs_src/`; `docs/` at the repository root is the generated
site, regenerated with `npm run build` and copied — never edited by hand.

| File | Change |
|------|--------|
| `docs_src/docs/Overview.md`, *Multi-Platform Support* table and *macOS Compatibility* | Table row: `15 (Sequoia)` → `15 (Sequoia), Apple Silicon or Intel`. *Tested on*: add the Intel package, marked as awaiting tester confirmation until `DPL-F8` is verified. *Architecture*: replace the "Apple Silicon only … Rosetta 2 …" bullet with two packages — `Katalog_macos_v3.0.dmg` for Apple Silicon (M1 and later), `Katalog_macos_intel_v3.0.dmg` for Intel Macs; pick by Apple menu → *About This Mac* (*Chip* = Apple Silicon, *Processor* = Intel); the packages are not interchangeable and neither is universal. *Build environment*: Apple Silicon — Qt 6 and KF6 via Homebrew; Intel — official Qt 6 binaries (6.11.3) and KF6 6.22 built from source. |
| `docs_src/i18n/fr/docusaurus-plugin-content-docs/current/Overview.md` and `docs_src/i18n/cs/docusaurus-plugin-content-docs/current/Overview.md` | The same four edits, translated; file names stay literal. Version badges follow the release that ships the package. |
| `docs_src/docs/development/Dev-Overview.md`, *GitHub workflows* | Add `- [Katalog 3 macOS Intel](https://github.com/StephaneCouturier/Katalog/actions/workflows/Katalog_3_Build_macOS_intel.yml)` after the Katalog 3 macOS entry. |
| `docs_src/docs/development/Dev-Release.md` | Functional updates: "macOS ymls (Apple Silicon and Intel) if using new KF6 libraries". Packaging: "macOS versions with GitHub Actions: Apple Silicon DMG and Intel DMG". Publishing: the default macOS package stays the Apple Silicon DMG. |
| Release notes (GitHub and SourceForge release text) | One line: "macOS: a separate Intel (x86-64) package, `Katalog_macos_intel_v3.0.dmg`, is now provided; Apple Silicon users keep `Katalog_macos_v3.0.dmg`. Both need macOS 15." The Flatpak `appdata.xml` entry is Linux-facing and does not change. |

No user-visible string changes anywhere.

---

## Phase 3 — Verification

**Workflow run** (`workflow_dispatch` on the feature branch; expect 45-75
minutes cold, 15-20 minutes with warm Qt and KF6 caches). The step summary
must show: Qt `6.11.3` official binaries (x86_64 slice) · KF6 `6.22.0` from
source · Architecture `x86_64` · QML modules bundled · Kirigami colour theme
bundled · SQLite driver bundled · Platform plugin bundled · Self-contained yes
(≥ 10 binaries) · Code signature valid · Starts without Qt, KF6 or Homebrew: yes
or inconclusive, never NO · File `Katalog_macos_intel_v3.0.dmg`. Inspect in the
log: whether `gettext` was poured or built and how long it took, the
`include/QtGui/` layout printed by step 6, `macdeployqt` lines mentioning
`org/kde/kirigami`, bundle size before and after thinning. Negative checks: feed
the captured 3.0.beta3 launch log through the step-16 grep (the `DPL-C9`
charter); force a wrong `lipo` comparison once to confirm step 11 exits
non-zero. `git diff --stat` touches only the new workflow and `docs_src/`.

**DMG on an Intel Mac (macOS 15)** — mount, right-click → *Open* (the Gatekeeper
path from `Overview.md`); About page shows `Architecture: x86_64` and the Qt /
KF6 lines; toolbar and drawer icons and combo-box arrows are drawn;
`file Katalog.app/Contents/MacOS/Katalog` reports `x86_64`; from Terminal,
`QT_FORCE_STDERR_LOGGING=1 Katalog.app/Contents/MacOS/Katalog` prints no
`libpng warning`, no `module "…" is not installed`, no
`Failed to load non-native`; create a File-mode collection (the file dialog
appears, SQLite works); toggle dark mode (`DPL-F5`).

**On Apple Silicon under Rosetta (pre-check only)** —
`softwareupdate --install-rosetta --agree-to-license`; mount the DMG;
`xattr -cr` the bundle; `arch -x86_64 "/Volumes/Katalog 3.0 (Intel)/Katalog.app/Contents/MacOS/Katalog"`;
Activity Monitor shows *Kind: Intel*; About shows `x86_64`. This validates bundle
completeness, dynamic linking, QML modules and libpng — not Intel-GPU rendering.
Final sign-off is the Intel Mac.

**Manual test charter lines to add in Phase 0:**

- **DPL-F8** — On an Intel Mac / macOS 15, mount `Katalog_macos_intel_v3.0.dmg`,
  right-click → Open. The window opens; About shows `Architecture: x86_64`;
  icons and combo-box arrows are drawn; a File-mode collection can be created;
  dark mode is followed. From Terminal: no `libpng warning`, no
  `module "…" is not installed`. Pre-check on Apple Silicon:
  `arch -x86_64 …/Contents/MacOS/Katalog`.
- **DPL-C10** — The workflow's `runs-on` is `macos-15-intel`; the step summary
  shows Qt `6.11.3` official binaries, KF6 `6.22.0` from source, Architecture
  `x86_64`; the architecture step exits non-zero on anything else.
- **DPL-C11** — Summary rows *Self-contained*, *Starts without Qt, KF6 or
  Homebrew* and *Images load* are all green; `git diff --stat` on
  `Katalog_3_Build_macOS.yml` is empty.

---

## Risks and mitigations

| Risk | Mitigation |
|------|------------|
| KF6 6.22 against the official Qt binaries: missing `Qt6GuiPrivateConfig.cmake` (`kguiaddons`, `kcolorscheme`), Python-binding `find_package(Shiboken6)` calls | Steps 6 and 8 reuse the Linux workflow's proven stub, prepend and `-DBUILD_PYTHON_BINDINGS=OFF` with a blanked `python/`. Compatibility itself is low-risk: Linux already builds KF6 6.24 on the official Qt 6.11.2. |
| Kirigami QML not bundled because KF6 has its own prefix | `-qmlimport="$KF6_QML_DIR"` in step 12; the Theme.qml step already fails loudly when the directory is absent. |
| Launch test masked by `QT_PLUGIN_PATH` / `QML2_IMPORT_PATH` | `env -u …` plus hiding `$QT_ROOT_DIR` (step 16). |
| Homebrew on Intel compiles `gettext` / `create-dmg` from source (Tier 3) | Tolerated: minutes, and not Qt. `create-dmg` is optional (`hdiutil` fallback). ECM is cloned in the KF6 loop. |
| KF6 ↔ Qt version mismatch | KF 6.22.0 requires Qt ≥ 6.8; Qt 6.11.3 satisfies it. Bump both pins together, never one. |
| Silent KF6 build failure from the copied `\| tail` pattern | `set -o pipefail` and `exit 1` in the loop (fact 4). |
| Job duration (Intel runner, 14 frameworks from source) | `cache: true` for Qt, `actions/cache` for the KF6 prefix, fail-fast loop, no `imagemagick` or KDE tap. About one hour cold; the 6-hour limit is far. |
| Universal Qt frameworks inflate the DMG | Optional step 13 (`lipo -thin x86_64`) before signing; `codesign --force --deep` afterwards re-signs everything. |
| `macos-15-intel` retirement | GitHub still adds Intel labels (`macos-26-intel`). `runs-on` is one line; `DPL-C10` names the runner so a move is an explicit amendment. Check `actions/runner-images` announcements before each release. |
| Deployment target 11.0 versus Homebrew `libintl` built for macOS 15 | Possible `ld: object file … built for newer macOS version` warnings, harmless. Keep 11.0 for parity with the Apple Silicon workflow; the package floor is documented as 15 regardless. Follow-up: build gettext from source with an explicit `MACOSX_DEPLOYMENT_TARGET` to lower the floor to 13. |
| Homebrew libraries under `/usr/local` picked up silently (e.g. xz, zstd for `karchive`) | Same situation as `/opt/homebrew` on Apple Silicon today: `macdeployqt` bundles them, and the sweep's `/usr/local` pattern catches anything left unrewritten. |
| No Intel Mac available to test | Rosetta on Apple Silicon as a proxy (`arch -x86_64`); ask a beta tester with an Intel Mac before publishing. |
| Specification gate | Phase 1 does not start before Phase 0 is approved and written. |

---

## Follow-ups (out of scope here)

- **K2 Intel package**: clone this workflow with `-DBUILD_QT_WIDGETS=ON -DBUILD_QT_QUICK=OFF`,
  the K2 KF6 list (KXmlGui, KCompletion) and a `K2D-C6` amendment.
- **Lower minimum macOS** for the Intel package to 13 (Qt 6.11 floor): build
  gettext from source with an explicit deployment target instead of Homebrew.
- **Universal binary** replacing the two DMGs: requires every non-Qt dependency
  built for both architectures; would amend `DPL-C7` and `DPL-C10` again.
- **Notarization** with a Developer ID (removes the Gatekeeper workaround on
  managed Macs) — unchanged by this plan, same situation for both packages.
