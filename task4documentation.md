
Issue 1: Syntax Error in install-eSim.sh During Execution

Issue Description

Upon navigating to the eSim/Ubuntu directory, making the script executable via chmod +x install-eSim.sh, and running ./install-eSim.sh --install, execution failed immediately with the following Bash error output:
```bash
./install-eSim.sh: line 70: syntax error near unexpected token `}'
./install-eSim.sh: line 70: `}'
```
Root Cause Analysis

In the original install-eSim.sh script, inside the run_version_script() function, an if block was opened at line 59 (if [[ -f "$SCRIPT" ]]; then). However, the block was never terminated with a matching fi keyword before closing the function with } at line 71. The Bash parser encountered the closing function brace } while still expecting a closing fi statement, raising a parse error.

Resolution Applied

Inserted the missing fi statement at line 62 immediately following the execution of bash "$SCRIPT" "$ARGUMENT", properly closing the conditional block prior to writing configuration keys.

Modified Code Snippet

<img width="922" height="1017" alt="image" src="https://github.com/user-attachments/assets/e9e95bed-69e6-4de3-9f02-d00a4aba3448" />


also verified with `bash -n install-eSim.sh `

note
The script attempts to write configuration variables ($config_dir, $eSim_Home) inside run_version_script() before those environment variables are initialized or declared globally. To prevent blank configuration file entries, execution logic writing to $config_file should ideally be encapsulated in a dedicated initConfig() function called after environment variables are exported. Although just corrected the fi thing leaving all as it is in this particular step

Issue 2 & 3

<img width="922" height="1017" alt="image" src="https://github.com/user-attachments/assets/57527eb8-df39-43b1-b371-6f700d345feb" />


Issue 2: ./install-eSim.sh: line 64: /: Is a directory

Root Cause:

Because $config_dir and $config_file were empty, the script evaluated echo "[eSim]" >> $config_dir/$config_file as echo "[eSim]" >> /, which tried to treat root directory / as a output file!

Modified Code Snippet

```
# Run the script if found
    if [[ -f "$SCRIPT" ]]; then
        echo "Running script: $SCRIPT $ARGUMENT"
        bash "$SCRIPT" "$ARGUMENT"
    else
        echo "Installer script not found: $SCRIPT"
        exit 1
    fi
}
```

Issue 3: tar: Cannot open: No such file or directory

Root Cause:
The sub-script ran tar -xJf library/kicadLibrary.tar.xz, but it executed inside ~/Downloads/eSim/Ubuntu instead of the top-level ~/Downloads/eSim folder where the library/ folder actually lives.

Fix: You need to either run the installation script from the main eSim/ directory or fix the relative file path inside the sub-script.

fix:
```
ln -s ../library library
bash -n install-eSim.sh
```



issue 4 : script stopped after [kicad 8.0 already installed]

<img width="922" height="1017" alt="image" src="https://github.com/user-attachments/assets/046137be-4996-4464-a65c-e56b5e22c3cf" />



root cause

Looking at the execution sequence, the script ran:

1.installDependency (Python packages, virtual environment, and system utilities)

2.installKicad (Detected KiCad 8.0)
It exited right after KiCad because the function calls for the remaining setup steps—such as extracting the custom KiCad library, NGHDL, SKY130 PDK, and setting up the desktop entry—were halted by an uncaught exit code, or placed after a missing function call inside install-eSim-25.04.sh

current code

<img width="922" height="1017" alt="image" src="https://github.com/user-attachments/assets/87a8d512-10cb-4e5c-a9e4-6ab8ae10b9e5" />


removed exit 0 from line 388

updated code
<img width="922" height="1017" alt="image" src="https://github.com/user-attachments/assets/e599701d-3a44-4e63-8371-30d8a63bd2c5" />


but after doing this it still stopped at same place

cause - When line 124 triggers, exit 0 shuts down the entire script process immediately. Because line 124 executes during installKicad, the flow stops right there, and functions 395–398 (copyKicadLibrary, installNghdl, installSky130Pdk, and createDesktopStartScript) never get called.

updated code ( changed exit 0 to return 0 in line 136)
 <img width="922" height="1017" alt="image" src="https://github.com/user-attachments/assets/d5e1a319-03e2-4951-9a53-932fb6e9588c" />



issue 5 : The installer made it past KiCad and reached copyKicadLibrary, but failed when trying to extract library/kicadLibrary.tar.xz

<img width="922" height="1017" alt="image" src="https://github.com/user-attachments/assets/799e1056-5713-4cea-9b1a-3003e5c9dddd" />


root cause
The function copyKicadLibrary expects the compressed archive at library/kicadLibrary.tar.xz relative to where you execute ./install-eSim.sh. Because you ran the installer from ~/Downloads/eSim/Ubuntu/ instead of the root eSim/ directory, tar cannot locate the file.

fix step:

checked with these commands

<img width="492" height="112" alt="image" src="https://github.com/user-attachments/assets/ebe27bb8-cea4-4cdd-b5cd-e7cdfbafd2e1" />


finding where kicadLibrary.tar.xz actually is:

