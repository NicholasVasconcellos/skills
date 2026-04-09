---
name: android-setup
description: >-
  Set up, build, test, and run a React Native Expo app on the Android emulator.

  Use this skill when the user wants to test an Android build, run their app on
  an Android emulator, set up the Android toolchain, or verify their Android
  build works. Covers the full lifecycle: environment detection and installation,
  Expo prebuild, Gradle build, APK verification, emulator launch, and generating
  a reusable run-android.sh script in the project.

  Common triggers:
  - "build for Android"
  - "run on Android emulator"
  - "test the Android build"
  - "set up Android development"
  - "get Android working"
---

# android-setup

Set up, build, verify, and run a React Native Expo project on Android emulator.
Generate a reusable `run-android.sh` script in the project root.

## Principles

- **Project-agnostic.** All project-specific details (package name, native modules,
  permissions, NDK requirements) are discovered by reading the project, never hardcoded.
- **Idempotent.** Detect what's already installed before installing anything.
  Never reinstall or overwrite existing tooling.
- **Incremental.** Commit fixes as you go so the user can track what changed.
- **Fail loudly.** Every verification step must produce clear pass/fail output.
  Never silently skip a broken step.

---

## Phase 1 — Discover the project

Read these files to understand the project before doing anything:

1. `package.json` — dependencies, scripts, Expo version
2. `app.json` or `app.config.js` — Android package name, permissions, plugins
3. `android/` directory — check if it exists (Expo prebuild may not have run yet)
4. `android/app/build.gradle` — if it exists, read for minSdk, NDK version, native build config
5. Look for native modules with C++ code:
   - `modules/**/android/**/CMakeLists.txt`
   - `modules/**/android/**/build.gradle` (look for `externalNativeBuild`)
   - If found, the project needs the Android NDK
6. `android/gradle.properties` — check for `newArchEnabled`, Hermes settings
7. Any `progress.md`, `CLAUDE.md`, or `PROJECT.md` — prior build notes and known issues

Record what you learn:

- **App package name** (e.g., `com.anonymous.MyApp`)
- **Target SDK / min SDK** versions
- **Whether NDK is needed** (and which version if specified in build.gradle)
- **Whether Expo prebuild has been run** (`android/` exists and is populated)
- **Native modules** that may need special build config

---

## Phase 2 — Environment setup

Check and install each component. Skip anything already present and working.

### 2.1 — Java (JDK 17)

```bash
# Check
java -version 2>&1 | grep -q "17\." && echo "JDK 17 OK"
```

If missing, install via Homebrew:

```bash
brew install openjdk@17
```

Set `JAVA_HOME`:

```bash
export JAVA_HOME=/opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk/Contents/Home  # Apple Silicon
# or
export JAVA_HOME=/usr/local/opt/openjdk@17/libexec/openjdk.jdk/Contents/Home     # Intel Mac
```

Verify the correct path exists before setting it. Use whichever matches the machine.

### 2.2 — Android SDK

Check for an existing SDK in this order:

1. `$ANDROID_HOME`
2. `$ANDROID_SDK_ROOT`
3. `~/Library/Android/sdk` (Android Studio default)
4. `/opt/homebrew/share/android-commandlinetools` (Homebrew default)

If none found:

```bash
brew install --cask android-commandlinetools
```

Set `ANDROID_HOME` to whichever path has the SDK.

### 2.3 — SDK components

Use `sdkmanager` to install missing components. Read the project's `build.gradle` files
to determine the correct versions. If not specified, use sensible defaults.

Required:

- `platform-tools`
- `platforms;android-{targetSdk}` (read from build.gradle or default to 35)
- `build-tools;{version}` (read from build.gradle or default to 35.0.0)
- `emulator`
- `system-images;android-{targetSdk};google_apis;arm64-v8a`

Conditional (only if project has native C++ code):

- `ndk;{version}` (read version from build.gradle `ndkVersion` or gradle.properties)

Accept licenses automatically:

```bash
yes | sdkmanager --sdk_root="$ANDROID_HOME" <packages>
```

### 2.4 — local.properties

Create or update `android/local.properties`:

```
sdk.dir=<ANDROID_HOME path>
```

### 2.5 — AVD (Android Virtual Device)

Check if an AVD already exists:

```bash
avdmanager list avd
```

If no suitable AVD exists, create one:

```bash
echo "no" | avdmanager create avd \
  -n <ProjectName>_Test \
  -k "system-images;android-{targetSdk};google_apis;arm64-v8a" \
  -d pixel_6 --force
```

Use the project name (from app.json `name` field) in the AVD name.

---

## Phase 3 — Expo prebuild

If `android/` doesn't exist or is empty, run:

```bash
npx expo prebuild --platform android
```

