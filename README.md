# Burpsuite-Professional — Code Review Plan & Findings

> This document replaces the original README with a structured code review plan, full findings, and arm64 macOS deployment requirements.

## Repository overview

This repository packages and installs Burp Suite Professional across Linux, macOS, Windows, and NixOS. It is not a web application; it is a distribution/installer project composed of:

- **Nix/NixOS packaging**: `flake.nix`, `flake.lock`, `default.nix`
- **Linux installer/maintenance**: `install.sh`, `update.sh`
- **macOS installer**: `install_macos.sh`
- **Windows installer**: `install.ps1`
- **Helper/documentation**: `help.sh`, `README.md`
- **Release automation**: `.github/workflows/burp-pro.yml`
- **Binary assets**: `loader.jar`, `launcher.jpg`, `burp_suite.icns`, `burp_suite.ico`

The core function of every installer is to download the official Burp Suite Professional JAR, place the existing `loader.jar` Java agent on the classpath, and expose a launcher command.

---

## Baseline architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     User-facing entrypoints                  │
│  README.md  │  help.sh  │  curl | bash / powershell / nix   │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
   ┌─────────┐          ┌───────────┐         ┌──────────┐
   │ install │          │ install_  │         │ install  │
   │ .sh     │          │ macos.sh  │         │ .ps1     │
   │ update  │          │           │         │          │
   │ .sh     │          │           │         │          │
   └────┬────┘          └─────┬─────┘         └────┬─────┘
        │                     │                    │
        ▼                     ▼                    ▼
   ┌─────────┐           ┌─────────┐        ┌─────────────┐
   │ apt +   │           │ brew +  │        │ Oracle JDK  │
   │ wget    │           │ curl +  │        │ + JRE       │
   │         │           │ jpackage│        │             │
   └────┬────┘           └────┬────┘        └──────┬──────┘
        │                     │                    │
        ▼                     ▼                    ▼
   ┌────────────────────────────────────────────────────────┐
   │              Download burpsuite_pro_*.jar             │
   │              Attach loader.jar as -javaagent          │
   │              Launch with --add-opens / -noverify     │
   └────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┴─────────────────────┐
        │                                           │
        ▼                                           ▼
   ┌──────────┐                              ┌──────────────┐
   │ default  │                              │ burp-pro.yml │
   │ .nix     │                              │  (CI release)│
   │ flake.nix│                              │              │
   └──────────┘                              └──────────────┘
