# Python to APK (Android) — Ultra-Detailed Beginner Guide

This guide shows two beginner-friendly ways to package a Python app into an Android APK:

- Buildozer + Kivy (most common)
- BeeWare + Briefcase (native toolchains)

Pick one approach and follow it end-to-end.

## When to Use This Format

Use the APK format when you want to distribute your Python app on Android devices. This guide focuses on apps built with a mobile UI framework (Kivy or BeeWare). Pure Python CLI scripts do not run standalone on Android — you need one of the frameworks below.

## Time Estimate

~45 minutes for a first build. The first run downloads Android SDK and NDK components (~1–2 GB), which can take longer on a slow connection. Subsequent builds with caching take ~5 minutes.

> **Device architecture note:** Buildozer and Briefcase build for ARM by default (`armeabi-v7a` and `arm64-v8a`). If you need x86_64 APKs for emulator-only distribution, set `android.archs = x86_64` in `buildozer.spec`. For Play Store distribution, build a Universal APK or an Android App Bundle (AAB) that includes all architectures.

## What You Will Build

- An Android APK you can install on a device or emulator
- A repeatable build process

## Who This Is For

- Beginners packaging a Python app for Android
- Teams who want a simple, step-by-step process

## Important Notes Before You Start

- Android APK builds require Linux tooling. On Windows, use WSL2 or a Linux VM.
- Pure Python CLI apps do not run on Android by themselves. You need a mobile framework.
- Build times are long on the first build because Android SDK/NDK components download.

## Related Guides

- Linux packaging: See [../linux/python-to-deb.md](../linux/python-to-deb.md) or [../linux/python-to-rpm.md](../linux/python-to-rpm.md)

---

## Option A: Buildozer + Kivy (Recommended for Beginners)

### Step 1: Prepare Linux or WSL2

Use Ubuntu 20.04+ (WSL2 is fine). Update your system:

<!-- Ensures the system package list is current before installing dependencies -->
```bash
sudo apt update
sudo apt upgrade -y
```

<!-- TODO: add screenshot: ../../images/android/ubuntu-terminal-updated.png -->

### Step 2: Install dependencies

<!-- Installs Java, build tools, and Python libraries required by Buildozer -->
```bash
sudo apt install -y python3 python3-pip python3-venv git zip unzip openjdk-17-jdk
sudo apt install -y libffi-dev libssl-dev libbz2-dev liblzma-dev libsqlite3-dev
```

Expected result: Java and build tools are installed.

### Step 3: Create and activate a virtual environment

<!-- Isolates the Buildozer installation from the system Python -->
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
```

Expected result: prompt shows `(.venv)`.

### Step 4: Install Buildozer and Cython

<!-- Installs the Buildozer tool and Cython, which is required for compiling Python extensions -->
```bash
pip install buildozer cython
```

### Step 5: Initialize Buildozer

In your project root:

<!-- Generates a buildozer.spec configuration file for your project -->
```bash
buildozer init
```

Expected result: a `buildozer.spec` file appears.

<!-- TODO: add screenshot: ../../images/android/buildozer-spec-in-folder.png -->

### Step 6: Edit buildozer.spec

Open `buildozer.spec` and update these fields:

- `title = MyApp`
- `package.name = myapp`
- `package.domain = org.example`
- `requirements = python3,kivy`
- `source.dir = .`
- `source.include_exts = py,png,jpg,kv,json`

If you use extra libraries, add them to `requirements`.

> **package.domain must be unique.** If you submit to the Play Store, `package.domain` + `package.name` becomes your app's permanent package ID (for example, `org.example.myapp`). Once published, this cannot be changed. Use a domain you own or a reverse-DNS style identifier you control.

### Step 7: Build the APK (debug)

<!-- Downloads Android SDK/NDK on first run, then compiles and packages the APK -->
```bash
buildozer -v android debug
```

Expected result:

- Buildozer downloads Android SDK/NDK on first run
- An APK appears under `bin/`

<!-- TODO: add screenshot: ../../images/android/bin-folder-debug-apk.png -->

### Step 8: Install on a device

Enable Developer Options and USB debugging on your device, then run:

<!-- Pushes the debug APK to a connected device and launches the app -->
```bash
buildozer android deploy run
```

Expected result: app launches on device.

### Step 9: Release build (production)

<!-- Creates an unsigned release APK — sign and zipalign it before distributing -->
```bash
buildozer -v android release
```

Then sign and align it — see the "Signing and Aligning a Release APK" section below.

---

## Option B: BeeWare + Briefcase (Native Toolchain)

### Step 1: Install Briefcase

<!-- Sets up an isolated environment and installs the Briefcase packaging tool -->
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install briefcase
```

