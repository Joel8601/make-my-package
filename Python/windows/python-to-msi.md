# Python to MSI (Windows) — Ultra-Detailed Beginner Guide

This guide converts a folder-based PyInstaller build into an MSI using WiX Toolset. It is written for beginners and includes small, safe steps with explanations and expected results.

## When to Use This Format

Use the MSI format when you need Start Menu shortcuts, a clean uninstall experience, or enterprise deployment via Group Policy or SCCM. MSI is the standard format for traditional Windows software distribution.

## Time Estimate

~30 minutes for a first build (includes WiX setup and writing the installer definition).

> **ARM64 note:** This guide produces x86_64 MSIs only. To build for ARM64 Windows, run the full build chain (PyInstaller + WiX) on an ARM64 Windows machine. GitHub Actions provides `windows-11-arm` hosted runners for this purpose. Change `InstallerVersion` in the `Package` element if targeting ARM64-only machines.

## Related Guides (Packaging Order)

- Required first: Build the EXE in [python-to-exe.md](python-to-exe.md)
- Optional next step for MSIX: See [python-to-msix.md](python-to-msix.md)

## Purpose

This guide shows how to convert a folder-based EXE build into a professional MSI installer using WiX Toolset.

## Who Benefits

- Software engineers distributing Windows apps via MSI
- Teams that need Start Menu/Desktop shortcuts and clean uninstall

## What You Will Build

- A Windows MSI installer that installs your app into Program Files
- Start Menu and Desktop shortcuts

## Who This Is For

- Beginners who already have a working EXE build
- Teams that want a professional Windows installer

## Prerequisites

- Windows 10/11
- PyInstaller build output (folder-based)
- WiX Toolset v3.x installed (`candle`, `light`, `heat`)
- PowerShell

### Install WiX Toolset (Beginner Steps)

1. Download WiX Toolset v3.x from the official site:
  - https://wixtoolset.org/releases/
2. Run the installer and finish setup.
3. Add WiX `bin` folder to your PATH:
  - Default path: `C:\Program Files (x86)\WiX Toolset v3.14\bin`
4. Verify in PowerShell:

<!-- Confirms that the WiX tools are installed and available on your PATH -->
```powershell
candle -?
light -?
heat -?
```

Expected result: each command prints help text.

## Concepts (Quick)

- `.wxs` is the WiX XML file that describes your installer
- A `Component` installs a file or folder
- `ComponentGroup` is a collection of components
- `heat` generates components from a folder (for dependencies)

## Step 1: Create a folder-based EXE

MSI installers install folders, not single-file EXEs. Build a folder-based EXE:

<!-- Builds the app into a dist\MyApp folder that the MSI will install -->
```powershell
pyinstaller --name MyApp .\main.py
```

Expected output (two common layouts):

- Layout A (newer PyInstaller):
  - `dist\MyApp\MyApp.exe`
  - `dist\MyApp\_internal\` folder
- Layout B (older PyInstaller):
  - `dist\MyApp\MyApp.exe`
  - dependency files next to the EXE

<!-- TODO: add screenshot: ../../images/windows/dist-folder-msi.png -->

## Step 2: Create a WiX project folder

Create a folder like `packaging\msi` and place your WiX files there:

```text
packaging\msi\
  installer.wxs
  HarvestedFiles.wxs
