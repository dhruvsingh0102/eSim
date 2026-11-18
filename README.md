eSim Packaging
====

It contains all the documentation for packaging eSim for distribution.

## Winter Internship Task 4

This work looks at what happens when you install eSim 2.5 on Ubuntu 25.04. The main blocker was PyQt5. Once the installer used Ubuntu's PyQt5 package from inside the eSim virtual environment, the installation completed and the main window opened on the ARM64 test VM.

[Read the report, reproduce the installation, and see the screenshots](REPORT.md).


# Packaging eSim for Distribution:

1. eSim is currently packaged and distributed for Ubuntu OS (Linux) and MS Windows OS.

2. Refer the [documentation](Version_Change.md) for the changes to be done when a new release is to be made.

> Note: These changes have to be made `first` before proceeding with the packaging on either platform.

3. Refer the [documentation](Ubuntu/README.md) to package eSim for Ubuntu OS.

4. Refer the [documentation](Windows/README.md) to package eSim for Windows OS.
