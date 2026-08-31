# CI — Windows 11 arm64 with Visual Studio 2026

Workflow: [`.github/workflows/build-windows11-arm64-vs2026.yml`](../.github/workflows/build-windows11-arm64-vs2026.yml)

Builds `assembleDebug` + `assembleRelease`, runs unit tests, reports Android Lint,
and uploads the APKs (plus the R8 mapping file) as run artifacts.

## Running it

**Manual only** — the workflow has a single `workflow_dispatch` trigger. Nothing
runs on push or on a pull request.

Actions → *Build (Windows 11 arm64 · VS 2026)* → **Run workflow**, then pick the
branch and any inputs.

> Dispatching from a non-default branch works: this workflow was dispatched
> against its feature branch via `POST /actions/workflows/{id}/dispatches` with
> `ref` set to that branch, and the run executed there, before anything was
> merged to `master`.
>
> If the *Run workflow* button is not visible in the Actions UI for a branch
> that has not been merged yet, use the API or `gh workflow run <file> --ref
> <branch>` instead — the dispatch itself is not blocked.

| Input | Default | Notes |
|---|---|---|
| `runner` | `windows-11-vs2026-arm` | or `windows-11-arm` |
| `java_version` | `17` | or `21` |
| `java_arch` | `x64` | `aarch64` is expected to fail on Gradle 8.9 — see below |
| `build_release` | `true` | off = debug variant only |
| `require_vs2026` | `true` | off = warn instead of failing when VS is not v18 |

---

## Runner

| | |
|---|---|
| Label | `windows-11-vs2026-arm` |
| OS | Windows 11 Enterprise 25H2, arm64 |
| Visual Studio | Enterprise 2026 (v18) — `C:\Program Files\Microsoft Visual Studio\18\Enterprise` |
| JDKs on the image | 21 and 23, **aarch64 only** |
| Android SDK on the image | **none** |
| Cost | free minutes for public repositories |

GitHub is rolling Visual Studio 2026 into the plain `windows-11-arm` label between
21 and 30 September 2026, after which both labels carry VS 2026. The label is a
`workflow_dispatch` input (`runner`) so it can be switched without editing the file.

The Gradle/Android build does not need the MSVC toolchain — this project has no
NDK or CMake sources. The `Locate Visual Studio 2026` step records which VS the
APKs came from and fails the job if the runner is not on v18 (turn that off with
the `require_vs2026` input).

---

## The ARM64 compatibility problem

**Gradle runs natively on Windows/ARM64 only from Gradle 9.2.0.** Before that,
`native-platform` has no `windows-aarch64` build and Gradle crashes as soon as it
reads the Windows registry — which it does routinely to discover JDK installations
and Visual Studio paths.

This project is pinned to **Gradle 8.9** (`gradle/wrapper/gradle-wrapper.properties`)
because **AGP 8.5.2** does not support Gradle 9.x. Upgrading the wrapper alone is
not an option; it would require an AGP upgrade as well.

### The fix: run the JVM under x64 emulation

The workflow installs an **x86-64 Temurin JDK 17** and runs the unmodified Gradle 8.9
on it. Gradle then loads its `amd64` native-platform DLLs, which Windows 11 on ARM
executes through its built-in x64 emulation (Prism). Nothing in the build has to change.

This costs nothing in correctness and little in coverage, because the rest of the
Android toolchain on Windows is x86-64-only regardless of which JDK is used:

- `aapt2` is published solely as `com.android.tools.build:aapt2:<version>:windows` (x86-64);
- SDK build-tools ship x86-64 executables (`zipalign`, `aidl`, `split-select`).

Those run emulated either way.

Measured on the first green run (cold caches, no Gradle or SDK cache to restore),
total 18m42s:

| Step | Time |
|---|---|
| Install Android SDK | 43s |
| `assembleDebug` | 8m31s |
| `testDebugUnitTest` | 47s |
| `lintDebug` | 2m08s |
| `assembleRelease` (R8, minified) | 5m05s |

A warm run (both caches restored) took **7m43s** for the same work:

| Step | Cold | Warm |
|---|---|---|
| Install Android SDK | 43s | skipped (cache hit) |
| `assembleDebug` | 8m31s | 1m08s |
| `testDebugUnitTest` | 47s | 42s |
| `lintDebug` | 2m08s | 1m51s |
| `assembleRelease` (R8, minified) | 5m05s | 1m36s |
| **total** | **18m42s** | **7m43s** |

The Gradle build cache (enabled for CI in `GRADLE_USER_HOME`) does most of that.
No x64 comparison run exists, so treat these as the emulated baseline rather than
a measured slowdown factor.

### Trying the native aarch64 lane

`workflow_dispatch` exposes `java_arch: aarch64`, which installs a **Zulu** JDK
(Temurin publishes no Windows/aarch64 JDK 17; Zulu does). It is expected to fail on
Gradle 8.9 and the workflow emits a warning saying so. It becomes viable after a
future Gradle 9.2+ / AGP upgrade.

### Other image gaps the workflow closes

| Gap | Handling |
|---|---|
| No JDK 17 on the image | `actions/setup-java` — Temurin for x64, Zulu for aarch64 |
| No Android SDK | `cmdline-tools` downloaded to `C:\android-sdk`, licences accepted, `platform-tools` + `platforms;android-35` + `build-tools;34.0.0` and `35.0.0` installed, verified, then cached |
| `sdkmanager` splitting package names | Package coordinates contain `;`, which `sdkmanager.bat`'s cmd tokenizer splits on unless the argument arrives quoted — PowerShell only auto-quotes arguments containing spaces. They are quoted explicitly and passed via `Start-Process`. |
| cmdline-tools `latest` | Pinned to 19.0. From 23.0, `sdkmanager` is a deprecation shim forwarding to the new [Android CLI](https://d.android.com/tools/agents/android-cli), which changes argument handling and drops `--licenses`. |
| Build-tools 34.0.0 | AGP 8.5.2's default `buildToolsVersion`; 35.0.0 matches `compileSdk 35` |
| Deep KSP/Compose output paths | `git config --global core.longpaths true`, SDK at a short space-free path |
| 2 GB heap in `gradle.properties` | CI-only override in `%USERPROFILE%\.gradle\gradle.properties` (4 GB Gradle, 3 GB Kotlin daemon, build cache on) — the repo's own properties are untouched |
| Defender scanning the caches | best-effort exclusions for the workspace, `~\.gradle` and the SDK |

---

## Release signing

`app/build.gradle.kts` points `signingConfigs.release` at `keys/release.keystore`,
which is `.gitignore`d and therefore absent on a fresh clone. Without it,
`assembleRelease` fails at `validateSigningRelease`.

**With a real key** — add a repository secret `RELEASE_KEYSTORE_BASE64`:

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("keys\release.keystore")) | Set-Clipboard
```

The keystore must use the alias and passwords hard-coded in `app/build.gradle.kts`;
the workflow reads them from that file rather than restating them. Artifacts from
such a run are named `apk-release-secret-<run>`.

**Without the secret** the workflow generates a throw-away key so the release
variant still compiles and R8 still runs. Those artifacts are named
`apk-release-ephemeral-<run>` and are **build verification only — do not distribute
them**; they will not upgrade an installed copy signed with the real key.

To sign from secrets without matching the committed credentials, make the signing
config read the environment, e.g.:

```kotlin
storePassword = System.getenv("RELEASE_STORE_PASSWORD") ?: "…"
```

---

## Lint

`lintDebug` runs with `continue-on-error: true` and the HTML/XML reports are
uploaded. The project has no lint baseline yet, so enforcing `abortOnError` would
turn pre-existing findings into a red first build. Once a baseline exists, drop
`continue-on-error` from the `Android Lint (report only)` step.

---

## Bumping the pinned versions

| Pin | Where |
|---|---|
| `cmdline-tools` build number | `CMDLINE_TOOLS_ZIP` in the workflow `env:` — the cache key interpolates it, so it invalidates automatically. Moving to 23.0+ means adopting the Android CLI. |
| Platform / build-tools | `ANDROID_PLATFORMS`, `ANDROID_BUILD_TOOLS_1/2` — bump the SDK cache key with them |
| JDK | `java_version` input default |