### Step 2: Create a Briefcase project

<!-- Runs an interactive wizard to scaffold a new BeeWare project -->
```bash
briefcase new
```

Follow the prompts and choose Android as a target.

<!-- TODO: add screenshot: ../../images/android/briefcase-new-wizard.png -->

### Step 3: Build the Android project

<!-- Compiles the app and sets up the Android Gradle project -->
```bash
briefcase build android
```

### Step 4: Run on device or emulator

<!-- Launches the app on a connected device or running emulator -->
```bash
briefcase run android
```

### Step 5: Create the APK

<!-- Packages the built project into a distributable APK file -->
```bash
briefcase package android
```

Expected result: an APK appears under the `dist/` folder.

---

## Signing and Aligning a Release APK

Before distributing a release APK, you must align and sign it. Both steps are required by the Play Store and by `apksigner`.

### Create a keystore (first time only)

<!-- Generates a new RSA keystore for signing your release APKs -->
```bash
keytool -genkeypair -v \
  -keystore my-release-key.jks \
  -keyalg RSA -keysize 2048 \
  -validity 10000 \
  -alias mykey
```

> **Keep your keystore safe.** If you lose it, you cannot update your app on the Play Store — users will have to uninstall and reinstall from scratch. Back it up to a secure location immediately.

> **Never commit your keystore to git.** Add `*.jks` and `*.keystore` to your `.gitignore`. In CI/CD, store the keystore as a base64-encoded GitHub secret (see CI/CD section below).

### Align and sign

<!-- zipalign must run before apksigner (Play Store requirement) -->
```bash
zipalign -v 4 bin/MyApp-release-unsigned.apk bin/MyApp-release-aligned.apk
```

<!-- Signs the aligned APK with your release keystore -->
```bash
apksigner sign \
  --ks my-release-key.jks \
  --ks-key-alias mykey \
  --out bin/MyApp-release-signed.apk \
  bin/MyApp-release-aligned.apk
```

<!-- Verifies the APK signature is valid -->
```bash
apksigner verify bin/MyApp-release-signed.apk
```

---

## CI/CD with GitHub Actions

Automate APK builds so every push produces a debug APK and every version tag produces a signed release APK.

> **Note on Option B (Briefcase):** The workflow below is for Buildozer + Kivy (Option A). For Briefcase, replace the Buildozer install and build steps with `pip install briefcase` and `briefcase build android && briefcase package android`. The signing and upload steps remain the same.

Create `.github/workflows/build-apk.yml` with this content:

