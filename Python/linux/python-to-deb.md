# Python to DEB (Debian/Ubuntu) — Ultra-Detailed Beginner Guide

This guide shows how to create a `.deb` package the standard Debian way. It is written for beginners and uses simple files and steps.

## When to Use This Format

Use the DEB format when you are distributing to Debian or Ubuntu users and want them to install your app with `apt` or `dpkg`. It is the standard format for these distributions and supports clean uninstall and dependency management.

## Time Estimate

~20 minutes for a first build once your app output is ready.

> **ARM64 note:** This guide produces x86_64 packages only. To build for ARM64 (aarch64) Linux — common on cloud VMs, Raspberry Pi, and Apple Silicon in Rosetta-free mode — you must build on an ARM64 machine or use QEMU emulation. PyInstaller cannot cross-compile: x86_64 to ARM64 is not supported. GitHub Actions offers hosted ARM64 Linux runners (`ubuntu-24.04-arm`) for this purpose. Change `Architecture: amd64` to `arm64` in `debian/control` when targeting ARM64.

## Related Guides

- RPM alternative for RHEL/Fedora: See [python-to-rpm.md](python-to-rpm.md)
- Windows EXE guide (for reference): See [../windows/python-to-exe.md](../windows/python-to-exe.md)

## What You Will Build

- A `.deb` package that installs your app under `/usr/lib/myapp`
- A package that can be installed with `apt` or `dpkg`

## Who This Is For

- Beginners who want Linux packaging done correctly
- Teams distributing internal packages

## Prerequisites

- Debian or Ubuntu (or WSL)
- Your app already runs with `python main.py`
- Basic terminal usage

## Step 1: Install packaging tools

<!-- Installs the Debian build tools needed to create and validate .deb packages -->
```bash
sudo apt update
sudo apt install -y build-essential debhelper devscripts dh-python lintian
```

Expected result: tools like `dpkg-buildpackage` and `lintian` are available.

## Step 2: Create Debian packaging folder

In your project root, create `debian/` with these files:

```text
debian/
   control
   rules
   install
   changelog
```

<!-- TODO: add screenshot: ../../images/linux/debian-folder-structure.png -->

## Step 3: Fill in `debian/control`

```text
Source: myapp
Section: utils
Priority: optional
Maintainer: Your Team <dev@example.com>
Build-Depends: debhelper-compat (= 13), dh-python, python3
Standards-Version: 4.6.2

Package: myapp
Architecture: amd64
Depends: ${misc:Depends}, ${python3:Depends}
Description: MyApp description
```

Explanation:

- `Package` is the install name
- `Architecture` should match your build machine

## Step 4: Fill in `debian/rules`

The `rules` file is a Makefile that debhelper runs. For a PyInstaller build, you must override the default build and install steps — debhelper's defaults expect a setuptools project and will produce an empty package otherwise.

```makefile
#!/usr/bin/make -f
%:
	dh $@

override_dh_auto_build:
	python3 -m venv .venv
	. .venv/bin/activate && pip install -r requirements.txt
	. .venv/bin/activate && pip install pyinstaller
	. .venv/bin/activate && pyinstaller --name MyApp ./main.py

override_dh_auto_install:
	mkdir -p debian/myapp/usr/lib/myapp
	cp -a dist/MyApp/* debian/myapp/usr/lib/myapp/
	mkdir -p debian/myapp/usr/bin
	ln -s /usr/lib/myapp/MyApp debian/myapp/usr/bin/myapp
```

> **Why override_dh_auto_build and override_dh_auto_install:** Debhelper's default build step expects a standard Python setuptools project. Since we use PyInstaller, we override both steps: `override_dh_auto_build` runs PyInstaller, and `override_dh_auto_install` copies the output into the staging directory debhelper uses to assemble the `.deb`. Without these overrides, `dpkg-buildpackage` will produce an empty package.

> **Entry point:** The `ln -s` line in `override_dh_auto_install` creates a `/usr/bin/myapp` symlink so users can type `myapp` in any terminal. If you do not want a command entry point, remove both the `mkdir -p` and `ln -s` lines referencing `/usr/bin`.

Make the rules file executable:

<!-- Makes the rules file executable so dpkg-buildpackage can run it -->
```bash
chmod +x debian/rules
```

## Step 5: Note on `debian/install`

> **Note:** If you use the `debian/rules` override approach above (recommended for PyInstaller builds), you do not need a `debian/install` file — the `override_dh_auto_install` target handles file placement directly. The `debian/install` approach works only for standard Python packages installed via setuptools. Remove `debian/install` if you are following this guide.

## Step 6: Fill in `debian/changelog`

