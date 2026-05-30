# Python to MSI (Windows) - Ultra-Detailed Beginner Guide

This guide converts a folder-based PyInstaller build into an MSI using WiX Toolset. It is written for beginners and includes small, safe steps with explanations and expected results.

## Related Guides (Packaging Order)

- Required first: Build the EXE in [Python/windows/python-to-exe.md](Python/windows/python-to-exe.md)
- Optional next step for MSIX: See [Python/windows/python-to-msix.md](Python/windows/python-to-msix.md)

## Purpose

This guide shows how to convert a folder-based EXE build into a professional MSI installer using WiX Toolset.

## Who Benefits

- Software engineers distributing Windows apps via MSI
- Teams that need Start Menu/Desktop shortcuts and clean uninstall

## What You Will Build

- A Windows MSI installer that installs your app into Program Files
- Start Menu shortcut (optional)

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

Screenshot placeholder: dist folder showing MyApp.exe and _internal

## Step 2: Create a WiX project folder

Create a folder like `packaging\msi` and place your WiX files there:

```
packaging\msi\
  installer.wxs
  HarvestedFiles.wxs
```

## Step 3: Generate GUIDs

Open PowerShell and run:

```powershell
[guid]::NewGuid()
```

Generate 3 to 5 GUIDs and keep them in a note. You will use:

- `UpgradeCode` (stable across releases)
- Component GUIDs (unique per component)

Screenshot placeholder: PowerShell showing GUIDs

## Step 4: Create the main WiX file

Create `installer.wxs` and paste this template:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Wix xmlns="http://schemas.microsoft.com/wix/2006/wi">
  <Product Id="*" Name="MyApp" Language="1033" Version="1.0.0" Manufacturer="YourCompany" UpgradeCode="PUT-UPGRADECODE-GUID-HERE">
    <Package InstallerVersion="500" Compressed="yes" InstallScope="perMachine" />
    <MediaTemplate />

    <Icon Id="AppIcon" SourceFile="packaging\\app.ico" />
    <Property Id="ARPPRODUCTICON" Value="AppIcon" />

    <Directory Id="TARGETDIR" Name="SourceDir">
      <Directory Id="ProgramFilesFolder">
        <Directory Id="INSTALLDIR" Name="MyApp">
          <Component Id="MainExecutable" Guid="PUT-GUID-HERE">
            <File Id="AppExe" Source="dist\MyApp\MyApp.exe" KeyPath="yes" />
          </Component>
        </Directory>
      </Directory>
      <Directory Id="ProgramMenuFolder">
        <Directory Id="StartMenuFolder" Name="MyApp">
          <Component Id="StartMenuShortcut" Guid="PUT-GUID-HERE">
            <Shortcut Id="StartMenuShortcut" Name="MyApp" Description="Launch MyApp" Target="[INSTALLDIR]MyApp.exe" WorkingDirectory="INSTALLDIR" Advertise="no" />
            <RemoveFolder Id="RemoveStartMenuFolder" On="uninstall" />
            <RegistryValue Root="HKCU" Key="Software\MyApp" Name="StartMenuShortcut" Type="integer" Value="1" KeyPath="yes" />
          </Component>
        </Directory>
      </Directory>
      <Directory Id="DesktopFolder">
        <Component Id="DesktopShortcut" Guid="PUT-GUID-HERE">
          <Shortcut Id="DesktopShortcut" Name="MyApp" Description="Launch MyApp from Desktop" Target="[INSTALLDIR]MyApp.exe" WorkingDirectory="INSTALLDIR" Advertise="no" Icon="AppIcon" />
          <RemoveFolder Id="RemoveDesktopShortcut" On="uninstall" />
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

Replace the GUID placeholders with real GUIDs.

Screenshot placeholder: installer.wxs open in editor

## Step 5: Harvest the dependency folder

Run `heat` to include all dependency files:

```powershell
heat dir dist\MyApp\_internal -cg InternalComponentGroup -dr INSTALLDIR -out packaging\msi\HarvestedFiles.wxs
```

If you do not have an `_internal` folder, point `heat` at `dist\MyApp` instead:

```powershell
heat dir dist\MyApp -cg InternalComponentGroup -dr INSTALLDIR -out packaging\msi\HarvestedFiles.wxs
```

### What the Harvested File Does

`HarvestedFiles.wxs` contains many `Component` entries for DLLs, images, and other dependencies. It keeps your MSI in sync with the files you built.

Expected result: `HarvestedFiles.wxs` is created.

If paths look like `SourceDir\...`, edit them to use the actual path, for example `dist\MyApp\_internal`.

## Step 6: Build the MSI

From the folder containing your `.wxs` files:

```powershell
candle packaging\msi\installer.wxs packaging\msi\HarvestedFiles.wxs
light installer.wixobj HarvestedFiles.wixobj -o MyApp.msi -ext WixUIExtension
```

Expected output: `MyApp.msi` appears in the current directory.

Screenshot placeholder: MSI created in folder

## Step 7: Install and verify

1. Double-click the MSI.
2. Install to the default path.
3. Check Start Menu and Desktop shortcuts.

Screenshot placeholder: MSI install wizard

## Step 7: Test the MSI install

- Double-click the MSI
- Confirm install location
- Check Start Menu shortcut

Screenshot placeholder: MSI install wizard

## Step 8: Log installations (recommended)

```powershell
msiexec /i MyApp.msi /l*v install.log
```

If installation fails, open `install.log` and look for errors.

## Step 9: Generate GUIDs (quick tip)

Use this PowerShell command any time you need new GUIDs:

```powershell
[guid]::NewGuid()
```

## Step 10: Sign the MSI (production)

```powershell
signtool sign /fd SHA256 /tr http://timestamp.digicert.com /td SHA256 MyApp.msi
```

## Upgrade Rules (Beginner-Friendly)

- Keep `UpgradeCode` the same for all versions of the same product line
- Increase `Version` for each release
- MSI creates a new `ProductCode` automatically when `Id="*"`

## Common Issues and Fixes

- `candle` not found: add WiX `bin` folder to PATH
- Missing files after install: ensure `HarvestedFiles.wxs` is referenced in `Feature`
- No shortcuts: check `StartMenuShortcut` component and KeyPath

## Final Checklist

- Folder-based EXE build exists
- MSI installs on a clean machine
- Start Menu shortcut works
- MSI is signed
- Install log has no errors
