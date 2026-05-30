# Python Packaging Guides

This folder contains ultra-detailed, beginner-friendly packaging guides for Windows, Linux, and Android. Use these guides to turn a Python app into installable formats like EXE, MSI, MSIX, DEB, RPM, and APK.

## How to Use

1. **Pick your target OS**: Choose the folder corresponding to your target platform.
2. **Follow guides in order**: For example, on Windows, complete the EXE guide before moving to MSI or MSIX.
3. **Automate with CI/CD**: Once your local builds work, commit the pre-configured [GitHub Actions Workflows](../.github/workflows/README.md) to automate the packaging, validation, and signing steps on every push and version release.
4. **Use the checklists**: Verify final release readiness using the checklists at the end of each guide.

---

## Windows Guides (Packaging Order)

- **Step 1: Standalone Executable** — [python-to-exe.md](windows/python-to-exe.md)
- **Step 2 (Option A): WiX MSI Installer** — [python-to-msi.md](windows/python-to-msi.md)
- **Step 2 (Option B): Modern MSIX Package** — [python-to-msix.md](windows/python-to-msix.md)

---

## Linux Guides

- **Debian / Ubuntu (DEB)** — [python-to-deb.md](linux/python-to-deb.md)
- **Red Hat / Fedora / CentOS (RPM)** — [python-to-rpm.md](linux/python-to-rpm.md)

---

## Android Guides

- **Mobile APK (Kivy/Briefcase)** — [python-to-apk.md](android/python-to-apk.md)

---

## How It Works (High-Level)

- **Build**: Convert your Python code into an executable output (usually a folder-based build).
- **Package**: Wrap that output into native platform formats (installers or packages).
- **Validate**: Install on a clean machine, run tests, and check that all dynamic dependencies are loaded.
- **Sign & Release**: Code-sign your package with a valid developer certificate or key to build system trust and bypass browser/OS security warnings.
