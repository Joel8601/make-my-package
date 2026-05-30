# Python Packaging Guides

This folder contains ultra-detailed, beginner-friendly packaging guides for Windows and Linux. Use these guides to turn a Python app into installable formats like EXE, MSI, MSIX, DEB, and RPM.

## How to Use

1. Pick your target OS folder.
2. Follow the guides in order (for example, EXE before MSI).
3. Use the checklists at the end of each guide to confirm release readiness.

## Windows Guides (Packaging Order)

- Start here: [Python/windows/python-to-exe.md](windows/python-to-exe.md)
- MSI installer: [Python/windows/python-to-msi.md](windows/python-to-msi.md)
- MSIX package: [Python/windows/python-to-msix.md](windows/python-to-msix.md)

## Linux Guides

- Debian/Ubuntu: [Python/linux/python-to-deb.md](linux/python-to-deb.md)
- RHEL/Fedora/CentOS: [Python/linux/python-to-rpm.md](linux/python-to-rpm.md)

## How It Works (High-Level)

- Build step: Convert your Python app into a distributable output (usually a folder-based build).
- Packaging step: Wrap that output into an installer or package format.
- Validation step: Install on a clean machine, run tests, and fix any missing files.
- Release step: Sign artifacts and publish with versioned names and hashes.

## Tips

- Use a clean virtual environment for builds.
- Prefer folder-based builds for installers (MSI/MSIX).
- Keep a consistent versioning scheme across releases.
