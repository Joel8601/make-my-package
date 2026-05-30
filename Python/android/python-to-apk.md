# Python to APK (Android) - Ultra-Detailed Beginner Guide

This guide shows two beginner-friendly ways to package a Python app into an Android APK:

- Buildozer + Kivy (most common)
- BeeWare + Briefcase (native toolchains)

Pick one approach and follow it end-to-end.

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

## Option A: Buildozer + Kivy (Recommended for Beginners)

### Step 1: Prepare Linux or WSL2

Use Ubuntu 20.04+ (WSL2 is OK). Update your system:

```bash
sudo apt update
sudo apt upgrade -y
```

Screenshot placeholder: Ubuntu terminal updated

### Step 2: Install dependencies

```bash
sudo apt install -y python3 python3-pip python3-venv git zip unzip openjdk-17-jdk
sudo apt install -y libffi-dev libssl-dev libbz2-dev liblzma-dev libsqlite3-dev
```

Expected result: Java and build tools are installed.

### Step 3: Create and activate a virtual environment

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
```

Expected result: prompt shows `(.venv)`.

### Step 4: Install Buildozer and Cython

```bash
pip install buildozer cython
```

### Step 5: Initialize Buildozer

In your project root:

```bash
buildozer init
```

Expected result: a `buildozer.spec` file appears.

Screenshot placeholder: buildozer.spec in project folder

### Step 6: Edit buildozer.spec

Open `buildozer.spec` and update these fields:

- `title = MyApp`
- `package.name = myapp`
- `package.domain = org.example`
- `requirements = python3,kivy`
- `source.dir = .`
- `source.include_exts = py,png,jpg,kv,json`

If you use extra libraries, add them to `requirements`.

### Step 7: Build the APK (debug)

```bash
buildozer -v android debug
```

Expected result:

- Buildozer downloads Android SDK/NDK
- An APK appears under `bin/`

Screenshot placeholder: bin folder showing debug APK

### Step 8: Install on a device

Enable Developer Options and USB debugging, then run:

```bash
buildozer android deploy run
```

Expected result: app launches on device.

### Step 9: Release build (production)

Create a release APK:

```bash
buildozer -v android release
```

Then sign and align it using Android tools (see Production Checklist).

## Option B: BeeWare + Briefcase (Native Toolchain)

### Step 1: Install Briefcase

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install briefcase
```

### Step 2: Create a Briefcase project

```bash
briefcase new
```

Follow the prompts and choose Android as a target.

Screenshot placeholder: briefcase new wizard

### Step 3: Build the Android project

```bash
briefcase build android
```

### Step 4: Run on device or emulator

```bash
briefcase run android
```

### Step 5: Create the APK

```bash
briefcase package android
```

Expected result: an APK appears under the `dist/` folder.

## Production Checklist (Both Options)

- Build on Linux or CI runner with Android SDK installed.
- Use a release keystore and sign the APK.
- Align the APK (`zipalign`) before distribution.
- Test on a real device and at least one emulator.
- Keep versionName and versionCode updated.

## Common Issues and Fixes

- Build fails on Windows: use WSL2 or Linux.
- Missing SDK/NDK: let Buildozer/Briefcase download, or install manually.
- App crashes on start: confirm requirements and permissions.
- Black screen in Kivy: ensure your `.kv` files are included.

## Final Checklist

- APK builds without errors
- APK installs on a device
- App launches and runs basic flows
- Release APK is signed and aligned
