# Portable Python IDE

**Tags:** `#Python`, `#IDE`, `#Portable`, `#USB`, `#Development`, `#CodeEditor`, `#Debugger`, `#PackageManagement`, `#PyInstaller`, `#Windows`

## Description

This application is a portable Python IDE designed to run directly from a USB drive on Windows. It lets you set up a self-contained Python development environment by creating a dedicated `PPython` folder on your USB drive. You can use either a regular or a portable version of Python. When you launch the IDE, you'll be prompted to select your Python installation folder (the one containing `python.exe`), and the IDE will use that environment to run your scripts.

## Key Features

### Code Editor
- **Syntax Highlighting and Line Numbering:** Makes your code easier to read and debug.
- **Search and Replace:** Quickly locate and update code segments.

### Integrated Output & Debug Panels
- **Real-time Output Display:** See your program's output instantly.
- **Debugger Interface:** Utilize Python’s built-in `pdb` for step-by-step execution.

### Package Management
- **Comprehensive Package Handling:** Install, upgrade, list, and import Python packages.
- **Requirements Files:** Easily export and import dependencies.

### Auto-Save & Backup
- **Auto-Save:** Automatically saves your work every few minutes.
- **Backup Options:** Back up your configurations and scripts to prevent data loss.

### Standalone App Creation
- **Executable Conversion:** Use PyInstaller to convert your scripts into standalone executables.

## Setup Instructions

### Prepare Your USB Drive
- **Create Folder:** Make a folder named `PPython` on your USB drive. This folder will serve as the destination for your Python installation.

### Install Python
- **Regular Installation:** Install the regular version of Python using the official installer.
- **Portable Option:** Optionally, copy the installed Python folder into your `PPython` folder if you want a fully portable setup.
- **Additional Packages:** Ensure `pip` and `pyqt5` are installed before the first launch, as they are required for the IDE's user interface.

### Copy the IDE Script
- **Script Placement:** Place the IDE script (e.g., `RDB_To_CSV_Database_Viewer.py`) on your USB drive.

### Run the IDE
- **Launch the Script:** Start the script on a Windows machine.
- **Select Python Folder:** When prompted, select your Python installation folder (the one containing `python.exe`). This configures the IDE to use the chosen Python environment.

### Start Developing
- **Write and Debug:** Use the integrated code editor to write, run, and debug your Python scripts.
- **Manage Packages and Build Executables:** All from your portable environment.

## License

This project is licensed under the [Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International License (CC BY-NC-ND 4.0)](https://creativecommons.org/licenses/by-nc-nd/4.0/). You are free to share this work as long as you give appropriate credit, do not use it for commercial purposes without explicit permission, and do not modify it.

For the full license text, see the LICENSE file or visit the [CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/) page.

## Disclaimer

This software is provided "as-is" without any warranty. The author is not liable for any damages or issues arising from its use.
