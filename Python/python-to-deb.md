# Python to DEB (Debian/Ubuntu) - Ultra-Detailed Beginner Guide

This guide shows how to create a `.deb` package the standard Debian way. It is written for beginners and uses simple files and steps.

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

```bash
sudo apt update
sudo apt install -y build-essential debhelper devscripts dh-python lintian
```

Expected result: tools like `dpkg-buildpackage` and `lintian` are available.

## Step 2: Create Debian packaging folder

In your project root, create `debian/` with these files:

```
debian/
   control
   rules
   install
   changelog
```

Screenshot placeholder: debian folder structure

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
- `Architecture` should match your build

## Step 4: Fill in `debian/rules`

```makefile
#!/usr/bin/make -f
%:
	dh $@
```

Make it executable:

```bash
chmod +x debian/rules
```

## Step 5: Fill in `debian/install`

This tells Debian where to put files:

```text
dist/MyApp/* usr/lib/myapp/
```

## Step 6: Fill in `debian/changelog`

```text
myapp (1.0.0-1) stable; urgency=medium

   * Initial release

 -- Your Team <dev@example.com>  Sat, 30 May 2026 10:00:00 +0000
```

## Step 7: Build your app output

Use PyInstaller or your own build system to produce a folder-based output in `dist/`:

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
pip install pyinstaller
pyinstaller --name MyApp ./main.py
```

Expected output:

- `dist/MyApp/MyApp.exe` and its dependency files

## Step 8: Build the package

```bash
dpkg-buildpackage -us -uc
```

Expected output: the `.deb` file appears one directory above your project root.

Screenshot placeholder: .deb file in parent folder

## Step 9: Validate with lintian

```bash
lintian ../myapp_1.0.0-1_amd64.deb
```

Fix any warnings before release.

## Step 10: Install and test

```bash
sudo dpkg -i ../myapp_1.0.0-1_amd64.deb
```

Test your app and then uninstall:

```bash
sudo dpkg -r myapp
```

## Common Issues and Fixes

- Build fails: missing Build-Depends in `debian/control`
- Files not installed: check `debian/install` paths
- Lintian warnings: follow Debian policy and fix each warning

## Final Checklist

- `debian/` folder exists and is correct
- `.deb` builds without errors
- Lintian warnings addressed
- Install and uninstall works on a clean machine
