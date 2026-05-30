# Python to MSIX (Windows) - Ultra-Detailed Beginner Guide

This guide packages a Python app into MSIX using beginner-friendly steps. It includes a GUI path and a CLI path, plus explanations and expected results.

## Related Guides (Packaging Order)

- Required first: Build the EXE in [Python/windows/python-to-exe.md](Python/windows/python-to-exe.md)
- If you already have an MSI workflow: See [Python/windows/python-to-msi.md](Python/windows/python-to-msi.md)

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

## Step 1: Build a folder-based EXE

MSIX packages folder outputs. Build your EXE like this:

```powershell
pyinstaller --name MyApp .\main.py
```

Expected output:

- `dist\MyApp\MyApp.exe`
- `dist\MyApp\_internal\` folder

Screenshot placeholder: dist folder showing MyApp.exe and _internal

## Step 2: Add an MSIX manifest and assets

Create a folder structure:

```
dist\MyApp\
   AppxManifest.xml
   Assets\
      StoreLogo.png
      Square44x44Logo.png
```

Your `AppxManifest.xml` defines identity, version, and display name. Use a template from Microsoft docs and update it for your app.

Screenshot placeholder: AppxManifest.xml open in editor

## Option A: MSIX Packaging Tool (GUI)

1. Open MSIX Packaging Tool.
2. Click "Application package".
3. Select "Create package on this computer".
4. Point it to your EXE output folder.
5. Fill in Publisher, Package Name, Version.
6. Save the `.msix` file.

Screenshot placeholder: MSIX Packaging Tool wizard

## Option B: CLI Build (Repeatable)

From the project root:

```powershell
makeappx pack /d dist\MyApp /p MyApp.msix
```

Expected result: `MyApp.msix` created.

## Step 3: Sign the MSIX

```powershell
signtool sign /fd SHA256 /tr http://timestamp.digicert.com /td SHA256 MyApp.msix
```

Expected result: Windows shows the package as signed.

## Step 4: Install and test

- Double-click the `.msix` file
- Follow the installer prompts
- Launch the app from Start Menu

Screenshot placeholder: MSIX install prompt

## Step 5: Optional AppInstaller for updates

Create a `.appinstaller` file that points to your `.msix` URL and version. Host both on a web server or internal share.

Expected result: Windows can auto-update the app.

## Common Issues and Fixes

- Install fails: publisher name does not match signing certificate
- Missing icons: check the `Assets` folder paths in the manifest
- App crashes: ensure `_internal` files are included

## Final Checklist

- Folder-based build exists
- Manifest and assets added
- MSIX builds successfully
- Package is signed
- App launches after install