```text
myapp (1.0.0-1) stable; urgency=medium

   * Initial release

 -- Your Team <dev@example.com>  Sat, 30 May 2026 10:00:00 +0000
```

## Step 7: Build the package

> **Note:** With the corrected `debian/rules` above, you do not need to run PyInstaller manually before this step. The build runs automatically as part of `dpkg-buildpackage` via `override_dh_auto_build`. If you prefer to build manually first (for faster iteration during debugging), you can still run the PyInstaller commands separately and comment out `override_dh_auto_build` in `debian/rules`.

<!-- Builds the .deb package — override_dh_auto_build inside debian/rules runs PyInstaller automatically -->
```bash
dpkg-buildpackage -us -uc
```

Expected output: the `.deb` file appears one directory above your project root.

<!-- TODO: add screenshot: ../../images/linux/deb-file-in-parent-folder.png -->

## Step 8: Validate with lintian

<!-- Checks the .deb package for common policy errors and warnings -->
```bash
lintian ../myapp_1.0.0-1_amd64.deb
```

Fix any warnings before release.

## Step 9: Install and test

<!-- Installs the .deb package locally to verify it works correctly -->
```bash
sudo dpkg -i ../myapp_1.0.0-1_amd64.deb
```

Test your app and then uninstall:

<!-- Removes the installed package cleanly -->
```bash
sudo dpkg -r myapp
```

## CI/CD with GitHub Actions

Automate `.deb` builds so every version tag produces a validated package ready to distribute.

Create `.github/workflows/build-deb.yml` with this content:

```yaml
name: Build Linux DEB

on:
  push:
    branches: [main]
    tags: ["v*.*.*"]
  pull_request:
    branches: [main]

jobs:
  build-deb:
    runs-on: ubuntu-22.04

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install Debian packaging tools
        shell: bash
        run: |
          sudo apt-get update
          sudo apt-get install -y \
            build-essential debhelper devscripts dh-python lintian

      - name: Extract version from git tag
        id: version
        shell: bash
        run: |
          # Debian version format: MAJOR.MINOR.PATCH (the "-1" revision is appended by dch)
          if [[ "${GITHUB_REF}" == refs/tags/v* ]]; then
            echo "VERSION=${GITHUB_REF_NAME#v}" >> "$GITHUB_OUTPUT"
          else
            echo "VERSION=1.0.0-dev" >> "$GITHUB_OUTPUT"
          fi

      - name: Inject version into debian/changelog
        shell: bash
        run: |
          VERSION="${{ steps.version.outputs.VERSION }}"
          # dch is used instead of editing changelog directly because
          # dpkg-parsechangelog requires exact date and format that dch guarantees
          dch --newversion "${VERSION}-1" \
            --distribution stable \
            --force-distribution \
            "Automated build from tag ${{ github.ref_name }}"

      - name: Build .deb package
        shell: bash
        run: |
          # -b builds binary package only; source package requires GPG signing setup
          # override_dh_auto_build in debian/rules runs PyInstaller automatically
          dpkg-buildpackage -us -uc -b

      - name: Validate with lintian
        shell: bash
        run: |
          # --fail-on error makes the job fail on lintian errors but allows warnings
          lintian --fail-on error ../*.deb

      - name: Upload .deb artifact
        uses: actions/upload-artifact@v4
        with:
          # dpkg-buildpackage places the .deb one directory above the project root by default
          name: MyApp-deb-${{ github.ref_name }}
          path: ../*.deb
          retention-days: 30
```

> **Note on GPG signing:** The workflow above produces an unsigned `.deb`. For hosting in an `apt` repository, use `reprepro` or `aptly` to handle repository-level GPG signing separately. Signing individual `.deb` files (not the repository) is uncommon in modern Debian packaging.

> **Note on `../*.deb` path:** The `.deb` file appears one level above your project root because `dpkg-buildpackage` works from the parent directory. This is standard behavior, not a CI quirk.

## Common Issues and Fixes

- Build fails: missing Build-Depends in `debian/control`
- Empty package: `debian/rules` is missing `override_dh_auto_install`
- Lintian warnings: follow Debian policy and fix each warning
- `dpkg-buildpackage` not found: install the `build-essential` package

## Final Checklist

- `debian/` folder exists and is correct
- `debian/rules` includes `override_dh_auto_build` and `override_dh_auto_install`
- `.deb` builds without errors
- Lintian warnings addressed
- Install and uninstall works on a clean machine
- GitHub Actions workflow committed to `.github/workflows/`
- Required secrets added to GitHub repository settings
- Build passes end-to-end on a tag push
- Signed artifact downloaded from Actions and tested on a clean machine
- Screenshot placeholders replaced with real screenshots in `images/linux/`
