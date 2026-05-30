# Python to MSIX (Windows) — Ultra-Detailed Beginner Guide

This guide packages a Python app into MSIX using beginner-friendly steps. It includes a GUI path and a CLI path, plus explanations and expected results.

## When to Use This Format

Use the MSIX format when you want Microsoft Store distribution or modern update support via AppInstaller. MSIX also provides clean install/uninstall and strong sandboxing compared to traditional EXE or MSI.

## Time Estimate

~20 minutes for a first build once your EXE is ready (longer if you need to set up signing certificates).

> **ARM64 note:** This guide produces x86_64 MSIX packages only. To build for ARM64 Windows, run PyInstaller and `makeappx` on an ARM64 Windows machine. Update the `ProcessorArchitecture` attribute in `AppxManifest.xml` to `arm64` when targeting ARM64. GitHub Actions provides `windows-11-arm` hosted runners for this purpose.

## Related Guides (Packaging Order)

- Required first: Build the EXE in [python-to-exe.md](python-to-exe.md)
- If you already have an MSI workflow: See [python-to-msi.md](python-to-msi.md)

## Purpose

This guide shows how to package a Windows EXE into an MSIX package with a proper identity, icons, and signing.

## Who Benefits

- Developers who want a modern Windows installer format
- Teams that need clean install/uninstall and update support

## What You Will Build

- A signed `.msix` package ready to install on Windows 10/11
- Optional AppInstaller for updates

## Who This Is For

- Beginners who have a working EXE build
- Teams that want a modern Windows packaging format

## Prerequisites

- A folder-based EXE build (recommended)
- MSIX Packaging Tool (GUI) or Windows SDK tools (`makeappx`, `signtool`)
- A code signing certificate for production

### Install MSIX Packaging Tool (GUI)

1. Open Microsoft Store.
2. Search for "MSIX Packaging Tool" and install it.
3. Launch it once to ensure it starts correctly.

### Install Windows SDK (CLI tools)

1. Download Windows SDK from Microsoft.
2. During install, select "Windows SDK for Desktop C++ Apps".
3. Confirm `makeappx.exe` and `signtool.exe` are available on PATH.

Verify in PowerShell:

<!-- Confirms that the makeappx and signtool commands are available on your PATH -->
```powershell
makeappx /?
signtool /?
```

Expected result: each command prints help text.

## Concepts (Quick)

- `AppxManifest.xml` defines identity, version, and logos
- Publisher name must match the signing certificate
- `makeappx` builds the package from a folder
- `signtool` signs the package

## Step 1: Build a folder-based EXE

MSIX packages folder outputs. Build your EXE like this:

<!-- Packages the Python app into a dist\MyApp folder ready for MSIX wrapping -->
```powershell
pyinstaller --name MyApp .\main.py
```

Expected output:

- `dist\MyApp\MyApp.exe`
- `dist\MyApp\_internal\` folder

<!-- TODO: add screenshot: ../../images/windows/dist-folder-msix.png -->

## Step 2: Add an MSIX manifest and assets

Create a folder structure inside `dist\MyApp\`:

```text
dist\MyApp\
   AppxManifest.xml
   Assets\
      StoreLogo.png
      Square44x44Logo.png
      Square150x150Logo.png
```

> **Note:** `Square150x150Logo.png` is required — it is referenced by the manifest's `uap:VisualElements` block. Without it the package will fail validation.

Your `AppxManifest.xml` defines identity, version, and display name. Use a template from Microsoft docs and update it for your app.

<!-- TODO: add screenshot: ../../images/windows/appxmanifest-editor.png -->

### Example Minimal Manifest (Beginner-Friendly)

The manifest below includes the `runFullTrust` capability, which is required for desktop Python applications that need full access to the file system and system resources.

> **Note:** Replace `MyCompany.MyApp` and `CN=YourCompany` with your actual package name and the subject of your signing certificate. The `Publisher` field must match exactly.

```xml
<?xml version="1.0" encoding="utf-8"?>
<Package
   xmlns="http://schemas.microsoft.com/appx/manifest/foundation/windows10"
   xmlns:uap="http://schemas.microsoft.com/appx/manifest/uap/windows10"
   xmlns:rescap="http://schemas.microsoft.com/appx/manifest/foundation/windows10/restrictedcapabilities"
   IgnorableNamespaces="uap rescap">

   <Identity Name="MyCompany.MyApp" Publisher="CN=YourCompany" Version="1.0.0.0" />
   <Properties>
      <DisplayName>MyApp</DisplayName>
      <PublisherDisplayName>YourCompany</PublisherDisplayName>
      <Logo>Assets\StoreLogo.png</Logo>
   </Properties>

   <Resources>
      <Resource Language="en-us" />
   </Resources>

   <Applications>
      <Application Id="App" Executable="MyApp.exe" EntryPoint="Windows.FullTrustApplication">
         <uap:VisualElements
            DisplayName="MyApp"
            Description="MyApp"
            BackgroundColor="transparent"
            Square44x44Logo="Assets\Square44x44Logo.png"
            Square150x150Logo="Assets\Square150x150Logo.png" />
      </Application>
   </Applications>

   <Capabilities>
      <rescap:Capability Name="runFullTrust" />
   </Capabilities>