```

---

## Review plan

The review is organized into six logical groups that map to the components above. Each group is evaluated against the high-confidence bug patterns: null/undefined safety, injection/command construction, resource handling, missing error handling, dead/unused code, wrong-variable/shadowing, type/ordering assumptions, and security-relevant input validation.

### Group 1 — Nix packaging correctness
**Files**: `flake.nix`, `flake.lock`, `default.nix`

- Verify the nixpkgs input is pinned and the lock file matches.
- Confirm the Burp JAR `version`, `urls`, and SRI `hash` are consistent and still resolvable.
- Check that `buildFHSEnv` options and `runScript` quoting are correct.
- Look for hardcoded version drift between `default.nix` and the rest of the repo.
- Validate `meta.homepage`, `meta.license`, and `mainProgram` fields.

### Group 2 — Linux installer security & correctness
**Files**: `install.sh`, `update.sh`

- Check `sudo` usage and whether non-root execution paths fail safely.
- Inspect command construction with `$(pwd)` for quoting/escaping issues.
- Verify version consistency between `install.sh` and `update.sh`.
- Confirm error handling (`set -e`, exit codes) and launcher creation.
- Validate `/bin/burpsuitepro` installation assumption.

### Group 3 — macOS installer security & correctness
**Files**: `install_macos.sh`

- Verify shebang, dependency checks, and Java/JDK availability.
- Inspect `curl` download URL and version variable.
- Check `java` and `jpackage` invocation and quoting.
- Inspect the generated `burp` wrapper for `$(pwd)` portability issues.
- Validate app bundle paths and icon references.

### Group 4 — Windows installer security & correctness
**Files**: `install.ps1`

- Check Oracle JDK-21 and JRE-8 download URLs and bundle IDs.
- Verify Java detection logic (deprecated `Win32_Product` usage).
- Inspect Burp.bat command construction for missing spaces or quoting bugs.
- Validate VBS wrapper path handling.
- Confirm `loader.jar` fallback download source and integrity checks.

### Group 5 — Release workflow security & correctness
**Files**: `.github/workflows/burp-pro.yml`

- Verify unauthenticated download from PortSwigger and lack of hash verification.
- Check checksum step semantics (compute vs. verify).
- Inspect release asset deletion scope and GitHub API usage.
- Validate runner architecture choice (`ubuntu-24.04-arm`).
- Confirm secrets handling (`GITHUB_TOKEN`).

### Group 6 — Docs/helper consistency
**Files**: `README.md`, `help.sh`

- Cross-check install instructions against actual script behavior.
- Verify version numbers, filenames, and command names in docs.
- Confirm `help.sh` arguments and color handling are safe.
- Check README macOS section for file-name typos.

---

## Full findings summary

### [P0] install_macos.sh downloads wrong JAR version for the macOS title
**File**: `install_macos.sh`, line 5

```bash
version=2025
url="https://portswigger.net/burp/releases/download?product=pro&type=Jar"
curl -L "$url" -o "burpsuite_pro_v$version.jar"
```

The script names the file `burpsuite_pro_v2025.jar` but the URL requests the **latest** JAR, currently 2026.x. The rest of the script uses `burpsuite_pro_v$version.jar` for the launcher and `jpackage` bundle, so it works by coincidence. However, the on-disk filename is misleading, and the README references `burpsuite_pro_v2025.5.6.jar`. This version inconsistency will break the `burp` shortcut if the user re-runs the script from a directory where the filename no longer matches. Treat as a deployment defect.

### [P0] install_macos.sh silently fails if Java or jpackage is missing
**File**: `install_macos.sh`, lines 17-29

The script runs `java`, `jpackage`, and `cp` without checking prerequisites. On a clean arm64 Mac, the system `/usr/bin/java` stub prints an error and exits non-zero, but the script does not `set -e`, so failures are ignored. `jpackage` is part of the JDK and is not installed by the script, so the app bundle step will silently fail on a JRE-only system.

**arm64 macOS requirement**: a full JDK (not just JRE) that includes `jpackage` is required. The README mentions `openjdk@17`, which is sufficient because Homebrew's `openjdk@17` package is a JDK and includes `jpackage`.

### [P1] macOS `burp` wrapper hardcodes `$(pwd)` and breaks when moved or run from another directory
**File**: `install_macos.sh`, lines 19-29

```bash
java ... -javaagent:$(pwd)/loader.jar ... -jar $(pwd)/burpsuite_pro_v$version.jar &
```

The generated `burp` script uses `$(pwd)` to locate `loader.jar` and the Burp JAR. If the user follows the README and copies `burp` to `/usr/local/bin/burp`, running it from anywhere other than the original clone directory will fail with "Unable to access jarfile". This contradicts the README's claim of global use.

### [P1] install.ps1 uses deprecated Win32_Product WMI query
**File**: `install.ps1`, lines 5 and 14

```powershell
$jdk21 = Get-WmiObject -Class Win32_Product ...
$jre8 = Get-WmiObject -Class Win32_Product ...
```

Microsoft documentation warns that `Win32_Product` triggers a consistency check of installed packages and is slow/unreliable. Using it in an installer can cause performance issues and false negatives. The script should query registry paths or use `Get-ItemProperty` under `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall`.

### [P1] Windows Burp.bat has a missing space between Java `--add-opens` flags
**File**: `install.ps1`, line 40

```powershell
$path = "java --add-opens=java.desktop/javax.swing=ALL-UNNAMED--add-opens=java.base/java.lang=ALL-UNNAMED ..."
```

There is no space between `ALL-UNNAMED` and `--add-opens=java.base/java.lang`. This produces an invalid JVM argument and Burp will fail to start from `Burp.bat`.

### [P1] Linux install.sh and update.sh download from different sources with mismatched versions
**Files**: `install.sh`, line 11; `update.sh`, line 11

`install.sh` uses `version=2026`. `update.sh` uses `version=2025`. Both download from the same GitHub release URL and both write `/bin/burpsuitepro`. A user running `install.sh` then later `update.sh` will downgrade from 2026 to 2025 without warning. The versions should be centralized in one source of truth.

### [P1] Linux scripts write to `/bin` and assume root without `set -e`
**Files**: `install.sh`, lines 14-18; `update.sh`, lines 14-18

```bash
cp burpsuitepro /bin/burpsuitepro
(./burpsuitepro)
```

`cp` to `/bin` requires root, but the scripts only use `sudo` for `apt`. If run as a normal user, `cp` fails silently (no `set -e`) and the script then tries to execute `./burpsuitepro`, which may succeed locally but leaves the system launcher missing.

### [P2] Workflow downloads unverified JAR and only computes checksums after the fact
**File**: `.github/workflows/burp-pro.yml`, lines 24-28

```yaml
axel -o burpsuite_pro_v2026.jar https://portswigger.net/burp/releases/download?product=pro&type=Jar
```

The workflow downloads the latest JAR without a pinned hash, then computes MD5/SHA1/SHA256/SHA512 of whatever was downloaded. There is no comparison against a known-good hash, so a corrupted or malicious artifact would be released as-is. This is a supply-chain integrity gap.

### [P2] default.nix version/hash mismatch with project branding
**File**: `default.nix`, line 4

```nix
version = "2025.1.1";
```

The README and workflow advertise "v2026-latest", `install.sh` uses `2026`, and `install_macos.sh`/`update.sh` use `2025`, but the Nix derivation pins `2025.1.1` with a fixed SRI hash. The package will not fetch the latest version and may fail when PortSwigger removes that exact URL or when the hash no longer matches a redirected download.

### [P2] flake.nix only supports x86_64-linux
**File**: `flake.nix`, line 12

```nix
system = "x86_64-linux";
```

There is no `aarch64-darwin` (arm64 macOS) output. Users on Apple Silicon cannot install via the Nix flake even though the rest of the project provides a macOS install script. For arm64 macOS, the only supported path is `install_macos.sh` + Homebrew `openjdk@17`.

### [P2] install_macos.sh lacks shebang and dependency checks
**File**: `install_macos.sh`, line 1

The file begins with `git clone ...` instead of `#!/bin/bash`. While `curl ... | bash` will usually run it under bash, execution via `chmod +x install_macos.sh; ./install_macos.sh` will fail because the kernel cannot determine the interpreter.

