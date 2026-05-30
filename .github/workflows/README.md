# GitHub Actions CI/CD Workflows

This folder contains six automated GitHub Actions workflows designed to package your Python application into distributable native formats for Windows, Linux, and Android.

## Workflow Summary

| Workflow File | Target Format | Runner OS | Build Toolchain | Release Output | Setup Steps |
|---|---|---|---|---|---|
| [`build-exe.yml`](build-exe.yml) | Windows Standalone EXE | `windows-latest` | PyInstaller | `dist/MyApp/` folder | [Configure Steps ➔](#step-2-customize-the-yaml-files-for-your-project) |
| [`build-msi.yml`](build-msi.yml) | Windows MSI Installer | `windows-latest` | WiX Toolset v3 + PyInstaller | `MyApp.msi` installer | [Configure Steps ➔](#step-2-customize-the-yaml-files-for-your-project) |
| [`build-msix.yml`](build-msix.yml) | Windows MSIX Package | `windows-latest` | makeappx + PyInstaller | `MyApp.msix` package | [Configure Steps ➔](#step-2-customize-the-yaml-files-for-your-project) |
| [`build-deb.yml`](build-deb.yml) | Debian/Ubuntu DEB | `ubuntu-22.04` | dpkg-buildpackage + lintian | `*.deb` package | [Configure Steps ➔](#step-2-customize-the-yaml-files-for-your-project) |
| [`build-rpm.yml`](build-rpm.yml) | RHEL/Fedora/CentOS RPM | `ubuntu-22.04` (Fedora container) | rpmbuild + rpmlint | `*.rpm` package | [Configure Steps ➔](#step-2-customize-the-yaml-files-for-your-project) |
| [`build-apk.yml`](build-apk.yml) | Android APK | `ubuntu-22.04` | Buildozer + JDK 17 | Signed/aligned release APK | [Configure Steps ➔](#step-2-customize-the-yaml-files-for-your-project) |


---

## How It Works

### Triggers
Each workflow is configured to run automatically:
1. **Pull Requests / Pushes to `main`**: Runs the full build and package pipeline, linting, and outputs an **unsigned/debug developer build**.
2. **Version Tag Pushes (`v*.*.*`)**: Triggers the **production release pipeline** which decodes code-signing certificates/keystores from GitHub Secrets, signs/aligns the packages, and uploads production-ready assets.

### Dynamic Versioning
- Workflows dynamically parse your Git tag name (e.g., `v1.2.3` becomes version `1.2.3` or `1.2.3.0` for MSIX).
- On development branch builds, a fallback version (e.g., `0.0.0-dev` or `1.0.0-dev`) is used.

---

## Required Secrets Setup

To enable automated production signing on version tags, add the following secrets to your GitHub repository under **Settings → Secrets and variables → Actions**:

### Windows Signing Secrets (EXE, MSI, MSIX)

These secrets are shared by the three Windows pipelines:

| Secret Name | Value Description |
|---|---|
| `WINDOWS_CERTIFICATE_BASE64` | Your `.pfx` code-signing certificate file encoded as a Base64 string. |
| `WINDOWS_CERTIFICATE_PASSWORD` | The password required to unlock and decode the `.pfx` certificate file. |

*To generate the Base64 value of your certificate:*
- **Linux/macOS**: `base64 -i my-certificate.pfx` (or use `-w 0` on Linux to strip newlines).
- **Windows (PowerShell)**: `[Convert]::ToBase64String([IO.File]::ReadAllBytes("my-certificate.pfx"))`

---

### Android Signing Secrets (APK)

These secrets are used by the Android pipeline to align and sign release APKs:

| Secret Name | Value Description |
|---|---|
| `KEYSTORE_BASE64` | Your `.jks` or `.keystore` signing key file encoded as a Base64 string. |
| `KEYSTORE_PASSWORD` | The master password to access the keystore file. |
| `KEY_ALIAS` | The alias given to the private key when created. |
| `KEY_PASSWORD` | The password for the specific private key alias. |

*To generate the Base64 value of your keystore:*
- **Linux/macOS**: `base64 -i my-release-key.jks`
- **Windows (PowerShell)**: `[Convert]::ToBase64String([IO.File]::ReadAllBytes("my-release-key.jks"))`

---

---

## How to Set Up and Adapt Workflows for Your Project

Follow this exhaustive, beginner-friendly guide to adapt, configure, and execute automated native packaging pipelines for your own Python repository.

---

### Step 1: Prepare Your Repository Structure

To support automated building, your project directory on GitHub must be laid out correctly.

1. **Create the Workflow Directory**: 
   In the root of your project, create the following path and copy the desired `.yml` files into it:
   ```text
   your-project/
     .github/
       workflows/
         build-exe.yml
         build-msi.yml
         ...
   ```

2. **Add a Proper `.gitignore`**:
   Ensure you do not accidentally commit temporary build folders, local binary artifacts, or highly sensitive signing keys. In your root directory, create a `.gitignore` file with the following contents:
   ```text
   # Python Environment
   .venv/
   __pycache__/
   *.pyc

   # PyInstaller Local Builds
   build/
   dist/
   *.spec

   # WiX Toolset Temporary Objects
   *.wixobj
   *.wixpdb
   packaging/msi/HarvestedFiles.wxs

   # Build Logs
   *.log

   # Code Signing & Keys (NEVER COMMIT THESE)
   *.pfx
   *.jks
   *.keystore
   ```

---

### Step 2: Customize the Workflow Files

Each workflow is pre-configured with generic template values (such as script path `main.py` and application name `MyApp`). You must update these inside the `.yml` files to match your project.

#### A. Customizing Windows Workflows (`build-exe.yml`, `build-msi.yml`, `build-msix.yml`)
1. **Entry Script Filename**:
   If your main execution script is named `run.py` or `app.py` instead of `main.py`, find the PyInstaller compilation line in the workflow and change it:
   - *Default*: `pyinstaller MyApp.spec --noconfirm` (or `pyinstaller --name MyApp .\main.py`)
   - *Customized*: `pyinstaller --name MyCoolApp --noconfirm .\run.py`
2. **Application Name Search-and-Replace**:
   Open the workflow file and search for the string `MyApp`. Replace it with your application's actual name (e.g., `MyCoolApp` or `TextEditor`). This modifies:
   - PyInstaller output target directory paths: `dist/MyApp/` ➔ `dist/MyCoolApp/`
   - WiX compilation files: `packaging/msi/HarvestedFiles.wxs`
   - Dynamically generated artifact zip uploads.
3. **MSIX Manifest & Assets (MSIX only)**:
   - In `build-msix.yml`, update `dist/MyApp/AppxManifest.xml` references to match your folder structure.
   - Ensure you replace the package identity parameters inside `AppxManifest.xml` (`Publisher="CN=YourCompany"`) to match the exact Certificate Subject Name of your `.pfx` certificate.

#### B. Customizing Linux Workflows (`build-deb.yml`, `build-rpm.yml`)
1. **Debian Control File Settings**:
   - In `build-deb.yml`, update references to your Debian control paths.
   - If your package relies on specific python modules, ensure they are defined in your project's `debian/control` file under `Depends:`.
2. **RPM Spec File Settings**:
   - In `build-rpm.yml`, look for the line:
     ```yaml
     cp packaging/rpm/myapp.spec ~/rpmbuild/SPECS/myapp.spec
     ```
     Change `myapp.spec` to match the exact name of your project spec file.
   - Verify that your spec file contains `%global project_dir %{getenv:PROJECT_DIR}` so the workspace path compiles seamlessly inside the Fedora container runner.

#### C. Customizing the Android Workflow (`build-apk.yml`)
1. **Mobile Configuration Requirements**:
   - Open `buildozer.spec` in your project root.
   - Update `title = YourAppName` and ensure `package.name = wrapp` is configured.
2. **Dynamic Version Injections**:
   - The workflow uses `sed` to search for and inject versions automatically:
     - `sed -i "s/^version = .*/version = ${VERSION}/" buildozer.spec`
     Ensure that your `buildozer.spec` contains a standard `version = ...` line so this regex targets it accurately.

---

### Step 3: Generate Code-Signing Assets

To build trusted production packages that users can install without operating system warnings (like Windows SmartScreen or Android Play Protect warnings), your code must be signed.

#### A. Generating a Self-Signed Windows Certificate (For Testing)
If you do not have a paid production EV (Extended Validation) certificate from a public CA, you can create a self-signed code-signing certificate for testing purposes.
1. Open **PowerShell (as Administrator)** on a Windows machine.
2. Run this command to generate the certificate inside your certificate store:
   ```powershell
   $cert = New-SelfSignedCertificate -Type CodeSigningCert -Subject "CN=MyTestingCompany" -KeyUsage DigitalSignature -FriendlyName "My Actions Signer" -NotAfter (Get-Date).AddYears(3)
   ```
3. Run this command to export that certificate to a `.pfx` file on your Desktop:
   ```powershell
   $pwd = ConvertTo-SecureString "YourSecurePasswordHere" -AsPlainText -Force
   Export-PfxCertificate -Cert $cert -FilePath "$home\Desktop\my-certificate.pfx" -Password $pwd
   ```

#### B. Generating an Android Release Keystore
To sign release APKs:
1. Open a terminal (Linux, macOS, or WSL).
2. Run `keytool` to generate an RSA keystore file:
   ```bash
   keytool -genkeypair -v \
     -keystore my-release-key.jks \
     -keyalg RSA -keysize 2048 \
     -validity 10000 \
     -alias mykey
   ```
3. Enter your keystore password, name, and organization details when prompted. This creates `my-release-key.jks` in the current folder.

---

### Step 4: Encode Your Keys to Base64

GitHub Repository Secrets cannot accept binary files (like `.pfx` or `.jks`). You must convert them to flat text strings using Base64 encoding.

Run the corresponding command for your operating system:

#### If you are on Linux or macOS (WSL / Bash):
```bash
# For Windows PFX Certificate:
base64 -w 0 my-certificate.pfx > certificate_base64.txt

# For Android JKS Keystore:
base64 -w 0 my-release-key.jks > keystore_base64.txt
```
*(Note: If you are on macOS, run the command without `-w 0`, like this: `base64 -i my-certificate.pfx > certificate_base64.txt`)*.

#### If you are on Windows (PowerShell):
```powershell
# For Windows PFX Certificate:
[Convert]::ToBase64String([IO.File]::ReadAllBytes("my-certificate.pfx")) | Out-File -FilePath certificate_base64.txt

# For Android JKS Keystore:
[Convert]::ToBase64String([IO.File]::ReadAllBytes("my-release-key.jks")) | Out-File -FilePath keystore_base64.txt
```

Open the generated `.txt` files in any text editor. Copy the long, continuous string of text to your clipboard.

---

### Step 5: Add Secrets to GitHub

To securely pass these credentials to your build runner without showing them in your public code, configure GitHub Actions Secrets.

1. Go to your repository home page on **GitHub.com**.
2. Click on the **Settings** tab (gear icon at the top right).
3. In the left navigation menu, expand **Secrets and variables** and click **Actions**.
4. Click the **New repository secret** button.
5. Add the secrets required by your workflow targets.

#### For Windows Pipelines (`build-exe.yml`, `build-msi.yml`, `build-msix.yml`):
- Create a secret named `WINDOWS_CERTIFICATE_BASE64` and paste the contents of `certificate_base64.txt`.
- Create a secret named `WINDOWS_CERTIFICATE_PASSWORD` and enter the raw password string you used to secure the `.pfx` file.

#### For the Android Pipeline (`build-apk.yml`):
- Create `KEYSTORE_BASE64` and paste the contents of `keystore_base64.txt`.
- Create `KEYSTORE_PASSWORD` with the master password of the keystore.
- Create `KEY_ALIAS` with your key alias string (e.g., `mykey`).
- Create `KEY_PASSWORD` with the alias private key password.

---

### Step 6: Trigger a Test Build (Development)

To verify that your compiler environment and dependencies build successfully without signing requirements:

1. Commit your customized workflow files and push them to your repository on any branch (or submit a Pull Request):
   ```bash
   git add .
   git commit -m "ci: set up native packaging workflows"
   git push origin main
   ```
2. Navigate to the **Actions** tab at the top of your GitHub repository.
3. Select your workflow name (e.g., **Build Windows MSI**) from the left-hand column.
4. Click on the running job to watch the progress logs. The runner will:
   - Initialize Python and compile dependencies.
   - Run PyInstaller or the appropriate bundler.
   - Skip the code-signing step automatically (which is normal for branch runs).
   - Compress the build output and upload an **unsigned developer artifact**.
5. Once the run shows a green checkmark, go to the **Summary** page, scroll to the bottom, and click the artifact name to download the package. Run it locally to make sure all data assets and modules load without errors.

---

### Step 7: Release a Signed Production Build

Once the test run is successful and you are ready to distribute a signed release version:

1. Create a version tag on your local machine matching the standard semantic versioning format (`v*.*.*`):
   ```bash
   git tag v1.0.0
   ```
2. Push the version tag to the remote repository on GitHub:
   ```bash
   git push origin v1.0.0
   ```
3. Go to the **Actions** tab on GitHub. You will see a new run starts.
4. The tag trigger will execute the production pipeline:
   - The runner extracts `1.0.0` dynamically and injects it into your packaging configurations (MSIX, Spec, Spec files, and Control targets).
   - It reads your Base64 repository secrets, reconstructs the binary `.pfx` or `.jks` files directly in memory/workspace, runs `signtool sign` or `apksigner sign` to write digital signatures, and immediately deletes (`rm`) the certificate from the build runner storage.
5. Download the final signed artifact. 

---

### Step 8: Verify Your Signed Packages

Always test your output packages on clean machines before sharing them publicly.

#### Verifying a Windows Installer (EXE, MSI, MSIX)
1. Download the signed artifact.
2. Right-click the `.exe`, `.msi`, or `.msix` file and choose **Properties**.
3. Select the **Digital Signatures** tab.
4. Verify that your certificate name (e.g., `MyTestingCompany` or your production CA name) is listed and marked as a valid digital signature.

#### Verifying an Android APK
1. Ensure the Android SDK command line tools are installed on your machine.
2. Run `apksigner` to verify the signing block:
   ```bash
   apksigner verify --verbose bin/MyApp-release-signed.apk
   ```
3. Check that the output confirms the package is signed and lists the correct certificate fingerprints.

---

## Security Best Practices

1. **Ephemeral Decoding**: Workflows decode certificate and keystore files directly to disk inside the job's sandbox workspace, use them immediately for signing, and run `rm` to securely purge them from the runner environment before completion.
2. **Pull Request Isolation**: Code signing steps contain explicit conditional triggers (`if: startsWith(github.ref, 'refs/tags/')`) so that pull requests from external forks or branch pushes never run the signing steps or expose sensitive secrets.
3. **Artifact Retention**: Artifacts uploaded to GitHub Actions are set with a retention window of 30 days to optimize storage limits.


