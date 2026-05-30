# Python to RPM (RHEL/Fedora/CentOS) — Ultra-Detailed Beginner Guide

This guide shows how to build a `.rpm` package using a spec file. It is written for beginners and includes step-by-step explanations.

## When to Use This Format

Use the RPM format when you are distributing to Red Hat, Fedora, CentOS, Rocky Linux, or AlmaLinux users. It is the standard package format for those distributions and integrates with `dnf` and `yum` for dependency management and clean uninstall.

## Time Estimate

~20 minutes for a first build once your app output is ready.

> **ARM64 note:** This guide produces x86_64 packages only. To build for ARM64 (aarch64) Linux — common on cloud VMs, Raspberry Pi, and Apple Silicon in Rosetta-free mode — you must build on an ARM64 machine or use QEMU emulation. PyInstaller cannot cross-compile: x86_64 to ARM64 is not supported. GitHub Actions offers hosted ARM64 Linux runners (`ubuntu-24.04-arm`) for this purpose. Change `BuildArch: x86_64` to `BuildArch: aarch64` in the spec file when targeting ARM64.

## Related Guides

- DEB alternative for Debian/Ubuntu: See [python-to-deb.md](python-to-deb.md)
- Windows EXE guide (for reference): See [../windows/python-to-exe.md](../windows/python-to-exe.md)

## What You Will Build

- An `.rpm` package that installs your app under `/usr/lib/myapp`
- A command entry point `myapp` in `/usr/bin`

## Who This Is For

- Beginners packaging for Fedora/RHEL/CentOS/Rocky
- Teams distributing internal RPMs

## Prerequisites

- A Linux machine with `dnf`
- Your app already runs with `python main.py`

## Step 1: Install RPM tools

<!-- Installs the RPM build tools and the linter needed to create and validate .rpm packages -->
```bash
sudo dnf install -y rpm-build rpmdevtools rpmlint gcc make
```

> **Why rpmdevtools:** The `rpmdev-setuptree` command in Step 2 comes from the `rpmdevtools` package, not `rpm-build`. Without it, Step 2 will fail with "command not found".

Expected result: `rpmbuild` and `rpmlint` are available.

## Step 2: Create the RPM build tree

<!-- Sets up the standard ~/rpmbuild directory tree that rpmbuild expects -->
```bash
rpmdev-setuptree
```

Expected result: `~/rpmbuild/` folder exists with subdirectories: `BUILD`, `RPMS`, `SOURCES`, `SPECS`, `SRPMS`.

<!-- TODO: add screenshot: ../../images/linux/rpmbuild-folder-tree.png -->

## Step 3: Build your app output

Use PyInstaller to generate a folder-based build:

<!-- Creates a virtual environment, installs dependencies, and builds the app binary -->
```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
pip install pyinstaller
pyinstaller --name MyApp ./main.py
```

Expected output:

- `dist/MyApp/` with the binary and supporting files

> **Note:** On Linux, PyInstaller produces a binary named `MyApp` (no `.exe` extension).

## Step 4: Create a spec file

Create `packaging/rpm/myapp.spec` with this content:

```text
Name:           myapp
Version:        1.0.0
Release:        1%{?dist}
Summary:        MyApp description

License:        Proprietary
BuildArch:      x86_64

# project_dir captures the current directory at spec-parse time for local builds.
# In CI/CD (see below), PROJECT_DIR is set as an environment variable instead,
# and %{getenv:PROJECT_DIR} is used to resolve it reliably across build environments.
%global project_dir %(pwd)

%description
MyApp description

%prep

%build

%install
mkdir -p %{buildroot}/usr/lib/myapp
cp -a %{project_dir}/dist/MyApp/* %{buildroot}/usr/lib/myapp/
mkdir -p %{buildroot}/usr/bin
ln -s /usr/lib/myapp/MyApp %{buildroot}/usr/bin/myapp

%files
%dir /usr/lib/myapp
/usr/lib/myapp/*
/usr/bin/myapp

%changelog
* Sat May 30 2026 Your Team <dev@example.com> - 1.0.0-1
- Initial release
```

Explanation:

- `%install` copies your build output into the RPM buildroot staging directory
- `%files` defines what gets installed on the user's machine
- The `ln -s` line creates a `/usr/bin/myapp` shortcut so users can type `myapp` in any terminal

> **Why %{buildroot}:** Every path in `%install` must be prefixed with `%{buildroot}`. This is a staging directory — rpmbuild assembles the package from it. Writing to `/usr/bin/myapp` directly (without `%{buildroot}`) would modify your build machine and the RPM would be empty or incorrect.

> **Why %global project_dir:** rpmbuild changes directory to `~/rpmbuild/BUILD` during the build. A relative `dist/` path would look for `~/rpmbuild/BUILD/dist/` which does not exist. The `%(pwd)` macro captures your actual project directory at spec-parse time so the path resolves correctly for local builds. In CI/CD, use `%{getenv:PROJECT_DIR}` instead (see CI/CD section below).

> **Why %dir in %files:** Listing `/usr/lib/myapp` without `%dir` causes rpmlint warnings and can create ownership conflicts on uninstall. `%dir` declares that rpmbuild owns the directory itself, while `/usr/lib/myapp/*` declares ownership of its contents.

## Step 5: Build the RPM

Before building, export your project directory so the spec resolves correctly:

<!-- Sets the project directory env var and builds both binary and source RPMs -->
```bash
export PROJECT_DIR=$(pwd)
rpmbuild -ba ~/rpmbuild/SPECS/myapp.spec
```

Expected output:

