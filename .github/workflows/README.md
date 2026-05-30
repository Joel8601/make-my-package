# GitHub Actions CI/CD Workflows

This folder contains six automated GitHub Actions workflows designed to package your Python application into distributable native formats for Windows, Linux, and Android.

## Workflow Summary

| Workflow File | Target Format | Runner OS | Build Toolchain | Release Output | Setup Steps |
|---|---|---|---|---|---|
| [`build-exe.yml`](build-exe.yml) | Windows Standalone EXE | `windows-latest` | PyInstaller | `dist/MyApp/` folder | [Configure Steps ➔](#step-2-customize-the-yaml-files-for-your-project) |
| [`build-msi.yml`](build-msi.yml) | Windows MSI Installer | `windows-latest` | WiX Toolset v3 + PyInstaller | `MyApp.msi` installer | [Configure Steps ➔](#step-2-customize-the-yaml-files-for-your-project) |
| [`build-msix.yml`](build-msix.yml) | Windows MSIX Package | `windows-latest` | makeappx + PyInstaller | `MyApp.msix` package | [Configure Steps ➔](#step-2-customize-the-yaml-files-for-your-project) |
| [`build-deb.yml`](build-deb.yml) | Debian/Ubuntu DEB | `ubuntu-22.04` | dpkg-buildpackage + lintian | `*.deb` package | [Configure Steps ➔](#step-2-customize-the-yaml-files-for-your-project) |
| [`build-rpm.yml`](build-rpm.yml) | RHEL/Fedora/CentOS RPM | `ubuntu-22.04` (Fedora container) | rpmbuild + rpmlint | `*.rpm` package | [Configure Steps ➔](#step-2-customize-the-yaml-files-for-your-project) |
| [`build-apk.yml`](build-apk.yml) | Android APK | `ubuntu-22.04` | Buildozer + JDK 17 | Signed/aligned release APK | [Configure Steps ➔](#step-2-customize-the-yaml-files-for-your-project) |


---

## How It Works

### Triggers
Each workflow is configured to run automatically:
1. **Pull Requests / Pushes to `main`**: Runs the full build and package pipeline, linting, and outputs an **unsigned/debug developer build**.
2. **Version Tag Pushes (`v*.*.*`)**: Triggers the **production release pipeline** which decodes code-signing certificates/keystores from GitHub Secrets, signs/aligns the packages, and uploads production-ready assets.

### Dynamic Versioning
- Workflows dynamically parse your Git tag name (e.g., `v1.2.3` becomes version `1.2.3` or `1.2.3.0` for MSIX).
- On development branch builds, a fallback version (e.g., `0.0.0-dev` or `1.0.0-dev`) is used.

---

## Required Secrets Setup

To enable automated production signing on version tags, add the following secrets to your GitHub repository under **Settings → Secrets and variables → Actions**:

### Windows Signing Secrets (EXE, MSI, MSIX)

These secrets are shared by the three Windows pipelines:

| Secret Name | Value Description |
|---|---|
| `WINDOWS_CERTIFICATE_BASE64` | Your `.pfx` code-signing certificate file encoded as a Base64 string. |
| `WINDOWS_CERTIFICATE_PASSWORD` | The password required to unlock and decode the `.pfx` certificate file. |

*To generate the Base64 value of your certificate:*
- **Linux/macOS**: `base64 -i my-certificate.pfx` (or use `-w 0` on Linux to strip newlines).
- **Windows (PowerShell)**: `[Convert]::ToBase64String([IO.File]::ReadAllBytes("my-certificate.pfx"))`

---

### Android Signing Secrets (APK)

These secrets are used by the Android pipeline to align and sign release APKs:

| Secret Name | Value Description |
|---|---|
| `KEYSTORE_BASE64` | Your `.jks` or `.keystore` signing key file encoded as a Base64 string. |
| `KEYSTORE_PASSWORD` | The master password to access the keystore file. |
| `KEY_ALIAS` | The alias given to the private key when created. |
| `KEY_PASSWORD` | The password for the specific private key alias. |

*To generate the Base64 value of your keystore:*
- **Linux/macOS**: `base64 -i my-release-key.jks`
- **Windows (PowerShell)**: `[Convert]::ToBase64String([IO.File]::ReadAllBytes("my-release-key.jks"))`

---

## How to Set Up and Adapt Workflows for Your Project

To configure automated packaging for a new or existing Python repository, follow this step-by-step setup guide:

### Step 1: Place files in your repository
Ensure your project contains:
1. The `.github/workflows/` directory containing the `.yml` files of the formats you want to support.
2. A `.gitignore` file that excludes local build outputs (`dist/`, `build/`, `*.spec`, `*.jks`, `*.pfx`, `*.log`).

### Step 2: Customize the YAML files for your project
Open the YAML workflow files and customize the following values to match your application:

1. **Application Name and Executable Path**:
   - In `build-exe.yml`, `build-msi.yml`, and `build-msix.yml`, search for `MyApp` and replace it with your application's actual name.
   - For example, if your entry script is `run.py` instead of `main.py`, update the PyInstaller step to:
     ```yaml
     pyinstaller --name MyCoolApp --noconfirm .\run.py
     ```
2. **Spec File References**:
   - If you use a custom PyInstaller Spec file, replace references to `MyApp.spec` with `MyCoolApp.spec`.
3. **Paths to Signed Artifacts**:
   - Update the artifact upload and signing paths from `dist/MyApp/` to your custom output directory (e.g., `dist/MyCoolApp/`).
4. **Android Package Details**:
   - In `build-apk.yml`, update references to `buildozer.spec` or replace the Buildozer commands with your specific target framework details (like Briefcase CLI paths).

### Step 3: Configure GitHub Secrets
1. Go to your repository on GitHub.
2. Navigate to **Settings** → **Secrets and variables** → **Actions**.
3. Click **New repository secret** and add the variables listed in the **Required Secrets Setup** section above (e.g., `WINDOWS_CERTIFICATE_BASE64` or `KEYSTORE_BASE64`).

### Step 4: Run a Test Build
1. Commit the changes and push them to your repository on any development branch or submit a Pull Request.
2. Go to the **Actions** tab in your GitHub repository.
3. Select your workflow from the list to watch the runner set up, compile dependencies, bundle your Python app, and upload an **unsigned test artifact**.

### Step 5: Publish a Signed Release
Once you are happy with the test build:
1. Tag your commit with a version number:
   ```bash
   git tag v1.0.0
   ```
2. Push the tag to GitHub:
   ```bash
   git push origin v1.0.0
   ```
3. GitHub Actions will detect the tag trigger, fetch the signing secrets, sign/align your packages, and upload **production-ready signed installers**.

---

## Security Best Practices

1. **Ephemeral Decoding**: Workflows decode certificate and keystore files directly to disk inside the job's sandbox workspace, use them immediately for signing, and run `rm` to securely purge them from the runner environment before completion.
2. **Pull Request Isolation**: Code signing steps contain explicit conditional triggers (`if: startsWith(github.ref, 'refs/tags/')`) so that pull requests from external forks or branch pushes never run the signing steps or expose sensitive secrets.
3. **Artifact Retention**: Artifacts uploaded to GitHub Actions are set with a retention window of 30 days to optimize storage limits.

