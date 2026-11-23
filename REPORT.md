# eSim-2.5 Installation Issues and Fixes on Ubuntu 25.04

## 1. Environment Details

**Operating System**
- Distribution: Ubuntu 25.04 (Plucky Puffin)
- VERSION_ID: `25.04`
- Kernel: `Linux 6.14.0-37-generic (x86_64)`
- Architecture: 64-bit

**Virtual Machine**
- Hypervisor: Oracle VirtualBox 7.2.4
- Host OS: Windows 11

**eSim Details**
- Version: **2.5**
- Installation Path: `/home/ubuntu/Downloads/eSim-2.5`
- Source: Downloaded `eSim-2.5.zip` from - https://esim.fossee.in/downloads and extracted

**Python Version**
- Python 3.13.3

**User Information**
- Username: `ubuntu`
- Home Directory: `/home/ubuntu`

**Installation Command Used**
```bash
cd /home/ubuntu/Downloads/eSim-2.5
chmod +x install-eSim.sh
./install-eSim.sh --install
```

### Issue 1: OS Version Check & Whitelist Failure on Ubuntu 25.04

- **Component:** `install-eSim.sh`
- **Severity:** High (Installation aborts immediately)
- **Status:** Resolved

#### Error:
```text
Detected Ubuntu Version: 
Unsupported Ubuntu version: 25.04 ()
```

#### Root Cause:
1. **Regex Mismatch:** `get_ubuntu_version()` used regex `\d+\.\d+\.\d+`, which requires 3 numbers (e.g., `22.04.4`). Ubuntu 25.04 only has two numbers (`25.04`), leaving the detected version string blank.
2. **Missing OS Branch:** The `case $VERSION_ID in` block only recognized older Ubuntu releases (up to 24.04) and lacked an entry for `25.04`, defaulting to exit.

#### Steps Taken & Fix:
1. Created `install-eSim-scripts/install-eSim-25.04.sh` using `install-eSim-24.04.sh` as a template:
   ```bash
   cp install-eSim-scripts/install-eSim-24.04.sh install-eSim-scripts/install-eSim-25.04.sh
   ```
2. In `install-eSim.sh`:
   - Updated regex in `get_ubuntu_version()` to: `\d+\.\d+(\.\d+)?`
   - Added `"25.04")` case in `run_version_script()` to point to `install-eSim-25.04.sh`.

#### Verification:
Running `./install-eSim.sh --install` now outputs `Detected Ubuntu Version: 25.04` and successfully launches the 25.04 script.

### Issue 2: KiCad 6.0 PPA Repository Incompatibility on Ubuntu 25.04

- **Component:** `install-eSim-scripts/install-eSim-25.04.sh` (`installKicad()`)
- **Severity:** High (Installation halts during repository update)
- **Status:** Resolved

#### Error:
```text
Err: https://ppa.launchpadcontent.net/kicad/kicad-6.0-releases/ubuntu plucky Release
E: The repository 'https://ppa.launchpadcontent.net/kicad/kicad-6.0-releases/ubuntu plucky Release' does not have a Release file.
Aborting Installation...
```

#### Root Cause:
Inside `installKicad()`, the script checked `if [[ "$ubuntu_version" == "24.04" ]]`. Since the operating system version was `25.04`, the check evaluated to false and defaulted to the `else` block, which set `kicadppa="kicad/kicad-6.0-releases"`. Ubuntu 25.04 (`plucky`) does not have Release files for the deprecated KiCad 6.0 PPA on Launchpad, causing `apt-get update` to abort securely.

#### Steps Taken & Fix:
1. Removed the broken PPA list from the system sources:
   ```bash
   sudo rm -f /etc/apt/sources.list.d/*kicad-6*
   sudo apt update
   ```
2. Updated the OS check in `installKicad()` to support Ubuntu 25.04 alongside 24.04:
   ```bash
   if [[ "$ubuntu_version" == "24.04" || "$ubuntu_version" == "25.04" ]]; then
   ```

#### Verification:
`apt update` ran without repository errors, allowing the script to properly target and resolve KiCad 8.0.

### Issue 3: Premature Installer Termination on Pre-installed KiCad

- **Component:** `install-eSim-scripts/install-eSim-25.04.sh` (`installKicad()`)
- **Severity:** Medium (Script terminates early if KiCad 8.0 is already present)
- **Status:** Resolved

#### Error:
When KiCad 8.0 is already present on the system, the installer prints:
```text
KiCad 8.0 is already installed.
```
and abruptly exits back to the terminal prompt without proceeding to the remaining installation steps (NGHDL, PDK, and launcher creation).

#### Root Cause:
Inside the `installKicad()` function, the script executed `exit 0` instead of `return 0` upon finding KiCad 8.0 already installed. This killed the parent process prematurely.

#### Steps Taken & Fix:
Replaced `exit 0` with `return 0` inside `installKicad()` so the script can proceed to downstream installation tasks.

