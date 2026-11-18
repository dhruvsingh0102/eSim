# Getting eSim 2.5 running on Ubuntu 25.04

FOSSEE Winter Internship 2026, Screening Task 4

The first installer stopped at the Ubuntu version check. Once I got past that, PyQt5 was the main blocker. The fixed installer completed on a clean Ubuntu 25.04 ARM64 VM, the eSim main window opened, and ngspice returned the expected result for a voltage-divider test.

## What I tested

I used the [official eSim 2.5 Linux ZIP](https://esim.fossee.in/downloads) for the application and bundled libraries. I used the [upstream installers branch at `a77d6b9`](https://github.com/FOSSEE/eSim/tree/a77d6b904c7bab275d819cf4d8cbb594802fe16d) for the installer scripts. The test VM ran in Lima 2.3.0 on an Apple M4 Mac, with two CPUs, 4 GiB of RAM, and a 40 GiB virtual disk. I saved a clean copy of the stopped VM disk and restored it before the final install.

## What broke and how I fixed it

### The installer could not reach the Ubuntu 25.04 script

The installer shipped in the ZIP reported `Unsupported Ubuntu version: 25.04` and exited with status 1. The ZIP only included scripts for Ubuntu 22.04, 23.04, and 24.04.

The upstream branch had a 25.04 script, but its main wrapper failed `bash -n` with `syntax error near unexpected token '}'` at line 70. An extra block of installer code interrupted an unfinished `if`. I restored the version-selection wrapper from upstream commit `f4cede1`. It now calls the 25.04 script and passes its exit status back to the terminal. It also prints the short Ubuntu version when the longer version string is unavailable. [Installer wrapper](Ubuntu/install-eSim.sh)

### PyQt5 was the main GUI blocker

The Ubuntu script installed PyQt5 with APT, but eSim's isolated Python environment could not import the system package. When the script ran `pip3 install PyQt5`, pip tried to build it from source on ARM64. That failed during package setup with a missing `qmake` error and stopped installation before the launcher was created.

I changed the script to install PyQt5, Matplotlib, and Watchdog through APT and create the venv with `--system-site-packages`. That lets eSim use Ubuntu's builds from inside the venv. The script still uses pip inside that venv for the HDL tools. [Ubuntu 25.04 installer](Ubuntu/install-eSim-scripts/install-eSim-25.04.sh)

### KiCad and desktop setup

- **KiCad was already installed.** The old script called `exit 0` inside the KiCad step, which ended the whole installation before launcher setup. The updated step returns to the installer so it can continue. On a clean system, the script installs KiCad 8.0.8 from Ubuntu's `universe` repository.
- **The symbol table went to the KiCad 6.0 folder.** It now goes into the KiCad 8.0 folder under `XDG_CONFIG_HOME`, with `~/.config` as the default. I removed the old uninstall commands that deleted KiCad PPA entries and the 6.0 config folder.
- **The minimal guest had no Desktop folder.** The installer now creates `~/Desktop` before copying the shortcut.

### The bundled NGHDL installer also rejected Ubuntu 25.04

Running the bundled NGHDL installer separately gave the same unsupported-version error. Its ZIP also lacks a 25.04 script, so it stopped before the LLVM step. The updated installer tries NGHDL when a matching script is present. Otherwise, or if that script fails, it warns that co-simulation is unavailable and installs Ubuntu's ngspice for basic circuit simulation.

## How to reproduce it

Use a clean Ubuntu 25.04 VM with Git, Python 3, and the `universe` repository enabled. Download the [eSim 2.5 Linux ZIP](https://esim.fossee.in/downloads). Its published SHA-256 is `b9860ae26f026930bae589520761f4a57b85d593a2e2cd1ac83d5e5db2f281ed`.

**To see the original failures**, extract the ZIP and run its installer from inside `eSim-2.5` with `bash ./install-eSim.sh --install`. That reproduces the version rejection. To reproduce the PyQt5 failure on ARM64, repeat on a fresh copy of the ZIP using the [working wrapper from `f4cede1`](https://github.com/FOSSEE/eSim/blob/f4cede1/Ubuntu/install-eSim.sh) and the [original Ubuntu 25.04 script from `a77d6b9`](https://github.com/FOSSEE/eSim/blob/a77d6b9/Ubuntu/install-eSim-scripts/install-eSim-25.04.sh). Put the wrapper at `eSim-2.5/install-eSim.sh` and the version-specific script in `eSim-2.5/install-eSim-scripts/`.

**To try the fix**, restore the clean VM, then run these commands in Bash from the directory containing the ZIP. Answer the installer’s proxy prompt for your connection.

```bash
git clone --branch installers --single-branch --filter=blob:none --sparse \
  https://github.com/dhruvsingh0102/eSim.git esim-installers
git -C esim-installers sparse-checkout set Ubuntu
python3 -m zipfile -e eSim-2.5.zip .
cp esim-installers/Ubuntu/install-eSim.sh eSim-2.5/
cp esim-installers/Ubuntu/install-eSim-scripts/*.sh eSim-2.5/install-eSim-scripts/
cd eSim-2.5
set -o pipefail
bash ./install-eSim.sh --install 2>&1 | tee ../install.log
printf 'Installation exit status: %s\n' "$?"
```

After installation, run `esim` from an X11 session.

## What passed

| Check | Result |
| --- | --- |
| Install after restoring the clean disk | Exit status 0. Installed KiCad 8.0.8 and ngspice 44.2, created `/usr/bin/esim`, and copied the symbol table and desktop shortcut. |
| Python imports | The venv imported PyQt5, Matplotlib, hdlparse, and Watchdog successfully. |
| Main eSim window | `esim` opened the eSim 2.5 main window under Xvfb with Openbox and a test workspace. |
| Basic circuit simulation | A standalone ngspice voltage divider used two 1 kΩ resistors and a 10 V source. It returned `v(out) = 5.000000e+00`, the expected 5 V. |
| Rerun with KiCad 8 already installed | The installer reached launcher setup and exited 0 again. |

I also checked Bash syntax for both scripts and tested how the wrapper routes install and uninstall requests for Ubuntu 22.04.4, 23.04, 24.04, and 25.04. Those checks covered invalid arguments, a missing version script, and a failing version script.

**Test limits.** The guest was ARM64 and used a virtual X11 display. The desktop icon trust metadata was unavailable in that setup. The terminal launcher worked. AMD64, a full uninstall, and NGHDL co-simulation remain unverified.

## Screenshots

1. [GitHub fork on the installers branch](screenshots/01-github-installer-branch.jpg)
2. [Clean Ubuntu VM before installing eSim](screenshots/02-clean-ubuntu-checkpoint.jpg)
3. [The first installer failures](screenshots/03-pyqt5-install-failure.jpg)
4. [The successful run after restoring the clean VM](screenshots/04-clean-installer-rerun.jpg)
5. [eSim 2.5 main window](screenshots/05-esim-main-window.png)
