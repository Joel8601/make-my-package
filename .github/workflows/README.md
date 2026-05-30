# GitHub Actions CI/CD Workflows

This folder contains six automated GitHub Actions workflows designed to package your Python application into distributable native formats for Windows, Linux, and Android.

## Workflow Summary

| Workflow File | Target Format | Runner OS | Build Toolchain | Release Output |
|---|---|---|---|---|
| [`build-exe.yml`](build-exe.yml) | Windows Standalone EXE | `windows-latest` | PyInstaller | `dist/MyApp/` folder |
| [`build-msi.yml`](build-msi.yml) | Windows MSI Installer | `windows-latest` | WiX Toolset v3 + PyInstaller | `MyApp.msi` installer |
| [`build-msix.yml`](build-msix.yml) | Windows MSIX Package | `windows-latest` | makeappx + PyInstaller | `MyApp.msix` package |
| [`build-deb.yml`](build-deb.yml) | Debian/Ubuntu DEB | `ubuntu-22.04` | dpkg-buildpackage + lintian | `*.deb` package |
| [`build-rpm.yml`](build-rpm.yml) | RHEL/Fedora/CentOS RPM | `ubuntu-22.04` (Fedora container) | rpmbuild + rpmlint | `*.rpm` package |
| [`build-apk.yml`](build-apk.yml) | Android APK | `ubuntu-22.04` | Buildozer + JDK 17 | Signed/aligned release APK |

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

## Security Best Practices

1. **Ephemeral Decoding**: Workflows decode certificate and keystore files directly to disk inside the job's sandbox workspace, use them immediately for signing, and run `rm` to securely purge them from the runner environment before completion.
2. **Pull Request Isolation**: Code signing steps contain explicit conditional triggers (`if: startsWith(github.ref, 'refs/tags/')`) so that pull requests from external forks or branch pushes never run the signing steps or expose sensitive secrets.
3. **Artifact Retention**: Artifacts uploaded to GitHub Actions are set with a retention window of 30 days to optimize storage limits.