### [P3] README macOS instructions reference nonexistent `installmacos.sh`
**File**: `README.md`, macOS section

The README says:

> The `installmacos.sh` script creates a `burp` script...

But the actual file in the repo is `install_macos.sh`. This typo will confuse users trying to follow the local-install path.

### [P3] README references `burpsuite_pro_v2025.5.6.jar` that does not match any script
**File**: `README.md`, macOS Notes section

The README says to run `burp` from the directory containing `loader.jar` and `burpsuite_pro_v2025.5.6.jar`, but no script downloads that filename. `install_macos.sh` writes `burpsuite_pro_v2025.jar`.

---

## arm64 macOS deployment requirements

Based on `install_macos.sh` and local environment verification, a clean arm64 Mac needs the following before deployment:

1. **Homebrew**
   ```bash
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   ```
   The script assumes `brew` is available but does not install it.

2. **A full JDK** (not just a JRE), because `jpackage` is required for the app bundle step:
   ```bash
   brew install git openjdk@17
   ```
   The system `/usr/bin/java` stub is insufficient and will cause silent failures.

3. **Command-line developer tools or Xcode** may be required for `jpackage` app-image signing/notarization on macOS. The script does not handle this.

4. **Run from a stable directory** instead of piping directly to bash if you want the generated `burp` wrapper and `.app` bundle to remain usable, because both rely on `$(pwd)`.

---

## Overall summary

The repository is a collection of platform installers with several real correctness issues. The highest-priority defects are version drift across Linux/macOS/Nix scripts, the missing-space bug in the Windows batch file, and the macOS script's silent failures and non-portable `$(pwd)` launcher. For arm64 macOS specifically, the install path is unsupported by the Nix flake and requires manual Homebrew JDK setup.

---

## Original project attribution

- Loader.jar: [h3110w0r1d-y/BurpLoaderKeygen](https://github.com/h3110w0r1d-y/BurpLoaderKeygen)
- Script foundation: [cyb3rzest/Burp-Suite-Pro](https://github.com/cyb3rzest/Burp-Suite-Pro)