#### Verification:
Re-running `./install-eSim.sh --install` prints `KiCad 8.0 is already installed.` and smoothly continues to the library copy and NGHDL installation steps.

### Issue 4: NGHDL Sub-Installer OS Version Check Failure

- **Component:** `nghdl/install-nghdl.sh`
- **Severity:** High (Halts NGHDL and digital simulation subsystem setup)
- **Status:** Resolved

#### Error:
```text
inflating: nghdl/src/ghdlserver/Vhpi_Package.vhdl
Detected Ubuntu Version: 
Unsupported Ubuntu version: 25.04 ()
```

#### Root Cause:
1. NGHDL is bundled as an internal archive (`nghdl.zip`) with its own independent installer (`install-nghdl.sh`).
2. Like the main installer, `nghdl/install-nghdl.sh` strictly matched 3-part version strings and had no case statement branch for Ubuntu `25.04`, causing it to exit immediately.
3. Furthermore, `install-eSim-25.04.sh` unconditionally ran `unzip -o nghdl.zip`, which would overwrite any in-place bug fixes on subsequent test runs.

#### Steps Taken & Fix:
1. Prevented unconditional archive overwrite in `install-eSim-25.04.sh` by commenting out `unzip -o nghdl.zip` (or adding directory existence checks).
2. Created `nghdl/install-nghdl-scripts/install-nghdl-25.04.sh` using the 24.04 script as a baseline.
3. Updated `nghdl/install-nghdl.sh` regex to `\d+\.\d+(\.\d+)?` and added the `"25.04")` case block.

#### Verification:
Running `./install-eSim.sh --install` successfully detected `25.04` inside the NGHDL installer and executed `install-nghdl-25.04.sh`.


### Issue 5: Deprecated GTK2 Canberra Module on Ubuntu 25.04

- **Component:** `nghdl/install-nghdl-scripts/install-nghdl-25.04.sh` (`installDependency()`)
- **Severity:** Medium (Package installation failure halts the installer)
- **Status:** Resolved

#### Error:
```text
Installing Gtk Canberra modules...........................
Package libcanberra-gtk-module is not available, but is referred to by another package.
This may mean that the package is missing, has been obsoleted, or is only available from another source

E: Package 'libcanberra-gtk-module' has no installation candidate
Aborting Installation...
```

#### Root Cause:
`libcanberra-gtk-module` is a legacy module for GTK 2. Ubuntu 25.04 (Plucky Puffin) completely retired all legacy GTK 2 packages from its official repositories, replacing them with GTK 3 equivalents.

#### Steps Taken & Fix:
In `nghdl/install-nghdl-scripts/install-nghdl-25.04.sh`, replaced the obsolete `libcanberra-gtk-module` with the modern `libcanberra-gtk3-module`:
```bash
sudo apt install -y libcanberra-gtk3-module
```

#### Verification:
`apt install` successfully located and installed `libcanberra-gtk3-module` without package lookup errors.

### Issue 6: GHDL 4.1.0 Configuration Failure with LLVM 20.1.2 on Ubuntu 25.04

- **Component:** `nghdl/install-nghdl-scripts/install-nghdl-25.04.sh` (`installDependency()`, `installGHDL()`)
- **Severity:** Critical (Core VHDL simulation backend fails to compile)
- **Status:** Resolved

#### Error:
```text
Changing directory to ghdl-4.1.0 installation
Configuring ghdl-4.1.0 build as per requirements
gcc (Ubuntu 14.2.0-19ubuntu2) 14.2.0
Use full IEEE library
Build machine is: x86_64-linux-gnu
Unhandled version llvm 20.1.2

Error! Kindly resolve above error(s) and try again.
Aborting Installation...
```

#### Root Cause:
The installer installed unversioned `llvm` and `llvm-dev`, which defaults to **LLVM 20.1.2** on Ubuntu 25.04. The bundled GHDL 4.1.0 build system only supports LLVM toolchains up to version 18. When GHDL queried `/usr/bin/llvm-config`, it detected the unsupported 20.1.2 release and aborted configuration with `Unhandled version llvm 20.1.2`.

#### Steps Taken & Fix:
1. Pinned the compiler toolchain to the stable, supported LLVM 18 release:
   ```bash
   sudo apt install -y llvm-18 llvm-18-dev clang-18
   ```
2. In `nghdl/install-nghdl-scripts/install-nghdl-25.04.sh`:
   - Updated `installDependency()` to install `llvm-18 llvm-18-dev clang-18` instead of generic `llvm`.
   - Updated `installGHDL()` to explicitly point to the LLVM 18 configuration binary:
     ```bash
     ./configure --with-llvm-config=/usr/bin/llvm-config-18
     ```

#### Verification:
GHDL configure script successfully detected LLVM 18, generating the makefiles and compiling the GHDL LLVM backend cleanly.