` find ~/ -iname "*kicad*" -o -iname "*.tar.xz" 2>/dev/null `

missing files were in ~/Downloads/eSim/MacOS/library/

When i ran ln -s ../library library inside the Ubuntu directory in previous steps, it created a relative link pointing to ~/Downloads/eSim/library. At that point: The installer ran, saw the link, but stopped early at KiCad 8.0 is already installed. due to the exit 0 inside installKicad. When the script reached copyKicadLibrary, it called rm -rf kicadLibrary to clean up extracted temporary files. That cleanup step deleted the actual library folder (or its target at ~/Downloads/eSim/library), leaving ../library as a broken symlink pointing to a non-existent location!

run
```
rm library
ln -s ~/Downloads/eSim/MacOS/library library
```

issue 6: failed on installNghdl

<img width="922" height="1017" alt="image" src="https://github.com/user-attachments/assets/870f9f7d-533d-4d35-a7a1-cd9d7e99bf71" />


root cause

The install-eSim.sh script expected nghdl.zip to be present in the eSim/Ubuntu directory, but the archive was located in the eSim/MacOS directory

fix
```
rm nghdl.zip
cp /home/ubuntu/Downloads/eSim/MacOS/nghdl.zip .
```

issue 7 : unsupported Ubuntu version : 25.04 ()

<img width="464" height="512" alt="image" src="https://github.com/user-attachments/assets/4330e30e-3eef-4d1d-a2f9-82a90498ff11" />


<img width="922" height="1017" alt="image" src="https://github.com/user-attachments/assets/b43131cf-3128-4106-8453-07ea3bf0ba9f" />


root cause

The NGHDL installer script did not have a case for Ubuntu 25.04, so the system was detected as unsupported. Additionally, install-eSim.sh re-extracted nghdl.zip on every run, overwriting any changes made directly to nghdl/install-nghdl.sh.

fix

A 25.04 case was added to nghdl/install-nghdl.sh to use the Ubuntu 24.04 NGHDL installer script. Since the main installer overwrote this modification during extraction, nghdl.zip was manually extracted, the script was patched, and the modified script was updated back into the archive using:

```
unzip -o nghdl.zip
zip -u nghdl.zip nghdl/install-nghdl.sh
```

modified code

<img width="922" height="1017" alt="image" src="https://github.com/user-attachments/assets/3c06a689-36e5-421a-9d40-7ef6a5c572d6" />


issue 8 : Package 'libcanberra-gtk-module' has no installation candidate

<img width="922" height="1017" alt="image" src="https://github.com/user-attachments/assets/fe967cfe-a289-460a-8607-17d936a592d4" />


Root Cause

The NGHDL installer attempted to install libcanberra-gtk-module, which was unavailable in the Ubuntu 25.04.

fix

The dependency section of install-nghdl-24.04.sh was adjusted to remove the unavailable package. It previously contained
` sudo apt install -y libcanberra-gtk-module libcanberra-gtk3-module `

updated code

<img width="922" height="1017" alt="image" src="https://github.com/user-attachments/assets/994f1da4-cbd5-4fd4-9ab4-9320ba99379c" />



then
` zip -u nghdl.zip nghdl/install-nghdl-scripts/install-nghdl-24.04.sh `


issue 9 : Unhandled version llvm 20.1.2

<img width="464" height="512" alt="image" src="https://github.com/user-attachments/assets/3dced727-cb60-44e6-8b76-9547a2f76784" />


root cause

When install-nghdl-24.04.sh runs configure for GHDL, it passes LLVM support without forcing a supported LLVM version. GHDL 4.1.0 expects LLVM versions up to 18 or 19.

Fix

` sudo apt install -y llvm-18 llvm-18-dev clang-18 `

updated code - Changed /usr/bin/llvm-config to /usr/bin/llvm-config-18 on that ./configure line

<img width="922" height="1017" alt="image" src="https://github.com/user-attachments/assets/0bdfb280-5f11-4117-bd07-cbf47414aaf5" />


then - ` zip -u nghdl.zip nghdl/install-nghdl-scripts/install-nghdl-24.04.sh `


issue 10 : cp: cannot stat 'images/logo.png': No such file or directory

<img width="922" height="1017" alt="image" src="https://github.com/user-attachments/assets/8a05c92f-f972-44bd-821a-65f4e9b8cb53" />


fix
` mkdir -p images && touch images/logo.png `


issue

<img width="952" height="1018" alt="image" src="https://github.com/user-attachments/assets/dcf31490-51ad-4fc9-bef3-fdde54341643" />


The error mv: cannot overwrite '/home/ubuntu/nghdl-simulator/nghdl-simulator-source': Directory not empty happens because a previous installation run created that directory, and mv cannot move or rename a new folder over an existing, non-empty folder.

fix

` rm -rf /home/ubuntu/nghdl-simulator/nghdl-simulator-source `



eSim installed successfully
<img width="922" height="1017" alt="image" src="https://github.com/user-attachments/assets/51720e06-dbbd-451c-9b66-c80d10370903" />