```

## Step 3: Generate GUIDs

Open PowerShell and run:

<!-- Generates a new random GUID — run this 3 to 5 times and save the results -->
```powershell
[guid]::NewGuid()
```

Generate 3 to 5 GUIDs and save them somewhere handy. You will use them for:

- `UpgradeCode` (stable across releases — never change this)
- Component GUIDs (unique per component)

<!-- TODO: add screenshot: ../../images/windows/powershell-guids.png -->

## Step 4: Create the main WiX file

Create `installer.wxs` and paste this template:

> **Note:** Replace the `UpgradeCode` and `Guid` attribute values with real GUIDs generated in Step 3. Do not reuse GUIDs between components.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Wix xmlns="http://schemas.microsoft.com/wix/2006/wi">
  <Product Id="*" Name="MyApp" Language="1033" Version="1.0.0" Manufacturer="YourCompany" UpgradeCode="YOUR-UPGRADE-CODE-GUID">
    <Package InstallerVersion="500" Compressed="yes" InstallScope="perMachine" />
    <MediaTemplate />

    <!-- MajorUpgrade handles uninstalling old versions automatically on upgrade.
         Without this element, installing a new version leaves the old one behind. -->
    <MajorUpgrade DowngradeErrorMessage="A newer version of MyApp is already installed." />

    <Icon Id="AppIcon" SourceFile="packaging\\app.ico" />
    <Property Id="ARPPRODUCTICON" Value="AppIcon" />

    <Directory Id="TARGETDIR" Name="SourceDir">
      <Directory Id="ProgramFilesFolder">
        <Directory Id="INSTALLDIR" Name="MyApp">
          <Component Id="MainExecutable" Guid="YOUR-MAIN-EXECUTABLE-GUID">
            <File Id="AppExe" Source="dist\MyApp\MyApp.exe" KeyPath="yes" />
          </Component>
        </Directory>
      </Directory>
      <Directory Id="ProgramMenuFolder">
        <Directory Id="StartMenuFolder" Name="MyApp">
          <Component Id="StartMenuShortcut" Guid="YOUR-STARTMENU-SHORTCUT-GUID">
            <Shortcut Id="StartMenuShortcut" Name="MyApp" Description="Launch MyApp" Target="[INSTALLDIR]MyApp.exe" WorkingDirectory="INSTALLDIR" Advertise="no" />
            <!-- RemoveFolder on a custom StartMenu subfolder is correct and required
                 so the subfolder is cleaned up on uninstall -->
            <RemoveFolder Id="RemoveStartMenuFolder" On="uninstall" />
            <RegistryValue Root="HKCU" Key="Software\MyApp" Name="StartMenuShortcut" Type="integer" Value="1" KeyPath="yes" />
          </Component>
        </Directory>
      </Directory>
      <Directory Id="DesktopFolder">
        <!-- Note: RemoveFolder is intentionally absent here.
             DesktopFolder is a standard system folder — never remove it on uninstall.
             Including RemoveFolder on DesktopFolder causes ICE64 validation errors. -->
        <Component Id="DesktopShortcut" Guid="YOUR-DESKTOP-SHORTCUT-GUID">
          <Shortcut Id="DesktopShortcut" Name="MyApp" Description="Launch MyApp from Desktop" Target="[INSTALLDIR]MyApp.exe" WorkingDirectory="INSTALLDIR" Advertise="no" Icon="AppIcon" />
          <RegistryValue Root="HKCU" Key="Software\MyApp" Name="DesktopShortcut" Type="integer" Value="1" KeyPath="yes" />
        </Component>
      </Directory>
    </Directory>

    <Feature Id="MainFeature" Title="MyApp" Level="1">
      <ComponentRef Id="MainExecutable" />
      <ComponentRef Id="StartMenuShortcut" />
      <ComponentRef Id="DesktopShortcut" />
      <ComponentGroupRef Id="InternalComponentGroup" />
    </Feature>

    <UI>
      <UIRef Id="WixUI_Minimal" />
      <UIRef Id="WixUI_ErrorProgressText" />
      <Property Id="WIXUI_WELCOMEFINISHPAGE" Value="1" />
    </UI>
  </Product>
</Wix>
```

> **Why MajorUpgrade:** Without the `MajorUpgrade` element, installing a new version of your MSI will not remove the old one. Users end up with two entries in Add/Remove Programs. The `MajorUpgrade` element tells Windows Installer to uninstall the old version automatically before installing the new one.

<!-- TODO: add screenshot: ../../images/windows/installer-wxs-editor.png -->

## Step 5: Harvest the dependency folder

Run `heat` to generate component entries for all dependency files in `_internal`:

<!-- Scans the _internal folder and writes a WiX component file for every file found -->
```powershell
heat dir dist\MyApp\_internal -cg InternalComponentGroup -dr INSTALLDIR -gg -g1 -sfrag -sreg -srd -out packaging\msi\HarvestedFiles.wxs
```

**What each flag does:**

| Flag | Meaning |
|------|---------|
| `-cg InternalComponentGroup` | Names the component group so `installer.wxs` can reference it |
| `-dr INSTALLDIR` | Sets the root install directory for harvested files |
| `-gg` | Generates GUIDs for every component automatically |
| `-g1` | Uses a stable GUID algorithm so GUIDs do not change between builds for the same file |
| `-sfrag` | Suppresses wrapping components in a `Fragment` element (avoids conflicts) |
| `-sreg` | Suppresses auto-generated registry keys |
| `-srd` | Suppresses the root directory element (lets `installer.wxs` own it) |

If you do not have an `_internal` folder, point `heat` at `dist\MyApp` instead:

<!-- Use this variant if your PyInstaller output is an older layout without _internal -->
```powershell
heat dir dist\MyApp -cg InternalComponentGroup -dr INSTALLDIR -gg -g1 -sfrag -sreg -srd -out packaging\msi\HarvestedFiles.wxs
```

### What the Harvested File Does

`HarvestedFiles.wxs` contains many `Component` entries for DLLs, images, and other dependencies. It keeps your MSI in sync with the files you built.

Expected result: `HarvestedFiles.wxs` is created.

If paths look like `SourceDir\...`, edit them to use the actual path, for example `dist\MyApp\_internal`.

## Step 6: Build the MSI

From the folder containing your `.wxs` files:

<!-- Compiles the WiX source files, then links them into a final MSI -->
```powershell
candle packaging\msi\installer.wxs packaging\msi\HarvestedFiles.wxs
light installer.wixobj HarvestedFiles.wixobj -o MyApp.msi -ext WixUIExtension
```

Expected output: `MyApp.msi` appears in the current directory.

<!-- TODO: add screenshot: ../../images/windows/msi-created.png -->

## Step 7: Install and verify

