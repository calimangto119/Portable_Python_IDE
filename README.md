# Portable Python IDE

## Description

This application is a portable Python IDE designed to run directly from a USB drive on Windows. It lets you set up a self-contained Python development environment by creating a dedicated **PPython** folder on your USB drive. You can use either a regular or a portable version of Python. When you launch the IDE, you'll be prompted to select your Python installation folder (the one containing `python.exe`), and the IDE will use that environment to run your scripts.

### Key Features

- **Code Editor:**  
  - Syntax highlighting and line numbering.
  - Search and replace functionality.

- **Integrated Output & Debug Panels:**  
  - Real-time output display.
  - Debugger interface using pdb for step-by-step execution.

- **Package Management:**  
  - Install, upgrade, list, and import Python packages.
  - Export and import requirements files.

- **Auto-Save & Backup:**  
  - Automatically saves your work every few minutes.
  - Options for backing up configurations and scripts.

- **Standalone App Creation:**  
  - Convert your scripts into standalone executables using PyInstaller.

## Setup Instructions

1. **Prepare Your USB Drive:**
   - Create a folder named **PPython** on your USB drive. This folder will serve as the destination for your Python installation.

2. **Install Python:**
   - Install the regular version of Python using the official installer.
   - Optionally, copy the installed Python folder into your **PPython** folder on the USB drive if you want to keep everything portable.

3. **Copy the IDE Script:**
   - Place the IDE script (e.g., `RDB_To_CSV_Database_Viewer.py`) on your USB drive.

4. **Run the IDE:**
   - Launch the script on a Windows machine.
   - When prompted, select your Python installation folder (the one containing `python.exe`). This configures the IDE to use the chosen Python environment.

5. **Start Developing:**
   - Use the integrated code editor to write, run, and debug your Python scripts.
   - Manage packages and create standalone executables—all from your portable environment.

## License

This project is licensed under the **Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International License (CC BY-NC-ND 4.0)**.  
You are free to share this work as long as you give appropriate credit, do not use it for commercial purposes without explicit permission, and do not modify it.

For the full license text, see the [LICENSE](LICENSE) file or visit [CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/).

## Disclaimer

This software is provided "as-is" without any warranty. The author is not liable for any damages or issues arising from its use.
