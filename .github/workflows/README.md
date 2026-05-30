# GitHub Actions CI/CD Workflows — Setup Guide

This document explains how to set up each of the six automated packaging workflows for your own Python project. Each section is self-contained: read only the one you need.

## Workflow Index

| Workflow | Format | Platform | Setup Guide |
|---|---|---|---|
| [`build-exe.yml`](build-exe.yml) | Standalone EXE | Windows | [Go to steps ➔](#build-exeyml--windows-standalone-exe) |
| [`build-msi.yml`](build-msi.yml) | MSI Installer | Windows | [Go to steps ➔](#build-msiyml--windows-msi-installer) |
| [`build-msix.yml`](build-msix.yml) | MSIX Package | Windows | [Go to steps ➔](#build-msixyml--windows-msix-package) |
| [`build-deb.yml`](build-deb.yml) | DEB Package | Debian / Ubuntu | [Go to steps ➔](#build-debyml--debianubuntu-deb-package) |
| [`build-rpm.yml`](build-rpm.yml) | RPM Package | RHEL / Fedora / CentOS | [Go to steps ➔](#build-rpmyml--rhelfe-doracentos-rpm-package) |
| [`build-apk.yml`](build-apk.yml) | Android APK | Android | [Go to steps ➔](#build-apkyml--android-apk) |

---

## How Every Workflow Works (Quick Concepts)

- **Branch push or pull request**: runs the full build pipeline and uploads an **unsigned** artifact for testing. No secrets needed.
- **Version tag push (`v*.*.*`)**: runs the same build, then additionally signs the output using your stored secrets, and uploads a **production-ready signed** artifact.
- **Dynamic versioning**: the tag name (e.g. `v1.2.3`) is automatically extracted and injected into your package version fields. On branches, a fallback version like `0.0.0-dev` is used.

---

---

## `build-exe.yml` — Windows Standalone EXE

### What You Will Build

An automated pipeline that produces a Windows EXE from your Python app using PyInstaller. Every push to `main` creates a downloadable unsigned EXE. Every version tag creates a digitally signed EXE.

### Time Estimate

~10 minutes to configure. First CI run takes ~5 minutes. Signed release runs take ~6 minutes.

### Related Guide

Read the full local build steps first: [python-to-exe.md](../../Python/windows/python-to-exe.md)

### Prerequisites

- Your app runs locally with `python main.py`
- `requirements.txt` exists in the project root
- A PyInstaller spec file (`MyApp.spec`) exists, **or** you build directly from `main.py`
- The project is hosted on GitHub

### Concepts

- `pyinstaller`: bundles Python and your app into a self-contained EXE
- `signtool`: signs the EXE with a certificate so Windows does not block it
- `GITHUB_REF`: the git branch or tag that triggered the workflow run
- GitHub Secrets: encrypted variables stored in your repo settings, injected at runtime

### Step 1: Copy the workflow file

Copy `build-exe.yml` into your project at this path:

```text
your-project/
  .github/
    workflows/
      build-exe.yml
```

Expected result: the file exists in your repository at `.github/workflows/build-exe.yml`.

### Step 2: Edit the workflow — set your app name

Open `build-exe.yml` in a text editor. Find every occurrence of `MyApp` and replace it with your actual application name (for example `MyCoolApp`).

Lines to update:

```yaml
# Line: Build EXE with PyInstaller
pyinstaller MyApp.spec --noconfirm
# Change to:
pyinstaller MyCoolApp.spec --noconfirm

# Line: Sign EXE
dist/MyApp/MyApp.exe
# Change to:
dist/MyCoolApp/MyCoolApp.exe

# Line: Upload artifact
path: dist/MyApp/
# Change to:
path: dist/MyCoolApp/
```

If your entry script is not `main.py`, also update the build line:

```yaml
pyinstaller --name MyCoolApp --noconfirm .\run.py
```

Expected result: no occurrences of `MyApp` remain in the file.

### Step 3: Add your app's `.gitignore` entries

In your project root, add these lines to `.gitignore` to prevent build outputs from being committed:

```text
dist/
build/
*.spec
*.pfx
*.log
```

Expected result: running `git status` after a local build does not show `dist/` or `build/` as untracked.

### Step 4: Commit and push

```bash
git add .github/workflows/build-exe.yml .gitignore
git commit -m "ci: add Windows EXE build workflow"
git push origin main
```

Expected result: the **Actions** tab on GitHub shows a new workflow run called **Build Windows EXE** has started.

### Step 5: Verify the test build

1. Go to your repository on GitHub and click the **Actions** tab.
2. Click the **Build Windows EXE** workflow in the left column.
3. Click the running or completed job to see step-by-step logs.
4. Once it shows a green checkmark, click the **Summary** link.
5. Scroll to the bottom and download the artifact ZIP.
6. Extract it and run the EXE to confirm it launches correctly.

Expected result: EXE runs without needing Python installed.

### Step 6: Set up signing secrets (production only)

Skip this step if you only need unsigned test builds.

**Create a self-signed certificate (for testing; use a CA certificate for production):**

Open PowerShell as Administrator:

```powershell
$cert = New-SelfSignedCertificate `
  -Type CodeSigningCert `
  -Subject "CN=YourCompanyName" `
  -KeyUsage DigitalSignature `
  -FriendlyName "GitHub Actions Signer" `
  -NotAfter (Get-Date).AddYears(3)

$pwd = ConvertTo-SecureString "YourPassword" -AsPlainText -Force
Export-PfxCertificate -Cert $cert -FilePath "$home\Desktop\my-cert.pfx" -Password $pwd
```

**Encode the certificate to Base64:**

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("$home\Desktop\my-cert.pfx")) | Out-File certificate_base64.txt
```

**Add secrets to GitHub:**

1. Go to your repository → **Settings** → **Secrets and variables** → **Actions**.
2. Click **New repository secret**.
3. Add `WINDOWS_CERTIFICATE_BASE64` — paste the full contents of `certificate_base64.txt`.
4. Add `WINDOWS_CERTIFICATE_PASSWORD` — type the plain-text password you used above.

Expected result: both secrets appear in the repository secrets list.

### Step 7: Release a signed EXE

```bash
git tag v1.0.0
git push origin v1.0.0
```

Expected result: a new Actions run starts. The signing step runs and produces a signed EXE artifact.

**Verify the signature:**

Download the artifact, right-click the `.exe` → **Properties** → **Digital Signatures** tab → confirm your certificate name is listed.

### Common Issues

- `MyApp.spec not found`: either create the spec file locally first with `pyinstaller --name MyApp main.py`, or switch the workflow to run `pyinstaller --name MyApp --noconfirm .\main.py` directly.
- Signing step fails with `certificate not found`: confirm `WINDOWS_CERTIFICATE_BASE64` contains no extra newlines and the password matches exactly.
- EXE missing files: add missing modules to `hiddenimports` and data folders to `datas` in your spec file.

### Final Checklist

- `build-exe.yml` copied into `.github/workflows/`
- All `MyApp` references replaced with your app name
- `.gitignore` updated
- Test build passes and unsigned EXE runs correctly
- (Optional) Signing secrets added and signed release verified

---

## `build-msi.yml` — Windows MSI Installer

### What You Will Build

An automated pipeline that builds a PyInstaller folder output and wraps it into an MSI installer using WiX Toolset v3. Every push to `main` creates an unsigned MSI. Every version tag creates a signed MSI.

### Time Estimate

~15 minutes to configure. First CI run takes ~8 minutes.

### Related Guide

Read the full local build steps first: [python-to-msi.md](../../Python/windows/python-to-msi.md)

### Prerequisites

- Folder-based EXE build works locally (complete the EXE guide first)
- `packaging/msi/installer.wxs` exists with real GUIDs and correct file references
- WiX Toolset v3 installed locally to author the `.wxs` file

### Concepts

- `heat`: WiX tool that scans a folder and auto-generates component XML
- `candle`: WiX compiler that turns `.wxs` files into `.wixobj` object files
- `light`: WiX linker that combines `.wixobj` files into the final `.msi`
- `MajorUpgrade`: WiX element that uninstalls old versions on upgrade automatically

### Step 1: Copy the workflow file

```text
your-project/
  .github/
    workflows/
      build-msi.yml
```

### Step 2: Edit the workflow — set your app name

Open `build-msi.yml`. Find every `MyApp` and replace with your app name. Key lines:

```yaml
# PyInstaller build
pyinstaller MyApp.spec --noconfirm
# Change to:
pyinstaller MyCoolApp.spec --noconfirm

# heat — harvest the _internal folder
heat dir dist/MyApp/_internal ...
# Change to:
heat dir dist/MyCoolApp/_internal ...

# candle — compile WiX sources
candle packaging/msi/installer.wxs packaging/msi/HarvestedFiles.wxs

# light — link into MSI
light installer.wixobj HarvestedFiles.wixobj -o MyApp.msi ...
# Change to:
light installer.wixobj HarvestedFiles.wixobj -o MyCoolApp.msi ...

# Upload artifact path
path: MyApp.msi
# Change to:
path: MyCoolApp.msi
```

> **Note:** If your PyInstaller build does not produce an `_internal` folder (older layout), point `heat` at `dist/MyCoolApp` instead of `dist/MyCoolApp/_internal`.

### Step 3: Verify `installer.wxs`

Open `packaging/msi/installer.wxs` and confirm:

- `UpgradeCode` is a real GUID (generate one with `[guid]::NewGuid()` in PowerShell — never use placeholder text).
- `File Source="dist\MyApp\MyApp.exe"` references your actual renamed EXE path.
- `ComponentGroupRef Id="InternalComponentGroup"` is inside the `Feature` element.

### Step 4: Add `.gitignore` entries

```text
dist/
build/
*.spec
*.wixobj
*.wixpdb
packaging/msi/HarvestedFiles.wxs
*.pfx
*.log
```

### Step 5: Commit and push

```bash
git add .github/workflows/build-msi.yml packaging/msi/installer.wxs .gitignore
git commit -m "ci: add Windows MSI build workflow"
git push origin main
```

Expected result: **Build Windows MSI** appears in the Actions tab and completes with a green checkmark.

### Step 6: Verify the test build

Download the MSI artifact from the Actions Summary. Double-click it to confirm:

- The installer wizard appears.
- The app installs to Program Files.
- Start Menu shortcut works.
- Uninstall from Add/Remove Programs works cleanly.

### Step 7: Set up signing secrets

Same process as the EXE workflow — see [Step 6 of the EXE guide](#step-6-set-up-signing-secrets-production-only). The same two secrets (`WINDOWS_CERTIFICATE_BASE64` and `WINDOWS_CERTIFICATE_PASSWORD`) are reused.

### Step 8: Release a signed MSI

```bash
git tag v1.0.0
git push origin v1.0.0
```

Expected result: signed `MyCoolApp.msi` appears as a downloadable artifact.

**Verify:** right-click the `.msi` → **Properties** → **Digital Signatures** tab.

### Common Issues

- `candle` not found: WiX was not added to PATH by Chocolatey. The workflow handles this automatically — check the WiX install step logs.
- `heat` produces empty `HarvestedFiles.wxs`: the `dist/MyApp/_internal` path does not exist — check which layout PyInstaller produced.
- ICE64 validation error: you have a `RemoveFolder` element inside a component that targets `DesktopFolder`. Remove it — only use `RemoveFolder` inside custom subfolder components.

### Final Checklist

- `build-msi.yml` copied into `.github/workflows/`
- All `MyApp` references replaced
- `installer.wxs` has real GUIDs and correct paths
- `.gitignore` updated
- Test build produces a working MSI installer
- (Optional) Signing secrets added and signed MSI verified

---

## `build-msix.yml` — Windows MSIX Package

### What You Will Build

An automated pipeline that packages a PyInstaller folder build into a signed MSIX package using `makeappx` and `signtool`.

### Time Estimate

~15 minutes to configure. First CI run takes ~6 minutes.

### Related Guide

Read the full local build steps first: [python-to-msix.md](../../Python/windows/python-to-msix.md)

### Prerequisites

- Folder-based EXE build works locally
- `dist\MyApp\AppxManifest.xml` exists with your identity fields filled in
- `dist\MyApp\Assets\` contains `StoreLogo.png`, `Square44x44Logo.png`, `Square150x150Logo.png`

### Concepts

- `AppxManifest.xml`: XML file that defines your package identity, version, and icon paths
- `makeappx`: Microsoft tool that packs a folder into an MSIX file
- `Publisher`: the certificate Subject Name — must match exactly between the manifest and the signing certificate

### Step 1: Copy the workflow file

```text
your-project/
  .github/
    workflows/
      build-msix.yml
```

### Step 2: Edit the workflow — set your app name

Open `build-msix.yml`. Replace every `MyApp` with your app name:

```yaml
# PyInstaller build
pyinstaller MyApp.spec --noconfirm
# Change to:
pyinstaller MyCoolApp.spec --noconfirm

# Version inject into manifest
dist/MyApp/AppxManifest.xml
# Change to:
dist/MyCoolApp/AppxManifest.xml

# makeappx pack
makeappx pack /d dist/MyApp /p MyApp.msix /nv
# Change to:
makeappx pack /d dist/MyCoolApp /p MyCoolApp.msix /nv

# Sign step
MyApp.msix
# Change to:
MyCoolApp.msix

# Upload path
path: MyApp.msix
# Change to:
path: MyCoolApp.msix
```

### Step 3: Update `AppxManifest.xml`

Open `dist\MyApp\AppxManifest.xml` (rename the folder first if you changed the app name). Confirm these fields match your actual values:

```xml
<Identity
  Name="YourCompany.YourApp"
  Publisher="CN=YourCompanyName"
  Version="1.0.0.0" />
```

> **Critical:** The `Publisher` value must exactly match the `Subject` of your `.pfx` certificate — for example if you created a certificate with `-Subject "CN=MyTestingCompany"`, the manifest must say `Publisher="CN=MyTestingCompany"`. A mismatch causes `signtool` to reject the package.

### Step 4: Add `.gitignore` entries

```text
dist/
build/
*.spec
*.msix
*.pfx
*.log
```

### Step 5: Commit and push

```bash
git add .github/workflows/build-msix.yml .gitignore
git commit -m "ci: add Windows MSIX build workflow"
git push origin main
```

Expected result: **Build Windows MSIX** appears in Actions and completes with a green checkmark and an MSIX artifact.

### Step 6: Verify the test build

Download the MSIX artifact. To install an unsigned MSIX locally for testing:

```powershell
Add-AppxPackage -Path .\MyCoolApp.msix
```

> **Note:** For an unsigned package to install, you must first trust your self-signed certificate. Right-click the `.msix` → **Properties** → **Digital Signatures** → install the certificate to **Trusted Root Certification Authorities** on the local machine.

### Step 7: Set up signing secrets

Same two secrets as the EXE workflow — see [Step 6 of the EXE guide](#step-6-set-up-signing-secrets-production-only).

### Step 8: Release a signed MSIX

```bash
git tag v1.0.0
git push origin v1.0.0
```

Expected result: signed `MyCoolApp.msix` appears as a downloadable artifact that installs with a double-click.

### Common Issues

- `makeappx` not found: Windows SDK is not installed or not on PATH — the workflow installs it automatically via the runner's existing SDK tools.
- Install fails with `Publisher does not match`: check `Publisher` in `AppxManifest.xml` matches the certificate Subject exactly, including the `CN=` prefix.
- Missing logo error: confirm `Square150x150Logo.png` exists inside `Assets\` — it is required even if `Square44x44Logo.png` is present.

### Final Checklist

- `build-msix.yml` copied into `.github/workflows/`
- All `MyApp` references replaced
- `AppxManifest.xml` has correct `Name`, `Publisher`, and logo paths
- All three required logo assets exist in `Assets\`
- Test build produces an MSIX that installs
- (Optional) Signing secrets added and signed MSIX installs with a double-click

---

## `build-deb.yml` — Debian/Ubuntu DEB Package

### What You Will Build

An automated pipeline that builds a `.deb` package using `dpkg-buildpackage` and validates it with `lintian` on an Ubuntu runner. Every version tag produces a versioned, uploadable `.deb`.

### Time Estimate

~10 minutes to configure. CI runs take ~5 minutes.

### Related Guide

Read the full local build steps first: [python-to-deb.md](../../Python/linux/python-to-deb.md)

### Prerequisites

- Your app builds locally with `dpkg-buildpackage -us -uc`
- `debian/control`, `debian/rules`, `debian/changelog` exist and are correct
- `debian/rules` has `override_dh_auto_build` (runs PyInstaller) and `override_dh_auto_install` (copies output)

### Concepts

- `dpkg-buildpackage`: builds the `.deb` from the `debian/` folder — runs `debian/rules` automatically
- `lintian`: Debian policy checker — flags common mistakes before release
- `dch`: updates `debian/changelog` with the new version — the workflow runs this automatically on tag builds
- `override_dh_auto_build`: custom build step that replaces debhelper's default and runs PyInstaller instead

### Step 1: Copy the workflow file

```text
your-project/
  .github/
    workflows/
      build-deb.yml
```

### Step 2: Edit the workflow — set your package name

Open `build-deb.yml`. The version injection and build steps are automatic. Update only the artifact name:

```yaml
# Artifact upload name
name: MyApp-deb-${{ github.ref_name }}
# Change to:
name: MyCoolApp-deb-${{ github.ref_name }}
```

### Step 3: Verify `debian/control`

Open `debian/control` and confirm the `Package` field matches your app name and the `Architecture` matches your build target:

```text
Package: mycoolapp
Architecture: amd64
Depends: ${misc:Depends}
Description: One-line description of your app
```

### Step 4: Verify `debian/rules`

Your `debian/rules` must override the default build and install steps. Confirm it looks like this (with real paths for your app):

```makefile
#!/usr/bin/make -f
%:
	dh $@

override_dh_auto_build:
	python3 -m venv .venv
	. .venv/bin/activate && pip install -r requirements.txt
	. .venv/bin/activate && pip install pyinstaller
	. .venv/bin/activate && pyinstaller --name MyCoolApp ./main.py

override_dh_auto_install:
	mkdir -p debian/mycoolapp/usr/lib/mycoolapp
	cp -a dist/MyCoolApp/* debian/mycoolapp/usr/lib/mycoolapp/
	mkdir -p debian/mycoolapp/usr/bin
	ln -s /usr/lib/mycoolapp/MyCoolApp debian/mycoolapp/usr/bin/mycoolapp
```

Expected result: `debian/rules` is executable (`chmod +x debian/rules`).

### Step 5: Add `.gitignore` entries

```text
dist/
build/
*.spec
*.log
```

### Step 6: Commit and push

```bash
git add .github/workflows/build-deb.yml debian/
git commit -m "ci: add Linux DEB build workflow"
git push origin main
```

Expected result: **Build Linux DEB** appears in the Actions tab and completes with a green checkmark and a `.deb` artifact.

### Step 7: Verify the test build

Download the `.deb` artifact and install it on a Debian or Ubuntu machine:

```bash
sudo dpkg -i mycoolapp_1.0.0-1_amd64.deb
mycoolapp
```

Expected result: the app launches. Uninstall cleanly:

```bash
sudo dpkg -r mycoolapp
```

### Step 8: Release a versioned DEB

```bash
git tag v1.0.0
git push origin v1.0.0
```

Expected result: the workflow injects `1.0.0` into `debian/changelog` via `dch`, builds, and uploads `mycoolapp_1.0.0-1_amd64.deb` as a release artifact.

### Common Issues

- Empty `.deb` after install: `debian/rules` is missing `override_dh_auto_install` — debhelper's default install step silently produces an empty package for non-setuptools projects.
- `dch: command not found`: install `devscripts` — the workflow installs it automatically but your local machine may not have it.
- `lintian` errors: fix each one before release. Common causes are missing `Description` fields in `debian/control` or incorrect file ownership in `%files`.

### Final Checklist

- `build-deb.yml` copied into `.github/workflows/`
- App name updated in artifact name line
- `debian/control` has correct `Package` name and `Architecture`
- `debian/rules` has both `override_dh_auto_build` and `override_dh_auto_install`
- `debian/rules` is executable
- Test build produces a `.deb` that installs and runs correctly
- Lintian passes without errors

---

## `build-rpm.yml` — RHEL/Fedora/CentOS RPM Package

### What You Will Build

An automated pipeline that builds an `.rpm` package using `rpmbuild` inside a Fedora container on an Ubuntu runner. Every version tag produces a versioned, uploadable `.rpm`.

### Time Estimate

~15 minutes to configure. CI runs take ~8 minutes (first run longer due to container image pull).

### Related Guide

Read the full local build steps first: [python-to-rpm.md](../../Python/linux/python-to-rpm.md)

### Prerequisites

- Your app builds locally with `rpmbuild -bb packaging/rpm/myapp.spec`
- `packaging/rpm/myapp.spec` exists with correct `%install` and `%files` sections
- Your spec file uses `%{getenv:PROJECT_DIR}` for the project path (not `%(pwd)` — that only works locally)

### Concepts

- `rpmbuild`: the RPM build tool — reads a `.spec` file and produces the `.rpm`
- `%buildroot`: the staging directory where `%install` copies files before packaging
- `%{getenv:PROJECT_DIR}`: reads the `PROJECT_DIR` environment variable set by the workflow — resolves the workspace path correctly inside the Fedora container
- Fedora container: the workflow runs inside `fedora:latest` to provide `dnf` and `rpmbuild` natively on the Ubuntu runner

### Step 1: Copy the workflow file

```text
your-project/
  .github/
    workflows/
      build-rpm.yml
```

### Step 2: Edit the workflow — set your app and spec name

Open `build-rpm.yml` and update these lines:

```yaml
# Copy spec file step
cp packaging/rpm/myapp.spec ~/rpmbuild/SPECS/myapp.spec
sed -i "s/^Version:.*/Version: ${VERSION}/" ~/rpmbuild/SPECS/myapp.spec
# Change both myapp.spec occurrences to your spec filename:
cp packaging/rpm/mycoolapp.spec ~/rpmbuild/SPECS/mycoolapp.spec
sed -i "s/^Version:.*/Version: ${VERSION}/" ~/rpmbuild/SPECS/mycoolapp.spec

# PyInstaller build step
pyinstaller --name MyApp ./main.py
# Change to:
pyinstaller --name MyCoolApp ./main.py

# rpmbuild step
rpmbuild -bb ~/rpmbuild/SPECS/myapp.spec
# Change to:
rpmbuild -bb ~/rpmbuild/SPECS/mycoolapp.spec

# rpmlint validation
rpmlint ~/rpmbuild/RPMS/x86_64/myapp-*.rpm
# Change to:
rpmlint ~/rpmbuild/RPMS/x86_64/mycoolapp-*.rpm

# Artifact name
name: MyApp-rpm-${{ github.ref_name }}
# Change to:
name: MyCoolApp-rpm-${{ github.ref_name }}
```

### Step 3: Update your spec file for CI

Open `packaging/rpm/mycoolapp.spec`. Make sure the project directory macro uses the environment variable (not `%(pwd)`) so it works inside the container:

```spec
# For CI — reads the environment variable set by the workflow:
%global project_dir %{getenv:PROJECT_DIR}

# For local builds — use this instead (comment out the line above):
# %global project_dir %(pwd)
```

Also confirm `%install` uses `%{buildroot}` on every path:

```spec
%install
mkdir -p %{buildroot}/usr/lib/mycoolapp
cp -a %{project_dir}/dist/MyCoolApp/* %{buildroot}/usr/lib/mycoolapp/
mkdir -p %{buildroot}/usr/bin
ln -s /usr/lib/mycoolapp/MyCoolApp %{buildroot}/usr/bin/mycoolapp

%files
%dir /usr/lib/mycoolapp
/usr/lib/mycoolapp/*
/usr/bin/mycoolapp
```

### Step 4: Add `.gitignore` entries

```text
dist/
build/
*.spec.bak
*.log
```

### Step 5: Commit and push

```bash
git add .github/workflows/build-rpm.yml packaging/rpm/mycoolapp.spec
git commit -m "ci: add Linux RPM build workflow"
git push origin main
```

Expected result: **Build Linux RPM** appears in the Actions tab and completes with a green checkmark and an `.rpm` artifact.

### Step 6: Verify the test build

Download the `.rpm` artifact and install it on a Fedora or RHEL machine:

```bash
sudo dnf install -y mycoolapp-1.0.0-1.x86_64.rpm
mycoolapp
```

Expected result: the app launches. Uninstall cleanly:

```bash
sudo dnf remove -y mycoolapp
```

### Step 7: Release a versioned RPM

```bash
git tag v1.0.0
git push origin v1.0.0
```

Expected result: the workflow injects `1.0.0` into the spec via `sed`, builds, and uploads `mycoolapp-1.0.0-1.x86_64.rpm` as a release artifact.

### Common Issues

- `rpmdev-setuptree: command not found`: the `rpmdevtools` package is missing — the workflow installs it automatically, but check the install step logs.
- Empty RPM after install: `%files` section is missing entries — every file copied in `%install` must appear in `%files`.
- `%{getenv:PROJECT_DIR}` is empty: the workflow sets `PROJECT_DIR` via `$GITHUB_ENV` before the `rpmbuild` step — confirm that step ran successfully and look for it in the logs.

### Final Checklist

- `build-rpm.yml` copied into `.github/workflows/`
- All `myapp` references replaced with your package name
- Spec file uses `%{getenv:PROJECT_DIR}` for the project path
- `%install` uses `%{buildroot}` prefix on all paths
- `%files` uses `%dir` for directory ownership
- Test build produces an `.rpm` that installs and runs correctly
- `rpmlint` passes without errors

---

## `build-apk.yml` — Android APK

### What You Will Build

An automated pipeline that builds a debug APK on every push to `main` and a signed, aligned release APK on every version tag, using Buildozer and Android SDK tools.

### Time Estimate

~20 minutes to configure. First CI run takes 20–30 minutes (Android SDK/NDK download ~1–2 GB). Subsequent cached runs take ~5 minutes.

### Related Guide

Read the full local build steps first: [python-to-apk.md](../../Python/android/python-to-apk.md)

### Prerequisites

- Your app builds locally with `buildozer -v android debug`
- `buildozer.spec` exists in the project root with a `version = 1.0.0` line
- `package.domain` + `package.name` form a unique reverse-DNS identifier (e.g. `com.yourcompany.myapp`)

### Concepts

- `buildozer`: automates Android SDK/NDK setup and APK compilation for Kivy apps
- `zipalign`: aligns the APK memory boundaries — required before signing
- `apksigner`: signs the aligned APK with your keystore
- Buildozer cache (`~/.buildozer`): caches Android SDK/NDK between runs to cut build times from ~30 to ~5 minutes
- `versionCode`: an integer that must increase with every release — the workflow uses `github.run_number` for this automatically

### Step 1: Copy the workflow file

```text
your-project/
  .github/
    workflows/
      build-apk.yml
```

### Step 2: Edit the workflow — set your app name

Open `build-apk.yml`. Replace every `MyApp` with your app name. Key lines:

```yaml
# Debug artifact upload
name: MyApp-apk-${{ github.ref_name }}
# Change to:
name: MyCoolApp-apk-${{ github.ref_name }}

# Release APK input to zipalign
bin/MyApp-*-release-unsigned.apk
# Change to: use the exact prefix buildozer uses for your app — check locally in bin/ after a release build
bin/MyCoolApp-*-release-unsigned.apk

# zipalign output and apksigner input/output
bin/MyApp-release-aligned.apk
bin/MyApp-release-signed.apk
# Change both to:
bin/MyCoolApp-release-aligned.apk
bin/MyCoolApp-release-signed.apk

# Signed release artifact upload
path: bin/MyApp-release-signed.apk
# Change to:
path: bin/MyCoolApp-release-signed.apk
```

### Step 3: Update `buildozer.spec`

Open `buildozer.spec` and confirm these fields are set correctly:

```ini
title = MyCoolApp
package.name = mycoolapp
package.domain = com.yourcompany
requirements = python3,kivy
version = 1.0.0
```

> **Important:** The `version = ` line must be present exactly as shown. The workflow uses `sed` to find and overwrite it on tag builds. If the line is missing or formatted differently, the version injection silently fails.

> **Important:** `package.domain` + `package.name` form your app's permanent package ID on the Play Store (e.g. `com.yourcompany.mycoolapp`). Choose a domain you own. This cannot be changed after publishing.

### Step 4: Create a release keystore

Do this once. Back up the keystore file securely — losing it means you can never update your app on the Play Store.

```bash
keytool -genkeypair -v \
  -keystore my-release-key.jks \
  -keyalg RSA -keysize 2048 \
  -validity 10000 \
  -alias mykey
```

Enter a strong password when prompted. Remember both the keystore password and the alias name.

Add the keystore to `.gitignore` immediately:

```text
*.jks
*.keystore
```

### Step 5: Encode the keystore and add secrets

**On Linux or macOS:**

```bash
base64 -w 0 my-release-key.jks > keystore_base64.txt
```

**On Windows (PowerShell):**

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("my-release-key.jks")) | Out-File keystore_base64.txt
```

Add all four secrets to GitHub under **Settings** → **Secrets and variables** → **Actions**:

| Secret name | Value |
|---|---|
| `KEYSTORE_BASE64` | Full contents of `keystore_base64.txt` |
| `KEYSTORE_PASSWORD` | The keystore password from `keytool` |
| `KEY_ALIAS` | The alias name you used (e.g. `mykey`) |
| `KEY_PASSWORD` | The alias password (often same as keystore password) |

### Step 6: Commit and push

```bash
git add .github/workflows/build-apk.yml buildozer.spec .gitignore
git commit -m "ci: add Android APK build workflow"
git push origin main
```

Expected result: **Build Android APK** appears in the Actions tab. The first run takes 20–30 minutes to download the Android SDK and NDK. Subsequent runs use the cache and take ~5 minutes.

### Step 7: Verify the test build

Download the debug APK artifact. Install it on a connected Android device:

```bash
adb install bin/mycoolapp-debug.apk
```

Expected result: the app installs and launches. Test all core features.

### Step 8: Release a signed APK

```bash
git tag v1.0.0
git push origin v1.0.0
```

Expected result: a signed, aligned release APK is uploaded as a workflow artifact.

**Verify the signature:**

```bash
apksigner verify --verbose bin/MyCoolApp-release-signed.apk
```

Expected result: output confirms `Verified using v1 scheme: true` or `v2 scheme: true`.

### Common Issues

- `buildozer` fails on Windows: Buildozer requires Linux. Use WSL2 locally or rely on the Ubuntu CI runner.
- Black screen on device: confirm `.kv` files are listed in `source.include_exts` in `buildozer.spec`.
- `zipalign: no such file`: the Android SDK build-tools are not on PATH — the workflow handles this via `android-actions/setup-android`. Check the SDK setup step logs.
- Version inject does not take effect: confirm `buildozer.spec` has a `version = ...` line with no leading spaces or extra characters.

### Final Checklist

- `build-apk.yml` copied into `.github/workflows/`
- All `MyApp` references replaced with your app name and APK filename prefix
- `buildozer.spec` has `title`, `package.name`, `package.domain`, `requirements`, and `version` set
- `*.jks` added to `.gitignore` before first commit
- Keystore backed up to a secure location outside the repository
- All four Android signing secrets added to GitHub
- Test build produces a debug APK that installs on a device
- Signed release APK verified with `apksigner`

---

## Security Best Practices (All Workflows)

1. **Ephemeral key decoding**: workflows decode `.pfx` and `.jks` files from Base64 at runtime, use them immediately for signing, then delete them with `rm` before the job ends. The key never persists on the runner.
2. **Tag-only signing**: all `signtool sign` and `apksigner sign` steps are wrapped in `if: startsWith(github.ref, 'refs/tags/')`. Branch builds and pull requests produce unsigned artifacts and never access secrets.
3. **Never commit keys**: add `*.pfx`, `*.jks`, and `*.keystore` to `.gitignore` before your first commit. If you accidentally commit a key, rotate it immediately.
4. **Artifact retention**: artifacts are kept for 30 days then automatically deleted by GitHub to manage storage.