- `~/rpmbuild/RPMS/x86_64/myapp-1.0.0-1.x86_64.rpm`

## Step 6: Validate with rpmlint

<!-- Checks the RPM for common packaging errors and policy warnings -->
```bash
rpmlint ~/rpmbuild/RPMS/x86_64/myapp-1.0.0-1.x86_64.rpm
```

Fix warnings if possible before release.

## Step 7: Install and test

<!-- Installs the RPM locally to verify it works correctly -->
```bash
sudo dnf install -y ~/rpmbuild/RPMS/x86_64/myapp-1.0.0-1.x86_64.rpm
```

Run your app and then uninstall:

<!-- Removes the installed package cleanly -->
```bash
sudo dnf remove -y myapp
```

## Optional: Sign the RPM

<!-- Signs the RPM with a GPG key so users can verify its authenticity -->
```bash
rpm --addsign ~/rpmbuild/RPMS/x86_64/myapp-1.0.0-1.x86_64.rpm
```

> **Note:** RPM signing uses GPG, not the same certificate format as Windows code signing. You need a GPG key set up in `~/.rpmmacros` before this command will work. For public distribution, signing is required; for internal use it is optional.

## CI/CD with GitHub Actions

Automate RPM builds on every version tag. The workflow uses a Fedora container so `dnf` and the RPM toolchain are available natively on the Ubuntu runner.

> **Why a Fedora container on ubuntu-22.04:** `dnf` and `rpmbuild` are not available on Ubuntu. Using a `fedora:latest` container at the job level gives a proper RPM environment without needing a separate runner type. All `run:` steps execute inside the container; `uses:` actions like `actions/checkout` run in the container context and require `git` to be installed first.

Create `.github/workflows/build-rpm.yml` with this content:

```yaml
name: Build Linux RPM

on:
  push:
    branches: [main]
    tags: ["v*.*.*"]
  pull_request:
    branches: [main]

jobs:
  build-rpm:
    runs-on: ubuntu-22.04
    # Fedora container provides dnf and the RPM toolchain natively
    container: fedora:latest

    steps:
      - name: Install RPM build tools and git
        run: |
          dnf install -y \
            rpm-build rpmdevtools rpmlint gcc make \
            python3 python3-pip git

      - name: Checkout code
        uses: actions/checkout@v4

      - name: Create RPM build tree
        run: rpmdev-setuptree

      - name: Create virtual environment and install dependencies
        run: |
          python3 -m venv .venv
          . .venv/bin/activate
          pip install --upgrade pip
          pip install -r requirements.txt
          pip install pyinstaller

      - name: Build PyInstaller output
        run: |
          . .venv/bin/activate
          pyinstaller --name MyApp ./main.py

      - name: Extract version from git tag
        id: version
        run: |
          # RPM versions cannot contain hyphens — replace any with underscores
          if [[ "${GITHUB_REF}" == refs/tags/v* ]]; then
            VERSION="${GITHUB_REF_NAME#v}"
            echo "VERSION=${VERSION//-/_}" >> "$GITHUB_OUTPUT"
          else
            echo "VERSION=1.0.0" >> "$GITHUB_OUTPUT"
          fi

      - name: Copy spec file and inject version
        run: |
          VERSION="${{ steps.version.outputs.VERSION }}"
          cp packaging/rpm/myapp.spec ~/rpmbuild/SPECS/myapp.spec
          sed -i "s/^Version:.*/Version:        ${VERSION}/" ~/rpmbuild/SPECS/myapp.spec
          # Export workspace path so the spec's %global project_dir resolves correctly.
          # The spec uses %{getenv:PROJECT_DIR} in CI instead of %(pwd) because
          # rpmbuild changes directory during the build and %(pwd) would point to
          # ~/rpmbuild/BUILD, not this workspace.
          echo "PROJECT_DIR=${GITHUB_WORKSPACE}" >> "$GITHUB_ENV"

      - name: Build binary RPM
        run: |
          # -bb builds binary RPM only; -ba would also require GPG for source RPM signing
          rpmbuild -bb ~/rpmbuild/SPECS/myapp.spec

      - name: Validate with rpmlint
        run: |
          rpmlint ~/rpmbuild/RPMS/x86_64/myapp-*.rpm

      - name: Upload RPM artifact
        uses: actions/upload-artifact@v4
        with:
          name: MyApp-rpm-${{ github.ref_name }}
          path: ~/rpmbuild/RPMS/x86_64/*.rpm
          retention-days: 30
```

> **Note on the spec in CI:** In the workflow above, `PROJECT_DIR` is exported to `$GITHUB_ENV` and the spec reads it with `%{getenv:PROJECT_DIR}`. Update the `%global project_dir` line in your spec to `%global project_dir %{getenv:PROJECT_DIR}` for CI builds, or keep both approaches with a comment explaining which to use locally vs. in CI.

## Common Issues and Fixes

- `rpmdev-setuptree` not found: install `rpmdevtools`, not just `rpm-build`
- Missing files after install: check `%files` section
- Build errors: confirm `%install` paths exist and use `%{buildroot}` prefix
- Empty RPM: verify `%files` lists actual files from `%{buildroot}`
- Dependencies: add `Requires:` lines in the spec file

## Final Checklist

- Spec file exists at `packaging/rpm/myapp.spec` and is correct
- RPM builds without errors
- rpmlint warnings addressed
- Install and uninstall works
- GitHub Actions workflow committed to `.github/workflows/`
- Required secrets added to GitHub repository settings
- Build passes end-to-end on a tag push
- Signed artifact downloaded from Actions and tested on a clean machine
- Screenshot placeholders replaced with real screenshots in `images/linux/`
