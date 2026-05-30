# Python to RPM (RHEL/Fedora/CentOS) - Ultra-Detailed Beginner Guide

This guide shows how to build a `.rpm` package using a spec file. It is written for beginners and includes step-by-step explanations.

## What You Will Build

- An `.rpm` package that installs your app under `/usr/lib/myapp`
- A command entry point `myapp`

## Who This Is For

- Beginners packaging for Fedora/RHEL/CentOS/Rocky
- Teams distributing internal RPMs

## Prerequisites

- A Linux machine with `dnf`
- Your app already runs with `python main.py`

## Step 1: Install RPM tools

```bash
sudo dnf install -y rpm-build rpmlint gcc make
```

Expected result: `rpmbuild` and `rpmlint` are available.

## Step 2: Create the RPM build tree

```bash
rpmdev-setuptree
```

Expected result: `~/rpmbuild/` folders exist.

Screenshot placeholder: rpmbuild folder tree

## Step 3: Build your app output

Use PyInstaller or your build system to generate a folder-based build:

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
pip install pyinstaller
pyinstaller --name MyApp ./main.py
```

Expected output:

- `dist/MyApp/` with the EXE and supporting files

## Step 4: Create a spec file

Create `~/rpmbuild/SPECS/myapp.spec` with this content:

```
Name:           myapp
Version:        1.0.0
Release:        1%{?dist}
Summary:        MyApp description

License:        Proprietary
BuildArch:      x86_64

%description
MyApp description

%prep

%build

%install
mkdir -p %{buildroot}/usr/lib/myapp
cp -a dist/MyApp/* %{buildroot}/usr/lib/myapp/
ln -s /usr/lib/myapp/MyApp.exe %{buildroot}/usr/bin/myapp

%files
/usr/lib/myapp
/usr/bin/myapp

%changelog
* Sat May 30 2026 Your Team <dev@example.com> - 1.0.0-1
- Initial release
```

Explanation:

- `%install` copies your build output into the RPM buildroot
- `%files` defines what gets installed

## Step 5: Build the RPM

```bash
rpmbuild -ba ~/rpmbuild/SPECS/myapp.spec
```

Expected output:

- `~/rpmbuild/RPMS/x86_64/myapp-1.0.0-1.x86_64.rpm`

## Step 6: Validate with rpmlint

```bash
rpmlint ~/rpmbuild/RPMS/x86_64/myapp-1.0.0-1.x86_64.rpm
```

Fix warnings if possible before release.

## Step 7: Install and test

```bash
sudo dnf install -y ~/rpmbuild/RPMS/x86_64/myapp-1.0.0-1.x86_64.rpm
```

Run your app and then uninstall:

```bash
sudo dnf remove -y myapp
```

## Optional: Sign the RPM

```bash
rpm --addsign ~/rpmbuild/RPMS/x86_64/myapp-1.0.0-1.x86_64.rpm
```

## Common Issues and Fixes

- Missing files after install: check `%files` section
- Build errors: confirm `%install` paths exist
- Dependencies: add `Requires:` in the spec

## Final Checklist

- Spec file exists and is correct
- RPM builds without errors
- rpmlint warnings addressed
- Install and uninstall works
