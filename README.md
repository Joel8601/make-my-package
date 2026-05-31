# Make-my-package

This repository contains ultra-detailed, beginner-friendly guides and automated CI/CD pipelines for packaging applications into common distribution formats on Windows, Linux, and Android.

Star it ! If you LIKE IT!

## Quick Start

Start with the Python guide folder:

- [Python/README.md](Python/README.md)

---

## What Is Inside

### 1. Beginner-Friendly Guides
- **Windows Packaging**: [EXE (PyInstaller)](Python/windows/python-to-exe.md), [MSI (WiX Toolset)](Python/windows/python-to-msi.md), and [MSIX (makeappx)](Python/windows/python-to-msix.md)
- **Linux Packaging**: [DEB (Debian/Ubuntu)](Python/linux/python-to-deb.md) and [RPM (Fedora/RHEL/CentOS)](Python/linux/python-to-rpm.md)
- **Android Packaging**: [APK (Kivy/Buildozer & BeeWare/Briefcase)](Python/android/python-to-apk.md)

### 2. Automated CI/CD Workflows (`.github/workflows/`)
Each guide is backed by a fully-tested GitHub Actions workflow that automates your builds:
- **Windows**: [Build Standalone EXE](.github/workflows/build-exe.yml), [Build MSI Installer](.github/workflows/build-msi.yml), and [Build MSIX Package](.github/workflows/build-msix.yml)
- **Linux**: [Build Debian DEB](.github/workflows/build-deb.yml) and [Build Red Hat RPM](.github/workflows/build-rpm.yml)
- **Android**: [Build Mobile APK](.github/workflows/build-apk.yml)

For full configuration steps, triggers, and signing secrets setup, see the [Workflows Documentation](.github/workflows/README.md).

---

## How It Works (High-Level)

1. **Build**: Bundle your app's Python code and dependencies into executable files using PyInstaller, Buildozer, or Briefcase.
2. **Package**: Wrap that output into native system installers (MSI, MSIX, DEB, RPM, APK).
3. **Automate**: Commit your code and let GitHub Actions run the build pipelines on branch pushes and pull requests.
4. **Sign & Release**: Push a version tag (`v*.*.*`) to automatically trigger production-level code signing with repository secrets and generate download-ready assets.

---

## Folder Layout

```text
.github/
  workflows/
    README.md          # Workflows setup and signing secrets guide
    build-exe.yml      # Windows EXE pipeline
    build-msi.yml      # Windows MSI pipeline
    build-msix.yml     # Windows MSIX pipeline
    build-deb.yml      # Linux DEB pipeline
    build-rpm.yml      # Linux RPM pipeline
    build-apk.yml      # Android APK pipeline
Python/
  README.md            # Python guides index
  windows/
    python-to-exe.md   # EXE packaging guide
    python-to-msi.md   # MSI packaging guide
    python-to-msix.md  # MSIX packaging guide
  linux/
    python-to-deb.md   # DEB packaging guide
    python-to-rpm.md   # RPM packaging guide
  android/
    python-to-apk.md   # APK packaging guide
images/                # Documentation screenshots and assets
```
