# eSim 2.5 installation on Ubuntu 25.04

FOSSEE Winter Internship 2026 · Screening Task 4 · 18 November 2026

## Test environment

| Item | Value |
| --- | --- |
| Application | FOSSEE eSim 2.5 Linux ZIP, verified against SHA-256 `b9860ae26f026930bae589520761f4a57b85d593a2e2cd1ac83d5e5db2f281ed` |
| Installer baseline | [FOSSEE/eSim installers branch at a77d6b9](https://github.com/FOSSEE/eSim/tree/a77d6b904c7bab275d819cf4d8cbb594802fe16d) |
| Guest | Ubuntu 25.04 Plucky, ARM64, 2 vCPUs, 4 GiB RAM, 40 GiB virtual disk |
| Host and hypervisor | Apple M4 Mac, Lima 2.3.0 with Apple Virtualization |
| Reset method | Shut down the guest before cloning the clean disk with APFS copy-on-write. Restore that clone before the fixed run. |

The release ZIP and the installers branch supply different inputs. The ZIP contains the eSim 2.5 application, KiCad library, SKY130 library and NGHDL archive. The branch supplies the installer scripts. [FOSSEE release downloads](https://esim.fossee.in/downloads)

## Issues and fixes

| Step | Observed result | Cause | Change |
| --- | --- | --- | --- |
| Execute the upstream installer | `bash -n Ubuntu/install-eSim.sh` exited 2 at line 70 with `syntax error near unexpected token '}'`. | An unrelated installer body followed the version dispatcher's child-script call and left the surrounding `if` unclosed. | Restored the upstream dispatcher from `f4cede1`, including the 25.04 case and child exit status propagation. |
| Run the released eSim 2.5 ZIP on Ubuntu 25.04 | The published installer reported `Unsupported Ubuntu version: 25.04` and exited 1 in the guest. | Its dispatcher and script directory cover only 22.04, 23.04 and 24.04. | Overlay both the repaired dispatcher and version scripts onto the released package. |
| Install Python GUI dependencies | The branch installer reached `pip3 install PyQt5`. It tried to build PyQt5 on ARM64 and exited 1 with `metadata-generation-failed` and a missing `qmake` error. The installer stopped before generating the launcher. | The installer installed PyQt5 using APT, created an isolated virtual environment that could not see it, and then reinstalled PyQt5 from PyPI. | Use Ubuntu's PyQt5, Matplotlib and Watchdog packages in a Python venv created with `--system-site-packages`. Keep HDL packages installed using that venv's Python. |
| Install eSim alongside existing KiCad 8 | A controlled Bash test reproduced an exit status of 0 before the script reached launcher setup. | `installKicad` called `exit 0` inside the function. | Detect installed KiCad 8 with `dpkg-query` and return from the function so installation continues. Clean installations use the KiCad 8.0.8 packages from Plucky universe. |
| Register eSim symbols with KiCad | The installer source targeted KiCad 6.0 even though Ubuntu 25.04 installs KiCad 8. | A hardcoded config path remained from a previous KiCad version. | Copy the symbol table into KiCad 8's config directory and honor `XDG_CONFIG_HOME`. Preserve that config directory on uninstall. |
| Install bundled NGHDL | Executing its launcher separately in the Ubuntu guest exited 1 with `Unsupported Ubuntu version: 25.04`. | This release's embedded NGHDL archive lacks a Plucky script. | Use the matching NGHDL installer if present and successful. Otherwise emit a warning and install Ubuntu's ngspice so regular SPICE simulations remain available. |
| Create the launcher on a minimal guest | The clean Ubuntu image lacks a user desktop directory. | The install script assumed `$HOME/Desktop` already existed. | Create the directory before copying the desktop launcher. |

## Reproduce

Use a clean Ubuntu 25.04 VM with the `universe` repository enabled. Download the eSim 2.5 Linux ZIP from FOSSEE and verify its checksum. Extract and run the original installer first to reproduce its version rejection. To reproduce the Python failure, copy the repaired dispatcher and the 25.04 child from the baseline commit into a separate fresh copy of the package.

For the fixed run, restore the clean VM and run the following inside Ubuntu. The current directory during installation must be the eSim application root.

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

Answer the proxy prompt for your connection. Launch the program with `esim` in an X11 session.

## Verification and evidence

Bash syntax checks passed for the dispatcher and the Ubuntu 25.04 child script. Isolated dispatch tests passed for install and uninstall routing on 22.04.4, 23.04, 24.04 and 25.04, as well as invalid arguments, missing child scripts and nonzero child exit status propagation.

The fixed installer exited **0** after a clean disk restore. It installed KiCad 8.0.8 and ngspice 44.2, generated `/usr/bin/esim`, and placed the symbol table and desktop launcher in the expected paths. The venv imported PyQt5, Matplotlib, hdlparse and Watchdog successfully. A second guest run with KiCad 8 already installed also reached launcher setup and exited **0**.

Launching `esim` on an Xvfb display with Openbox and a test workspace produced the eSim 2.5 main window. A separate ngspice smoke test drove two equal 1 kΩ resistors with a 10 V source and measured 5 V at their midpoint.

```text
Baseline: error: metadata-generation-failed (PyQt5)
Baseline: Ubuntu 25.04 child exit status: 1
Fixed:    eSim Installed Successfully
Fixed:    Fixed installer exit status: 0
GUI:      eSim 2.5 main window rendered successfully
ngspice:  v(out) = 5.000000e+00
```

- [Public fork on the installers branch](screenshots/01-github-installer-branch.jpg)
- [Clean Ubuntu guest and disk checkpoint](screenshots/02-clean-ubuntu-checkpoint.jpg)
- [Original release rejection and PyQt5 baseline error](screenshots/03-pyqt5-install-failure.jpg)
- [Successful fixed run after restoring the clean disk](screenshots/04-clean-installer-rerun.jpg)
- [eSim 2.5 main window in the Ubuntu VM](screenshots/05-esim-main-window.png)

The checks used ARM64 and a virtual X11 display. The headless guest reported that trusted desktop icon metadata was unavailable. The installed terminal launcher opened successfully. NGHDL co-simulation and an AMD64 install remain unverified. The bundled NGHDL installer failed its version check before reaching LLVM.