1. Double-click the MSI.
2. Install to the default path.
3. Check Start Menu and Desktop shortcuts.

<!-- TODO: add screenshot: ../../images/windows/msi-install-wizard.png -->

## Step 8: Log installations (recommended)

<!-- Runs the MSI installer and writes a detailed log for troubleshooting -->
```powershell
msiexec /i MyApp.msi /l*v install.log
```

If installation fails, open `install.log` and look for errors.

## Step 9: Sign the MSI (production)

> **EV certificate requirement:** Windows SmartScreen builds reputation based on your signing certificate. A standard OV (Organization Validation) certificate will sign the MSI successfully, but SmartScreen may still show a warning. An EV (Extended Validation) certificate grants immediate reputation and is strongly recommended for any MSI distributed outside your organization.

<!-- Signs the MSI with a trusted certificate to prevent SmartScreen warnings -->
```powershell
signtool sign /fd SHA256 /tr http://timestamp.digicert.com /td SHA256 MyApp.msi
```

Expected result: MSI properties show a valid digital signature.

## Upgrade Rules (Beginner-Friendly)

- Keep `UpgradeCode` the same for all versions of the same product line
- Increase `Version` for each release
- MSI creates a new `ProductCode` automatically when `Id="*"`
- `MajorUpgrade` handles old version removal automatically

## CI/CD with GitHub Actions

Automate MSI builds so every version tag produces a signed MSI ready to distribute.

Create `.github/workflows/build-msi.yml` with this content:

```yaml
name: Build Windows MSI

on:
  push:
    branches: [main]
    tags: ["v*.*.*"]
  pull_request:
    branches: [main]

jobs:
  build-msi:
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

      - name: Install WiX Toolset v3
        shell: bash
        run: |
          # Chocolatey is pre-installed on GitHub-hosted Windows runners
          choco install wixtoolset -y
          # Locate the WiX bin directory dynamically in case the patch version varies
          WIX_BIN=$(ls -d "/c/Program Files (x86)/WiX Toolset"*/bin 2>/dev/null | head -1)
          echo "$WIX_BIN" >> "$GITHUB_PATH"

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

      - name: Inject version into installer.wxs
        shell: bash
        run: |
          VERSION="${{ steps.version.outputs.VERSION }}"
          sed -i "s/Version=\"[0-9][^\"]*\"/Version=\"${VERSION}\"/" packaging/msi/installer.wxs

      - name: Harvest dependency files with heat
        shell: bash
        run: |
          heat dir dist/MyApp/_internal \
            -cg InternalComponentGroup \
            -dr INSTALLDIR \
            -gg -g1 -sfrag -sreg -srd \
            -out packaging/msi/HarvestedFiles.wxs

      - name: Compile WiX source files with candle
        shell: bash
        run: |
          candle packaging/msi/installer.wxs packaging/msi/HarvestedFiles.wxs

      - name: Link object files into MSI with light
        shell: bash
        run: |
          light installer.wixobj HarvestedFiles.wixobj \
            -o MyApp.msi \
            -ext WixUIExtension

      - name: Sign MSI (tag pushes only)
        if: startsWith(github.ref, 'refs/tags/')
        shell: bash
        env:
          CERTIFICATE_BASE64: ${{ secrets.WINDOWS_CERTIFICATE_BASE64 }}
          CERTIFICATE_PASSWORD: ${{ secrets.WINDOWS_CERTIFICATE_PASSWORD }}
        run: |
          echo "$CERTIFICATE_BASE64" | base64 --decode > certificate.pfx
          signtool sign /fd SHA256 /f certificate.pfx /p "$CERTIFICATE_PASSWORD" \
            /tr http://timestamp.digicert.com /td SHA256 \
            MyApp.msi
          rm certificate.pfx

      - name: Upload MSI artifact
        uses: actions/upload-artifact@v4
        with:
          name: MyApp-msi-${{ github.ref_name }}
          path: MyApp.msi
          retention-days: 30
```

### Required Secrets

Add these two secrets under Settings → Secrets and variables → Actions:

| Secret name | Value |
|---|---|
| `WINDOWS_CERTIFICATE_BASE64` | Your `.pfx` file encoded as base64 |
| `WINDOWS_CERTIFICATE_PASSWORD` | The password for the `.pfx` file |

## Common Issues and Fixes

- `candle` not found: add WiX `bin` folder to PATH
- Missing files after install: ensure `HarvestedFiles.wxs` is referenced in `Feature`
- No shortcuts: check `StartMenuShortcut` component and KeyPath
- ICE64 validation error: remove `RemoveFolder` from any component inside `DesktopFolder`

## Final Checklist

- Folder-based EXE build exists
- MSI installs on a clean machine
- Start Menu shortcut works
- MSI is signed
- Install log has no errors
- GitHub Actions workflow committed to `.github/workflows/`
- Required secrets added to GitHub repository settings
- Build passes end-to-end on a tag push
- Signed artifact downloaded from Actions and tested on a clean machine
- Screenshot placeholders replaced with real screenshots in `images/windows/`