```yaml
name: Build Android APK

on:
  push:
    branches: [main]
    tags: ["v*.*.*"]
  pull_request:
    branches: [main]

jobs:
  build-apk:
    runs-on: ubuntu-22.04

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Java 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "17"

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Set up Android SDK
        uses: android-actions/setup-android@v3

      - name: Install system dependencies for Buildozer
        shell: bash
        run: |
          sudo apt-get update
          sudo apt-get install -y \
            git zip unzip \
            libffi-dev libssl-dev libbz2-dev liblzma-dev libsqlite3-dev

      - name: Create virtual environment and install Buildozer
        shell: bash
        run: |
          python -m venv .venv
          source .venv/bin/activate
          pip install --upgrade pip
          pip install buildozer cython

      - name: Cache Buildozer directory
        uses: actions/cache@v4
        with:
          path: ~/.buildozer
          # Cache key changes when buildozer.spec changes, forcing a fresh SDK/NDK download
          # only when the spec is updated — cuts subsequent build times from ~30 to ~5 min
          key: buildozer-${{ runner.os }}-${{ hashFiles('buildozer.spec') }}
          restore-keys: |
            buildozer-${{ runner.os }}-

      - name: Extract version from git tag
        id: version
        shell: bash
        run: |
          if [[ "${GITHUB_REF}" == refs/tags/v* ]]; then
            echo "VERSION=${GITHUB_REF_NAME#v}" >> "$GITHUB_OUTPUT"
          else
            echo "VERSION=1.0.0-dev" >> "$GITHUB_OUTPUT"
          fi

      - name: Inject version into buildozer.spec
        shell: bash
        run: |
          VERSION="${{ steps.version.outputs.VERSION }}"
          sed -i "s/^version = .*/version = ${VERSION}/" buildozer.spec
          # versionCode must be a monotonically increasing integer; use the run number
          sed -i "s/^# android.numeric_version.*/android.numeric_version = ${{ github.run_number }}/" buildozer.spec

      - name: Build debug APK (branches and pull requests)
        if: "!startsWith(github.ref, 'refs/tags/')"
        shell: bash
        run: |
          source .venv/bin/activate
          buildozer -v android debug

      - name: Build release APK (tag pushes only)
        if: startsWith(github.ref, 'refs/tags/')
        shell: bash
        run: |
          source .venv/bin/activate
          buildozer -v android release

      - name: Sign release APK (tag pushes only)
        if: startsWith(github.ref, 'refs/tags/')
        shell: bash
        env:
          KEYSTORE_BASE64: ${{ secrets.KEYSTORE_BASE64 }}
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
        run: |
          # Decode the base64-encoded keystore stored in GitHub Secrets
          echo "$KEYSTORE_BASE64" | base64 --decode > my-release-key.jks
          # zipalign must run before signing (required by Play Store and apksigner)
          zipalign -v 4 \
            bin/MyApp-*-release-unsigned.apk \
            bin/MyApp-release-aligned.apk
          apksigner sign \
            --ks my-release-key.jks \
            --ks-key-alias "$KEY_ALIAS" \
            --ks-pass "pass:$KEYSTORE_PASSWORD" \
            --key-pass "pass:$KEY_PASSWORD" \
            --out bin/MyApp-release-signed.apk \
            bin/MyApp-release-aligned.apk
          apksigner verify bin/MyApp-release-signed.apk
          # Remove the decoded keystore from disk immediately
          rm my-release-key.jks

      - name: Upload debug APK artifact (branches and pull requests)
        if: "!startsWith(github.ref, 'refs/tags/')"
        uses: actions/upload-artifact@v4
        with:
          name: MyApp-apk-${{ github.ref_name }}
          path: bin/*.apk
          retention-days: 30

      - name: Upload signed release APK artifact (tag pushes only)
        if: startsWith(github.ref, 'refs/tags/')
        uses: actions/upload-artifact@v4
        with:
          name: MyApp-apk-${{ github.ref_name }}
          path: bin/MyApp-release-signed.apk
          retention-days: 30
```

### Setting Up Android Signing Secrets

Add these four secrets to your GitHub repository under Settings → Secrets and variables → Actions:

| Secret name | Value |
|---|---|
| `KEYSTORE_BASE64` | Your `.jks` file encoded as base64 |
| `KEYSTORE_PASSWORD` | The keystore password |
| `KEY_ALIAS` | The key alias inside the keystore |
| `KEY_PASSWORD` | The key password (often the same as keystore password) |

Encode your keystore file on Linux:

```bash
base64 -w 0 my-release-key.jks | xclip -selection clipboard
```

On macOS:

```bash
base64 -i my-release-key.jks | pbcopy
```

> **Play Store note:** For Play Store submissions, Google recommends Android App Bundles (`.aab`) over APKs. Briefcase supports AABs via `briefcase build android` followed by `./gradlew bundleRelease` in the generated Android project folder.

## Production Checklist (Both Options)

- Build on Linux or a CI runner with Android SDK installed.
- Use a release keystore and sign the APK with `apksigner`.
- Align the APK with `zipalign` before signing.
- Test on a real device and at least one emulator.
- Keep `versionName` and `versionCode` updated in your config.

## Common Issues and Fixes

- Build fails on Windows: use WSL2 or a Linux machine.
- Missing SDK/NDK: let Buildozer or Briefcase download them, or install manually.
- App crashes on start: confirm all requirements and permissions are declared.
- Black screen in Kivy: ensure your `.kv` files are listed in `source.include_exts`.

## Final Checklist

- APK builds without errors
- APK installs on a device
- App launches and runs basic flows
- Release APK is signed and aligned
- Keystore backed up to a secure location
- `*.jks` added to `.gitignore`
- GitHub Actions workflow committed to `.github/workflows/`
- Required secrets added to GitHub repository settings
- Build passes end-to-end on a tag push
- Signed artifact downloaded from Actions and tested on a device
- Screenshot placeholders replaced with real screenshots in `images/android/`