If `android/` exists but looks stale (e.g., package name mismatch), run:

```bash
npx expo prebuild --platform android --clean
```

---

## Phase 4 — Build

Run the Gradle debug build:

```bash
export JAVA_HOME=<detected path>
export ANDROID_HOME=<detected path>
cd android && ./gradlew assembleDebug
```

### Handling build errors

Common issues to watch for (not exhaustive — diagnose what you see):

- **STL mismatch with Oboe or other prefab libraries:**
  Add `arguments "-DANDROID_STL=c++_shared"` to the cmake block in the native module's `build.gradle`.

- **NDK not found:**
  Install the NDK version specified in `build.gradle` or `gradle.properties`.

- **Linker errors for native modules:**
  Check `CMakeLists.txt` for missing source files. Compare with the module's iOS build
  to see if shared C++ sources need to be included.

- **Mismatched API types between native and JS:**
  Compare the native module's return values with the TypeScript type definitions.
  Fix the native side to match the TypeScript contract. (Example: returning `"D2"` for
  a note field when TS expects `"D"` with a separate `octave` field.)

Fix errors, re-run the build, and commit each fix with a descriptive message.

---

## Phase 5 — Build verification

After a successful build, create `scripts/verify-android-build.sh` in the project.
This script must:

1. **Check environment** — JDK, Android SDK, NDK (if needed)
2. **Run Gradle assembleDebug** — fail if build fails
3. **Verify APK exists** — check `android/app/build/outputs/apk/debug/app-debug.apk`
4. **Inspect APK contents** — verify expected native libraries are present

The script must be project-aware. Discover what to verify by reading the project:

- If the project has native C++ modules, verify the `.so` files are in the APK
- Check for all ABIs the project targets (read from build.gradle `abiFilters`)
- Check for any third-party native libs the project depends on

The script must:

- Use `set -euo pipefail`
- Print clear PASS/FAIL for each check
- Exit 0 only if all checks pass
- Exit 1 with details on any failure
- Be runnable standalone: `bash scripts/verify-android-build.sh`

Run the script after creating it to confirm it passes.

---

## Phase 6 — Emulator launch and app verification

### 6.1 — Start emulator

```bash
emulator -avd <AVD_NAME> -gpu swiftshader_indirect -no-boot-anim &
adb wait-for-device
# Wait for full boot
while [ "$(adb shell getprop sys.boot_completed)" != "1" ]; do sleep 2; done
```

### 6.2 — Install and launch

```bash
adb install -r <APK_PATH>
adb reverse tcp:8081 tcp:8081
npx expo start --android
```

### 6.3 — Verify the app runs

- Check logcat for `"Running \"main\""` — confirms the JS bundle loaded
- Check for fatal errors or crashes in logcat
- Take a screenshot: `adb shell screencap -p /sdcard/screenshot.png && adb pull /sdcard/screenshot.png /tmp/screenshot.png`
- View the screenshot to verify UI renders
- Navigate to other screens if possible (use `adb shell input tap X Y`) and screenshot each

### 6.4 — Mic/audio/hardware notes

If the project uses hardware features (microphone, camera, sensors, Bluetooth):

- These likely won't work reliably in the emulator
- Document this clearly — don't waste time debugging emulator hardware limitations
- Flag for on-device testing

---

## Phase 7 — Generate run-android.sh

Create a `run-android.sh` script at the project root. This is for the **user** to invoke
directly — it's separate from the agent test framework.

The script must:

- Set `JAVA_HOME` and `ANDROID_HOME` to the detected paths
- Add platform-tools and emulator to PATH
- Run preflight checks (fail fast with clear messages if JDK or SDK missing)
- Accept a `--rebuild` flag to force a fresh Gradle build
- Skip Gradle build if APK already exists (unless `--rebuild`)
- Start the emulator if not already running (detect via `adb devices`)
- Wait for full boot
- Install the APK
- Forward the Metro port (`adb reverse`)
- Kill any existing Metro on the port
- Start Metro with `npx expo start --android`
- Be executable (`chmod +x`)

Read the project to fill in:

- The correct `JAVA_HOME` path for this machine
- The correct `ANDROID_HOME` path
- The AVD name created in Phase 2
- The APK output path

---

## Phase 8 — Document and commit

Update the project's `progress.md` (or create a section if it doesn't exist) with:

- Environment setup details (paths, versions)
- Build errors encountered and fixes applied
- Emulator/hardware limitations discovered
- What the verification script checks

Commit the `run-android.sh` and `scripts/verify-android-build.sh` scripts.

---

## Commit discipline

- Commit each distinct fix separately with a descriptive message
- Commit the verification script separately
- Commit `run-android.sh` separately
- Commit documentation updates separately
- Never batch unrelated changes into one commit
