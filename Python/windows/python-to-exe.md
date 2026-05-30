# Python to EXE (Windows) - Ultra-Detailed Beginner Guide

This guide walks you through creating a Windows EXE from a Python app using PyInstaller. It is written for beginners and includes detailed steps, expected results, and troubleshooting. It covers both folder-based EXE (recommended for installers) and single-file EXE.

## Related Guides (Packaging Order)

- Start here for all Windows packaging
- Next step for MSI: See [Python/windows/python-to-msi.md](Python/windows/python-to-msi.md)
- Optional next step for MSIX: See [Python/windows/python-to-msix.md](Python/windows/python-to-msix.md)

## What You Will Build

- A Windows EXE that runs your Python app without needing Python installed
- Optionally, a single-file EXE for easy sharing

## Who This Is For

- Beginners packaging their first Python app for Windows
- Teams that want a repeatable, readable build process

## Glossary (Quick)

- EXE: Windows executable file
- PyInstaller: tool that bundles Python + your code into an EXE
- Spec file: a build recipe PyInstaller uses

## Prerequisites

- Windows 10/11
- Python 3.10 or newer installed
- Your app runs locally with `python main.py`
- A terminal (PowerShell)

## Recommended Project Layout

```
project/
  main.py
  src/
  data/
  requirements.txt
  packaging/
    app.ico
    version_info.txt
```

## Step 1: Open a terminal in your project folder

Open PowerShell and `cd` into your project folder.

Expected result: `Get-Location` shows your project folder.

Screenshot placeholder: PowerShell open in project folder

## Step 2: Create and activate a virtual environment

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

Expected result: your prompt starts with `(.venv)`.

Screenshot placeholder: venv activated in PowerShell

## Step 3: Install dependencies and PyInstaller

```powershell
pip install -r requirements.txt
pip install pyinstaller
```

Expected result: PyInstaller installs without errors.

## Step 4: Create a spec file (recommended)

The spec file is a build recipe you can edit later.

```powershell
pyinstaller --name MyApp --specpath . --noconfirm .\main.py
```

Expected result: `MyApp.spec` appears in your project folder.

Screenshot placeholder: spec file visible in folder

## Step 5: Edit the spec file for data files and imports

Open `MyApp.spec` and update these fields if needed:

```python
datas=[('data', 'data')]
hiddenimports=['pkgutil', 'importlib']
```

Explanation:

- `datas` bundles non-code files like images or JSON
- `hiddenimports` fixes modules that PyInstaller misses

## Step 6: Build a folder-based EXE (recommended for MSI)

```powershell
pyinstaller MyApp.spec --noconfirm
```

Expected output:

- `dist\MyApp\MyApp.exe`
- `dist\MyApp\_internal\` folder

Screenshot placeholder: dist folder showing MyApp.exe and _internal

## Step 7: Build a single-file EXE (optional)

If you want one file, run:

```powershell
pyinstaller --onefile --name MyApp .\main.py
```

Expected output:

- `dist\MyApp.exe`

## Step 8: Add icon and version info (optional but recommended)

Add this to your spec file in the `EXE` section:

```python
exe = EXE(
    ...,
    icon='packaging\\app.ico',
    version='packaging\\version_info.txt',
)
```

Expected result: your EXE shows a custom icon and version in Properties.

## Step 9: Test the EXE on a clean machine

- Copy the EXE to a clean VM or another PC
- Run it and test key features
- Verify bundled files load correctly

Screenshot placeholder: EXE running on a clean VM

## Step 10: Sign the EXE (production requirement)

```powershell
signtool sign /fd SHA256 /tr http://timestamp.digicert.com /td SHA256 dist\MyApp.exe
```

Expected result: EXE properties show a valid digital signature.

## Common Issues and Fixes

- EXE crashes on start: add missing modules to `hiddenimports`
- Data file missing: add to `datas` and use correct relative paths
- Antivirus warning: avoid UPX and sign the EXE

## Final Checklist

- App runs with `python main.py`
- EXE runs on a clean machine
- All data files are bundled
- EXE is signed and versioned
- Build steps are repeatable