</Package>
```

Update:

- `Name`: a unique package name
- `Publisher`: must match your signing certificate
- `Version`: increase on every release

## Option A: MSIX Packaging Tool (GUI)

1. Open MSIX Packaging Tool.
2. Click "Application package".
3. Select "Create package on this computer".
4. Point it to your EXE output folder.
5. Fill in Publisher, Package Name, Version.
6. Save the `.msix` file.

<!-- TODO: add screenshot: ../../images/windows/msix-packaging-tool-wizard.png -->

## Option B: CLI Build (Repeatable)

From the project root:

<!-- Packs the dist\MyApp folder (including AppxManifest.xml) into an MSIX file -->
```powershell
makeappx pack /d dist\MyApp /p MyApp.msix
```

Expected result: `MyApp.msix` created.

## Step 3: Sign the MSIX

> **EV certificate requirement:** MSIX packages installed outside the Microsoft Store must be signed with a certificate that the target machine trusts. For internal distribution, you can use a self-signed certificate and install it as a trusted root — but only do this on machines you control. For public distribution, use an EV certificate from a trusted CA (DigiCert, Sectigo, etc.) so users can install without extra steps.

<!-- Signs the MSIX package with a trusted certificate so Windows will install it -->
```powershell
signtool sign /fd SHA256 /tr http://timestamp.digicert.com /td SHA256 MyApp.msix
```

Expected result: Windows shows the package as signed.

## Step 4: Install and test

- Double-click the `.msix` file
- Follow the installer prompts
- Launch the app from Start Menu

<!-- TODO: add screenshot: ../../images/windows/msix-install-prompt.png -->

## Step 5: Optional AppInstaller for updates

Create a `.appinstaller` file that points to your `.msix` URL and version. Host both on a web server or internal share.

Expected result: Windows can auto-update the app.

> **Note:** Replace `https://example.com/` with your actual hosting URL before deploying.

### Example AppInstaller File

```xml
<?xml version="1.0" encoding="utf-8"?>
<AppInstaller
   xmlns="http://schemas.microsoft.com/appx/appinstaller/2017/2"
   Uri="https://example.com/MyApp.appinstaller"
   Version="1.0.0.0">

   <MainPackage
      Name="MyCompany.MyApp"
      Publisher="CN=YourCompany"
      Version="1.0.0.0"
      Uri="https://example.com/MyApp.msix" />
</AppInstaller>
```

## CI/CD with GitHub Actions

Automate MSIX builds so every version tag produces a signed package ready to distribute or upload to the Store.

Create `.github/workflows/build-msix.yml` with this content:

```yaml
name: Build Windows MSIX

on:
  push:
    branches: [main]
    tags: ["v*.*.*"]
  pull_request:
    branches: [main]

jobs:
  build-msix:
    runs-on: windows-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies and PyInstaller
        shell: bash
        run: |
          pip install -r requirements.txt
          pip install pyinstaller

      - name: Extract version from git tag
        id: version
        shell: bash
        run: |
          if [[ "${GITHUB_REF}" == refs/tags/v* ]]; then
            echo "VERSION=${GITHUB_REF_NAME#v}" >> "$GITHUB_OUTPUT"
          else
            echo "VERSION=0.0.0-dev" >> "$GITHUB_OUTPUT"
          fi

      - name: Build EXE with PyInstaller
        shell: bash
        run: |
          pyinstaller MyApp.spec --noconfirm

      - name: Inject version into AppxManifest.xml
        shell: bash
        run: |
          VERSION="${{ steps.version.outputs.VERSION }}"
          # MSIX Version must have 4 parts (e.g. 1.2.3.0); append .0 if only 3 parts
          MSIX_VERSION="${VERSION}.0"
          sed -i "s/Version=\"[^\"]*\"/Version=\"${MSIX_VERSION}\"/" dist/MyApp/AppxManifest.xml

      - name: Build MSIX package with makeappx
        shell: bash
        run: |
          # /nv skips manifest validation so unsigned builds do not fail in CI
          makeappx pack /d dist/MyApp /p MyApp.msix /nv

      - name: Sign MSIX (tag pushes only)
        if: startsWith(github.ref, 'refs/tags/')
        shell: bash
        env:
          CERTIFICATE_BASE64: ${{ secrets.WINDOWS_CERTIFICATE_BASE64 }}
          CERTIFICATE_PASSWORD: ${{ secrets.WINDOWS_CERTIFICATE_PASSWORD }}
        run: |
          echo "$CERTIFICATE_BASE64" | base64 --decode > certificate.pfx
          signtool sign /fd SHA256 /f certificate.pfx /p "$CERTIFICATE_PASSWORD" \
            /tr http://timestamp.digicert.com /td SHA256 \
            MyApp.msix
          rm certificate.pfx

      - name: Upload MSIX artifact
        uses: actions/upload-artifact@v4
        with:
          name: MyApp-msix-${{ github.ref_name }}
          path: MyApp.msix
          retention-days: 30
```

### Required Secrets

Add these two secrets under Settings → Secrets and variables → Actions:

| Secret name | Value |
|---|---|
| `WINDOWS_CERTIFICATE_BASE64` | Your `.pfx` file encoded as base64 |
| `WINDOWS_CERTIFICATE_PASSWORD` | The password for the `.pfx` file |

## Common Issues and Fixes

- Install fails: publisher name does not match signing certificate
- Missing icons: check the `Assets` folder paths in the manifest
- App crashes: ensure `_internal` files are included
- MSIX Packaging Tool fails: verify it can access your EXE output folder

## Final Checklist

- Folder-based build exists
- Manifest and assets added (including `Square150x150Logo.png`)
- MSIX builds successfully
- Package is signed
- App launches after install
- GitHub Actions workflow committed to `.github/workflows/`
- Required secrets added to GitHub repository settings
- Build passes end-to-end on a tag push
- Signed artifact downloaded from Actions and tested on a clean machine
- Screenshot placeholders replaced with real screenshots in `images/windows/`
