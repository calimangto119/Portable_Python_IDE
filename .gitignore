import ctypes
import os
import sys
import subprocess
import threading
import shutil
import re
import io
import importlib
import platform
import time
import tempfile
import json
import datetime
import ast
import logging
import traceback
from pathlib import Path
from PyQt5.QtWidgets import (
    QApplication, QMainWindow, QWidget, QFileDialog, QMessageBox, QVBoxLayout, 
    QHBoxLayout, QPushButton, QPlainTextEdit, QLabel, QLineEdit, QAction, 
    QProgressBar, QTabWidget, QSplitter, QStyleFactory, QInputDialog, 
    QSizePolicy, QTextEdit, QCheckBox
)
from PyQt5.QtCore import Qt, QRect, QSize, QThread, pyqtSignal, QTimer
from PyQt5.QtGui import QFont, QColor, QSyntaxHighlighter, QTextCharFormat, QTextCursor, QPainter, QTextFormat

# Configure logging
logging.basicConfig(
    filename='error_log.txt',
    filemode='a',
    format='%(asctime)s - %(levelname)s - %(message)s',
    level=logging.ERROR
)

# Mapping of import names to PyPI package names
IMPORT_TO_PYPI = {
    'PIL': 'Pillow',
    'yaml': 'PyYAML',
    # Add more mappings as needed
}

# Helper functions
def save_python_path(path):
    """Save the selected Python installation path to a JSON file in a fixed application directory."""
    config_path = Path("config")  # Adjust to root path as needed
    config_path.mkdir(exist_ok=True)  # Ensure the config directory exists
    path_file = config_path / "python_path.json"
    with open(path_file, "w") as file:
        json.dump({"python_path": str(path)}, file)

def load_python_path():
    """Load the saved Python installation path from the fixed application directory."""
    path_file = Path("config") / "python_path.json"
    if path_file.exists():
        with open(path_file, "r") as file:
            data = json.load(file)
            return Path(data.get("python_path"))
    return None

def is_hidden_or_system(filepath):
    """Checks if a file or directory is hidden or a system file."""
    if platform.system() != 'Windows':
        return False  # Simplistic check for non-Windows systems
    attrs = ctypes.windll.kernel32.GetFileAttributesW(str(filepath))
    if attrs == -1:
        return False
    return bool(attrs & 2 or attrs & 4)
def get_python_executable():
    # Dynamically locate the python executable on the drive
    drive_root = Path(sys.executable).drive  # Detects the current drive letter
    return str(Path(drive_root) / "PPython" / "python.exe")

class InstallPackageThread(QThread):
    progress_signal = pyqtSignal(str)
    error_signal = pyqtSignal(str)
    skip_signal = pyqtSignal(str)

    def __init__(self, package, python_executable):
        super().__init__()
        self.package = package
        self.python_executable = python_executable

    def run(self):
        try:
            self.progress_signal.emit(f"Attempting to install: {self.package}")
            process = subprocess.Popen(
                [self.python_executable, "-m", "pip", "install", self.package],
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE,
                text=True,
                bufsize=1  # Enable line-buffered output for real-time updates
            )

            # Read output line-by-line for real-time updates
            for line in iter(process.stdout.readline, ''):
                if line:
                    self.progress_signal.emit(line.strip())
                    QApplication.processEvents()  # Force UI to update

            process.stdout.close()
            process.wait()

            # Capture any error messages after completion
            stderr_output = process.stderr.read()
            if process.returncode == 0:
                self.progress_signal.emit(f"Successfully installed: {self.package}")
            else:
                self.skip_signal.emit(f"Skipping installation of {self.package}: {stderr_output.strip()}")
            process.stderr.close()

        except Exception as e:
            self.skip_signal.emit(f"Unexpected error installing {self.package}: {str(e)}")


class InstallRequirementsThread(QThread):
    progress_signal = pyqtSignal(str)
    error_signal = pyqtSignal(str)
    skip_signal = pyqtSignal(str)

    def __init__(self, requirements_path, python_executable):
        super().__init__()
        self.requirements_path = requirements_path
        self.python_executable = python_executable

    def run(self):
        try:
            with open(self.requirements_path, 'r') as file:
                packages = file.read().splitlines()

            for package in packages:
                self.progress_signal.emit(f"Attempting to install: {package}")

                process = subprocess.Popen(
                    [self.python_executable, "-m", "pip", "install", package],
                    stdout=subprocess.PIPE,
                    stderr=subprocess.PIPE,
                    text=True,
                    bufsize=1,
                    universal_newlines=True  # Ensures real-time text output
                )

                # Read and display output line-by-line
                while True:
                    line = process.stdout.readline()
                    if not line:
                        break
                    self.progress_signal.emit(line.strip())
                    QApplication.processEvents()  # Update UI for each output line

                # Handle errors after each package installation attempt
                stderr_output = process.stderr.read()
                if process.returncode == 0:
                    self.progress_signal.emit(f"Successfully installed: {package}")
                else:
                    self.skip_signal.emit(f"Failed to install {package}: {stderr_output.strip()}")
                process.stderr.close()

        except Exception as e:
            self.error_signal.emit(f"Error reading {self.requirements_path}: {str(e)}")




class PythonHighlighter(QSyntaxHighlighter):
    def __init__(self, parent=None, theme='Modern'):
        super().__init__(parent)
        self.theme = theme
        self.highlighting_rules = []

        keyword_format = QTextCharFormat()
        keyword_format.setForeground(QColor("blue"))
        keyword_format.setFontWeight(QFont.Bold)

        comment_format = QTextCharFormat()
        comment_format.setForeground(QColor("green"))
        comment_format.setFontItalic(True)

        string_format = QTextCharFormat()
        string_format.setForeground(QColor("orange"))

        keywords = [
            "if", "else", "elif", "for", "while", "break", "continue", "return",
            "try", "except", "finally", "raise", "assert", "with", "as", "pass",
            "async", "await", "yield", "def", "class", "lambda", "global",
            "nonlocal", "import", "from", "and", "or", "not", "in", "is",
            "True", "False", "None"
        ]

        for word in keywords:
            pattern = r'\b' + re.escape(word) + r'\b'
            self.highlighting_rules.append((re.compile(pattern), keyword_format))

        self.highlighting_rules.append((re.compile(r"#.*"), comment_format))
        self.highlighting_rules.append((re.compile(r'\".*?\"'), string_format))
        self.highlighting_rules.append((re.compile(r"\'.*?\'"), string_format))

    def highlightBlock(self, text):
        for pattern, fmt in self.highlighting_rules:
            for match in pattern.finditer(text):
                start, end = match.span()
                self.setFormat(start, end - start, fmt)

class LineNumberArea(QWidget):
    def __init__(self, editor):
        super().__init__(editor)
        self.code_editor = editor

    def sizeHint(self):
        return QSize(self.code_editor.line_number_area_width(), 0)

    def paintEvent(self, event):
        self.code_editor.line_number_area_paint_event(event)

class DebugConsole(QPlainTextEdit):
    def __init__(self, parent=None):
        super().__init__(parent)
        self.setReadOnly(False)  # Allow user input for debugger commands
        self.setFont(QFont("Consolas", 10))
        self.setPlaceholderText("Enter debugger commands here (e.g., 'n' for next, 'c' for continue)")

class CodeEditor(QPlainTextEdit):
    def __init__(self, parent=None, font_size=12, theme='Modern'):
        super().__init__(parent)
        self.setFont(QFont("Consolas", font_size))
        self.highlighter = PythonHighlighter(self.document(), theme)
        self.theme = theme
        self.line_number_area = LineNumberArea(self)

        # Set tab width to 4 spaces
        tab_width = 4 * self.fontMetrics().horizontalAdvance(' ')
        self.setTabStopDistance(tab_width)

        # Setup editor properties and signals
        self.setLineWrapMode(QPlainTextEdit.NoWrap)
        self.setHorizontalScrollBarPolicy(Qt.ScrollBarAsNeeded)
        self.setVerticalScrollBarPolicy(Qt.ScrollBarAsNeeded)
        
        # Connect signals for line numbers and highlighting
        self.blockCountChanged.connect(self.update_line_number_area_width)
        self.updateRequest.connect(self.update_line_number_area)
        self.cursorPositionChanged.connect(self.highlight_current_line)

        # Initial setup for line numbers and highlighting
        self.update_line_number_area_width(0)
        self.highlight_current_line()

        self.current_search_pos = 0  # Initialize search position tracker

    def set_theme(self, theme):
        self.theme = theme
        if theme == "Dark":
            self.setStyleSheet("background-color: #2B2B2B; color: #FFFFFF;")
        elif theme == "Modern":
            self.setStyleSheet("""
                background-color: #F0F0F0;
                color: #333333;
                border: 1px solid #CCCCCC;
                padding: 5px;
            """)
        else:
            self.setStyleSheet("background-color: #FFFFFF; color: #000000;")
        self.highlighter = PythonHighlighter(self.document(), theme)

    def keyPressEvent(self, event):
        if event.key() == Qt.Key_Tab:
            # Insert 4 spaces when Tab key is pressed
            self.insertPlainText(' ' * 4)
        else:
            # Handle other key events normally
            super().keyPressEvent(event)

    def line_number_area_width(self):
        digits = len(str(max(1, self.blockCount())))
        space = 3 + self.fontMetrics().horizontalAdvance('9') * digits
        return space

    def update_line_number_area_width(self, _):
        self.setViewportMargins(self.line_number_area_width(), 0, 0, 0)

    def update_line_number_area(self, rect, dy):
        if dy:
            self.line_number_area.scroll(0, dy)
        else:
            self.line_number_area.update(0, rect.y(), self.line_number_area.width(), rect.height())

        if rect.contains(self.viewport().rect()):
            self.update_line_number_area_width(0)

    def highlight_current_line(self):
        extra_selections = []
        if not self.isReadOnly():
            selection = QTextEdit.ExtraSelection()
            line_color = QColor("#e0e0e0")  # Light grey background
            selection.format.setBackground(line_color)
            selection.format.setProperty(QTextFormat.FullWidthSelection, True)  # Use QTextFormat here
            selection.cursor = self.textCursor()
            selection.cursor.clearSelection()
            extra_selections.append(selection)
        self.setExtraSelections(extra_selections)

    def resizeEvent(self, event):
        super().resizeEvent(event)
        cr = self.contentsRect()
        self.line_number_area.setGeometry(QRect(cr.left(), cr.top(), self.line_number_area_width(), cr.height()))

    def line_number_area_paint_event(self, event):
        painter = QPainter(self.line_number_area)
        if self.theme == "Dark":
            painter.fillRect(event.rect(), QColor("#3C3C3C"))
        else:
            painter.fillRect(event.rect(), QColor("#F0F0F0"))

        block = self.firstVisibleBlock()
        block_number = block.blockNumber()
        top = int(self.blockBoundingGeometry(block).translated(self.contentOffset()).top())
        bottom = top + int(self.blockBoundingRect(block).height())

        while block.isValid() and top <= event.rect().bottom():
            if block.isVisible() and bottom >= event.rect().top():
                number = str(block_number + 1)
                if self.theme == "Dark":
                    painter.setPen(Qt.white)
                else:
                    painter.setPen(Qt.black)
                painter.drawText(0, top, self.line_number_area.width() - 2, self.fontMetrics().height(),
                                 Qt.AlignRight, number)

            block = block.next()
            top = bottom
            bottom = top + int(self.blockBoundingRect(block).height())
            block_number += 1

    def clear_highlights(self):
        """
        Clears any existing highlights in the document by resetting formats.
        """
        try:
            cursor = QTextCursor(self.document())
            format_clear = QTextCharFormat()  # Default formatting with no background
            cursor.setPosition(0)  # Move cursor to the start of the document
            cursor.movePosition(QTextCursor.End, QTextCursor.KeepAnchor)
            cursor.setCharFormat(format_clear)
        except Exception as e:
            print(f"Error clearing highlights: {e}")

    def search_text(self, search_term):
        """
        Searches for the next occurrence of the search term in the editor, highlights it,
        and scrolls to its position. Resets to the top if no more matches are found.
        """
        try:
            # Clear existing highlights
            self.clear_highlights()

            # Document text to be searched
            document_text = self.toPlainText()
            
            # Compile the regex pattern
            regex_pattern = re.compile(re.escape(search_term), re.IGNORECASE)
            
            # Find the next match starting from the current search position
            match = regex_pattern.search(document_text, self.current_search_pos)
            
            # If a match is found, highlight and scroll to it
            if match:
                start_pos, end_pos = match.span()

                # Create cursor for match and set formatting
                cursor = QTextCursor(self.document())
                cursor.setPosition(start_pos)
                cursor.setPosition(end_pos, QTextCursor.KeepAnchor)

                # Highlight formatting
                highlight_format = QTextCharFormat()
                highlight_format.setBackground(QColor("yellow"))
                cursor.setCharFormat(highlight_format)

                # Move to the found match
                self.setTextCursor(cursor)
                self.ensureCursorVisible()

                # Update current_search_pos to continue from the end of this match
                self.current_search_pos = end_pos
                return True  # Indicates a match was found

            # If no match found from current position, wrap around and start from the top
            elif self.current_search_pos > 0:
                # Reset search position to start of document and try again
                self.current_search_pos = 0
                return self.search_text(search_term)  # Recursive call to search from the top

            # If no match found at all, show message and reset position
            else:
                QMessageBox.information(self, "Search", f"'{search_term}' not found.")
                return False  # Indicates no match found

        except Exception as e:
            print(f"Error during search: {e}")

    def replace_text(self, search_term, replace_text):
        """
        Replaces the current selection (if it matches search_term) with replace_text.
        """
        cursor = self.textCursor()
        if cursor.hasSelection() and cursor.selectedText() == search_term:
            cursor.insertText(replace_text)
            self.setTextCursor(cursor)
            return True
        return False

    def replace_all(self, search_term, replace_text):
        """
        Replaces all occurrences of search_term with replace_text, respecting match case and whole word options.
        """
        cursor = self.textCursor()
        cursor.movePosition(QTextCursor.Start)
        self.setTextCursor(cursor)
        
        while self.search_text(search_term):
            self.replace_text(search_term, replace_text)

class OutputPanel(QPlainTextEdit):
    def __init__(self, parent=None, bg_color="white", text_color="black"):
        super().__init__(parent)
        self.setReadOnly(True)
        self.setStyleSheet(f"background-color: {bg_color}; color: {text_color};")
        self.setFont(QFont("Consolas", 10))
        self.setLineWrapMode(QPlainTextEdit.NoWrap)
        self.setHorizontalScrollBarPolicy(Qt.ScrollBarAsNeeded)
        self.setVerticalScrollBarPolicy(Qt.ScrollBarAsNeeded)
        
    def write(self, message):
        self.appendPlainText(message)
        self.verticalScrollBar().setValue(self.verticalScrollBar().maximum())

    def flush(self):
        pass
def redirect_output(self):
    sys.stdout = self.output_panel
    sys.stderr = self.output_panel
    
class InstallThread(QThread):
    package_installed_signal = pyqtSignal(str)
    error_signal = pyqtSignal(str)

    def __init__(self, python_executable, package_name):
        super().__init__()
        self.python_executable = python_executable
        self.package_name = package_name

    def run(self):
        install_command = [self.python_executable, "-m", "pip", "install", self.package_name]
        try:
            process = subprocess.Popen(
                install_command,
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE,
                text=True
            )

            for line in process.stdout:
                self.package_installed_signal.emit(line.strip())
            for line in process.stderr:
                self.error_signal.emit(line.strip())
                
            process.stdout.close()
            process.stderr.close()
            process.wait()
        except Exception as e:
            self.error_signal.emit(f"Error installing {self.package_name}: {str(e)}")


    
class MainWindow(QMainWindow):
    def __init__(self):
        super(MainWindow, self).__init__()
        self.theme = "Modern"
        self.font_size = 12
        self.python_executable = None
        self.pythonw_executable = None
        self.usb_drive = None  # You might want to set this based on your specific logic
        self.process = None
        self.skipped_packages = []
        self.project_scripts_folder = None
        self.install_threads = []  # Initialize install_threads
        self.init_ui_structure()
        self.select_python_folder()  # Select Python installation during initialization
        self.active_thread = None  # Reference to the active thread
        self.active_process = None  # Reference to the active subprocess
        self.command_in_progress = False
        self.setup_auto_save()
        self.init_auto_save_timer()

    def init_ui_structure(self):
        self.setWindowTitle("Portable Python IDE")
        self.resize(1400, 900)
        central_widget = QWidget()
        self.setCentralWidget(central_widget)
        self.main_layout = QVBoxLayout()
        central_widget.setLayout(self.main_layout)

        self.create_menu()
        self.status_bar = self.statusBar()
        self.status_bar.showMessage("Ready")

        # Initialize the progress bar at the bottom of the tabs
        self.progress_bar = QProgressBar()
        self.progress_bar.setMaximum(0)  # Indeterminate state
        self.progress_bar.setVisible(False)
        self.main_layout.addWidget(self.progress_bar)  # Add progress bar at the bottom of tabs

        self.tabs = QTabWidget()
        self.tabs.setSizePolicy(QSizePolicy.Expanding, QSizePolicy.Expanding)
        self.main_layout.addWidget(self.tabs)

        self.code_editor_tab = QWidget()
        self.tabs.addTab(self.code_editor_tab, "Code Editor")
        self.init_code_editor_tab()

        self.package_tab = QWidget()
        self.tabs.addTab(self.package_tab, "Application Maintenance")
        self.init_package_tab()

        self.apply_theme()

    def init_auto_save_timer(self):
        self.auto_save_timer = QTimer(self)
        self.auto_save_timer.timeout.connect(self.auto_save_code)
        self.auto_save_interval = 300000  # 5 minutes in milliseconds
        self.auto_save_timer.start(self.auto_save_interval)

    def setup_auto_save(self):
        # Set up the auto-save timer to save the code every 5 minutes (300,000 ms)
        self.auto_save_timer = QTimer(self)
        self.auto_save_timer.timeout.connect(self.auto_save_code)
        self.auto_save_timer.start(300000)  # 5 minutes in milliseconds

    def auto_save_code(self):
        code = self.code_editor.toPlainText().strip()
        if not code:
            return  # Do not save if there's no code

        if hasattr(self, 'current_file_path') and self.current_file_path:
            # Save directly to the open file
            try:
                with open(self.current_file_path, 'w') as file:
                    file.write(code)
                self.output_panel.appendPlainText(f"Auto-saved to {self.current_file_path}")
            except Exception as e:
                self.output_panel.appendPlainText(f"Failed to auto-save to {self.current_file_path}: {e}")
        else:
            # Save to a unique auto-save file in the AutoSave folder
            auto_save_folder = os.path.join(self.usb_drive, "PPython", "AutoSave") if self.usb_drive else "AutoSave"
            os.makedirs(auto_save_folder, exist_ok=True)
            timestamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
            auto_save_path = os.path.join(auto_save_folder, f"auto_save_script_{timestamp}.pyw")
            
            try:
                with open(auto_save_path, 'w') as file:
                    file.write(code)
                self.output_panel.appendPlainText(f"Auto-saved to backup file {auto_save_path}")
            except Exception as e:
                self.output_panel.appendPlainText(f"Failed to auto-save backup: {e}")

    def finalize_ui(self):
        if self.usb_drive:
            self.output_panel.appendPlainText("USB drive selected successfully.")
        else:
            self.output_panel.appendPlainText("USB drive not selected.")

    def create_menu(self):
        menu_bar = self.menuBar()
        file_menu = menu_bar.addMenu("File")
        
        save_action = QAction("Save Script", self)
        save_action.setShortcut("Ctrl+S")
        save_action.triggered.connect(self.save_code)
        file_menu.addAction(save_action)

        load_action = QAction("Load Script", self)
        load_action.setShortcut("Ctrl+O")
        load_action.triggered.connect(self.load_code)
        file_menu.addAction(load_action)

        file_menu.addSeparator()
        
        exit_action = QAction("Exit", self)
        exit_action.triggered.connect(self.close)
        file_menu.addAction(exit_action)

        view_menu = menu_bar.addMenu("View")
        dark_mode_action = QAction("Toggle Dark Mode", self, checkable=True)
        dark_mode_action.setChecked(self.theme == "Dark")
        dark_mode_action.triggered.connect(self.toggle_dark_mode)
        view_menu.addAction(dark_mode_action)

        help_menu = menu_bar.addMenu("Help")
        documentation_action = QAction("Documentation", self)
        documentation_action.triggered.connect(self.show_documentation)
        help_menu.addAction(documentation_action)

        about_action = QAction("About", self)
        about_action.triggered.connect(self.show_about)
        help_menu.addAction(about_action)

    def init_code_editor_tab(self):
        layout = QVBoxLayout()
        self.code_editor_tab.setLayout(layout)

        # Main Splitter for the code editor and output/debug panels
        main_splitter = QSplitter(Qt.Horizontal)
        layout.addWidget(main_splitter)

        # Left Side - Code Editor and Buttons
        left_widget = QWidget()
        left_layout = QVBoxLayout()
        left_widget.setLayout(left_layout)
        self.code_editor = CodeEditor(font_size=self.font_size, theme=self.theme)
        left_layout.addWidget(self.code_editor)

        # Search/Replace Layout
        search_replace_layout = QHBoxLayout()
        self.search_input = QLineEdit()
        self.search_input.setPlaceholderText("Search")
        self.match_case_checkbox = QCheckBox("Match Case")
        self.whole_word_checkbox = QCheckBox("Whole Word Only")
        self.replace_input = QLineEdit()
        self.replace_input.setPlaceholderText("Replace")
        search_button = QPushButton("Search")
        search_button.clicked.connect(self.perform_search)
        replace_button = QPushButton("Replace")
        replace_button.clicked.connect(self.perform_replace)
        replace_all_button = QPushButton("Replace All")
        replace_all_button.clicked.connect(self.perform_replace_all)

        search_replace_layout.addWidget(self.search_input)
        search_replace_layout.addWidget(self.match_case_checkbox)
        search_replace_layout.addWidget(self.whole_word_checkbox)
        search_replace_layout.addWidget(self.replace_input)
        search_replace_layout.addWidget(search_button)
        search_replace_layout.addWidget(replace_button)
        search_replace_layout.addWidget(replace_all_button)

        # Add the layout to the main editor layout
        left_layout.addLayout(search_replace_layout)

        # Button Layout
        button_layout = QHBoxLayout()
        run_code_button = QPushButton("Run Code")
        run_code_button.clicked.connect(self.run_code)
        save_code_button = QPushButton("Save Script")
        save_code_button.clicked.connect(self.save_code)
        load_code_button = QPushButton("Load Script")
        load_code_button.clicked.connect(self.load_code)
        stop_code_button = QPushButton("Stop Script")
        stop_code_button.clicked.connect(self.stop_script)
        clear_code_button = QPushButton("Clear Code")
        clear_code_button.clicked.connect(self.clear_code)
        debug_code_button = QPushButton("Debug Code")
        debug_code_button.clicked.connect(self.run_debugger)
        button_layout.addWidget(run_code_button)
        button_layout.addWidget(save_code_button)
        button_layout.addWidget(load_code_button)
        button_layout.addWidget(stop_code_button)
        button_layout.addWidget(clear_code_button)
        button_layout.addWidget(debug_code_button)
        left_layout.addLayout(button_layout)

        main_splitter.addWidget(left_widget)

        # Right Side - Output and Debugging Tools
        right_splitter = QSplitter(Qt.Vertical)  # For stacking output and debugging console
        main_splitter.addWidget(right_splitter)

        # Output Panel
        self.output_panel = OutputPanel(bg_color="#F0F0F0", text_color="#333333")
        right_splitter.addWidget(self.output_panel)

        # Debugging Console with Legend
        debug_area = QWidget()
        debug_layout = QHBoxLayout(debug_area)
        
        # Left side: Command Legend
        legend = QLabel("Debugger Commands:\n"
                        "n - Next\n"
                        "s - Step In\n"
                        "c - Continue\n"
                        "q - Quit\n"
                        "p <var> - Print variable\n"
                        "l - List code\n")
        legend.setFont(QFont("Consolas", 10))
        legend.setAlignment(Qt.AlignTop)
        legend.setFixedWidth(150)  # Set a fixed width for the legend
        
        # Right side: Debug Console
        self.debug_console = DebugConsole()
        debug_layout.addWidget(legend)
        debug_layout.addWidget(self.debug_console)
        
        right_splitter.addWidget(debug_area)

        # Set default stretch factors to have debug console at 20% height
        right_splitter.setStretchFactor(0, 4)  # Output panel takes up 80%
        right_splitter.setStretchFactor(1, 1)  # Debug console takes up 20%

        # Adjust initial size of the main splitter to make debug area 20% by default
        main_splitter.setStretchFactor(0, 3)  # Code editor takes up 60%
        main_splitter.setStretchFactor(1, 2)  # Output/debug takes up 40%

    def init_package_tab(self):
        layout = QVBoxLayout()
        self.package_tab.setLayout(layout)

        # Package Input Layout
        package_layout = QHBoxLayout()
        package_label = QLabel("Enter package name:")
        self.package_entry = QLineEdit()
        package_layout.addWidget(package_label)
        package_layout.addWidget(self.package_entry)
        layout.addLayout(package_layout)

        # Button Layout
        button_layout = QHBoxLayout()
        install_button = QPushButton("Install Package")
        install_button.clicked.connect(lambda: self.install_package(self.package_entry.text()))
        list_button = QPushButton("List Installed Packages")
        list_button.clicked.connect(self.list_installed_packages)
        upgrade_button = QPushButton("Upgrade Outdated Packages")
        upgrade_button.clicked.connect(self.upgrade_packages)
        update_pip_button = QPushButton("Update pip")
        update_pip_button.clicked.connect(self.update_pip)
        button_layout.addWidget(install_button)
        button_layout.addWidget(list_button)
        button_layout.addWidget(upgrade_button)
        button_layout.addWidget(update_pip_button)
        layout.addLayout(button_layout)

        # Additional Buttons Layout for Requirements and Standalone App
        additional_buttons_layout = QHBoxLayout()
        export_requirements_button = QPushButton("Export Requirements")
        export_requirements_button.clicked.connect(self.export_requirements_txt)
        import_requirements_button = QPushButton("Import Requirements")
        import_requirements_button.clicked.connect(self.import_requirements)  # Button for import_requirements function
        standalone_app_button = QPushButton("Create Standalone App")
        standalone_app_button.clicked.connect(self.create_standalone_app)
        additional_buttons_layout.addWidget(export_requirements_button)
        additional_buttons_layout.addWidget(import_requirements_button)  # Add import requirements button here
        additional_buttons_layout.addWidget(standalone_app_button)
        layout.addLayout(additional_buttons_layout)

        # Backup Button Layout
        backup_layout = QHBoxLayout()
        backup_button = QPushButton("Backup Configuration")
        backup_button.clicked.connect(self.backup_configuration)
        backup_layout.addWidget(backup_button)
        layout.addLayout(backup_layout)

        # Package Output Panel
        self.package_output_panel = OutputPanel(
            bg_color="#F0F0F0",
            text_color="#333333"
        )
        layout.addWidget(self.package_output_panel)

    def log_error(self, error_message, panel='package'):
        """
        Logs errors to the error log file on the USB drive and appends a message to the specified panel.

        :param error_message: The error message to log.
        :param panel: The target panel to display the error message ('package' or 'editor').
        """
        log_path = os.path.join(self.usb_drive, "PPython", "error_log.txt") if self.usb_drive else "error_log.txt"
        timestamp = time.strftime("%Y-%m-%d %H:%M:%S")
        full_message = f"[{timestamp}] {error_message}\n"

        try:
            with open(log_path, "a") as log_file:
                log_file.write(full_message)

            if panel == 'package':
                self.package_output_panel.appendPlainText("An error occurred. Check the error log for details.")
            elif panel == 'editor':
                self.output_panel.appendPlainText("An error occurred. Check the error log for details.")
            else:
                # Default to package panel if an unknown panel is specified
                self.package_output_panel.appendPlainText("An error occurred. Check the error log for details.")
        except Exception as log_error:
            # Append the logging failure to the specified panel or default to package panel
            if panel == 'editor':
                self.output_panel.appendPlainText(f"Failed to write to error log: {log_error}")
            else:
                self.package_output_panel.appendPlainText(f"Failed to write to error log: {log_error}")

    def backup_configuration(self):
        """Backs up all files from the selected USB drive except the 'PPython' folder and exports requirements.txt to the backup location."""
        try:
            # Let user select the USB drive
            usb_drive = QFileDialog.getExistingDirectory(self, "Select USB Drive Folder")
            if not usb_drive:
                self.package_output_panel.appendPlainText("USB drive not selected. Backup cancelled.")
                return
            self.usb_drive = usb_drive  # Set the selected USB drive

            # Let user select the backup folder
            backup_folder = QFileDialog.getExistingDirectory(self, "Select Backup Folder")
            if not backup_folder:
                self.package_output_panel.appendPlainText("Backup location not selected. Backup cancelled.")
                return

            self.package_output_panel.clear()
            self.package_output_panel.appendPlainText("Starting backup process...")
            self.start_progress()
            QApplication.processEvents()

            for item in os.listdir(self.usb_drive):
                if item.lower() == "ppython":
                    self.package_output_panel.appendPlainText(f"Skipped 'PPython' folder: {item}")
                    QApplication.processEvents()
                    continue

                source = os.path.join(self.usb_drive, item)
                destination = os.path.join(backup_folder, item)

                if is_hidden_or_system(source):
                    self.package_output_panel.appendPlainText(f"Skipped hidden or system file/folder: {item}")
                    QApplication.processEvents()
                    continue

                try:
                    if os.path.isdir(source):
                        shutil.copytree(source, destination, dirs_exist_ok=True)
                    else:
                        shutil.copy2(source, destination)
                    self.package_output_panel.appendPlainText(f"Backed up {item} successfully.")
                except Exception as copy_error:
                    self.package_output_panel.appendPlainText(f"Failed to back up {item}: {copy_error}")
                    self.log_error(f"Failed to back up {item}: {copy_error}", panel='package')
                
                self.package_output_panel.verticalScrollBar().setValue(
                    self.package_output_panel.verticalScrollBar().maximum()
                )
                QApplication.processEvents()

            # Export requirements.txt
            self.package_output_panel.appendPlainText("Exporting requirements.txt...")
            try:
                list_command = [self.python_executable, "-m", "pip", "freeze"]
                result = subprocess.run(list_command, capture_output=True, text=True, check=True)
                requirements_path = os.path.join(backup_folder, "requirements.txt")
                with open(requirements_path, 'w') as req_file:
                    req_file.write(result.stdout)
                self.package_output_panel.appendPlainText(f"requirements.txt exported to {requirements_path}")
            except subprocess.CalledProcessError as e:
                self.package_output_panel.appendPlainText("Failed to export requirements.txt.")
                self.package_output_panel.appendPlainText(e.stderr)
                self.log_error(f"Failed to export requirements.txt: {e.stderr}", panel='package')

            self.package_output_panel.appendPlainText("Backup completed successfully.")

        except Exception as e:
            error_message = f"Backup failed: {traceback.format_exc()}"
            self.package_output_panel.appendPlainText(error_message)
            self.log_error(error_message, panel='package')  # Specify the package panel
        finally:
            self.stop_progress()

    def toggle_dark_mode(self):
        if self.theme == "Dark":
            self.theme = "Modern"
        else:
            self.theme = "Dark"
        self.apply_theme()

    def apply_theme(self):
        if self.theme == "Modern":
            self.setStyleSheet("""
                QMainWindow { background-color: #F5F5F5; color: #333333; }
                QTabWidget::pane { border: 1px solid #CCCCCC; background-color: #FFFFFF; }
                QTabBar::tab { background: #E0E0E0; color: #333333; padding: 10px; margin: 2px; }
                QTabBar::tab:selected { background: #FFFFFF; border-bottom: 2px solid #0078D7; }
                QPushButton { background-color: #0078D7; color: #FFFFFF; border: none; padding: 5px 10px; border-radius: 4px; }
                QPushButton:hover { background-color: #005A9E; }
                QLineEdit, QPlainTextEdit { background-color: #FFFFFF; color: #333333; border: 1px solid #CCCCCC; border-radius: 4px; }
            """)
            self.code_editor.set_theme("Modern")
            self.output_panel.setStyleSheet("""
                background-color: #FFFFFF;
                color: #333333;
                border: 1px solid #CCCCCC;
                padding: 5px;
            """)
            self.package_output_panel.setStyleSheet("""
                background-color: #FFFFFF;
                color: #333333;
                border: 1px solid #CCCCCC;
                padding: 5px;
            """)
        elif self.theme == "Dark":
            self.setStyleSheet("""
                QMainWindow { background-color: #2B2B2B; color: #FFFFFF; }
                QTabWidget::pane { border: 1px solid #444444; background-color: #3C3C3C; }
                QTabBar::tab { background: #444444; color: #FFFFFF; padding: 10px; margin: 2px; }
                QTabBar::tab:selected { background: #3C3C3C; border-bottom: 2px solid #6A6A6A; }
                QPushButton { background-color: #555555; color: #FFFFFF; border: none; padding: 5px 10px; border-radius: 4px; }
                QPushButton:hover { background-color: #666666; }
                QLineEdit, QPlainTextEdit { background-color: #3C3C3C; color: #FFFFFF; border: 1px solid #444444; border-radius: 4px; }
            """)
            self.code_editor.set_theme("Dark")
            self.output_panel.setStyleSheet("""
                background-color: #333333;
                color: #FFFFFF;
                border: 1px solid #444444;
                padding: 5px;
            """)
            self.package_output_panel.setStyleSheet("""
                background-color: #333333;
                color: #FFFFFF;
                border: 1px solid #444444;
                padding: 5px;
            """)

    def save_code(self):
        file_path, _ = QFileDialog.getSaveFileName(self, "Save Script", self.project_scripts_folder,
                                                   "Python Files (*.pyw)")
        if file_path:
            if not file_path.endswith('.pyw'):
                file_path += '.pyw'
            try:
                with open(file_path, 'w') as file:
                    code = self.code_editor.toPlainText()
                    file.write(code)
                self.output_panel.appendPlainText(f"Script saved to {file_path}")

                # Prompt the user to decide whether to create a .vbs file
                reply = QMessageBox.question(
                    self,
                    "Create .vbs File",
                    "Do you want to create a corresponding .vbs file for this script?",
                    QMessageBox.Yes | QMessageBox.No,
                    QMessageBox.No
                )

                if reply == QMessageBox.Yes:
                    script_name = os.path.basename(file_path)
                    vbs_content = (
                        'Set objShell = CreateObject("WScript.Shell")\n'
                        'drive = Left(WScript.ScriptFullName, 2)\n'
                        f'objShell.Run """" & drive & "\\PPython\\pythonw.exe"" """ & drive & "\\Saved Scripts\\{script_name}"""'
                    )
                    vbs_filename = f"Open Project - {os.path.splitext(script_name)[0]}.vbs"
                    vbs_path = os.path.join(os.path.dirname(file_path), vbs_filename)
                    with open(vbs_path, 'w') as vbs_file:
                        vbs_file.write(vbs_content)
                    self.output_panel.appendPlainText(f".vbs file created at {vbs_path}")
                else:
                    self.output_panel.appendPlainText("User opted not to create a .vbs file.")
            except Exception as e:
                QMessageBox.critical(self, "Error", f"Failed to save script: {e}")
                self.output_panel.appendPlainText(f"Failed to save script: {e}")

    def load_code(self):
        file_path, _ = QFileDialog.getOpenFileName(self, "Load Script", self.project_scripts_folder, "Python Files (*.pyw *.py)")
        if file_path:
            try:
                with open(file_path, 'r') as file:
                    code = file.read()
                    self.code_editor.setPlainText(code)
                self.output_panel.appendPlainText(f"Loaded script from {file_path}")

                # Check and install missing packages before running
                threading.Thread(target=lambda: self.check_and_install_missing_packages(code)).start()

            except Exception as e:
                QMessageBox.critical(self, "Error", f"Failed to load script: {e}")
                self.output_panel.appendPlainText(f"Failed to load script: {e}")

    def clear_code(self):
        self.code_editor.clear()
        self.output_panel.clear()
        self.package_output_panel.clear()
        self.output_panel.appendPlainText("Code editor and output panels cleared.")
        self.package_output_panel.appendPlainText("Package output panel cleared.")

    def run_code(self):
        code = self.code_editor.toPlainText()
        if code.strip():
            def task():
                try:
                    self.start_progress()
                    self.output_panel.clear()  # Clear the Code Editor output panel
                    self.output_panel.appendPlainText(f"Running the following code:\n\n{code}\n\n")

                    # Check and install missing packages
                    self.check_and_install_missing_packages(code)

                    # Wait for all install threads to finish
                    for thread in self.install_threads:
                        thread.wait()
                    self.install_threads.clear()

                    self.output_panel.appendPlainText("\nAll missing packages installed. Executing the code...\n")

                    # Execute the code after ensuring all packages are installed
                    self.execute_code(code)

                except Exception as e:
                    error_message = f"Run Code Error: {str(e)}\n{traceback.format_exc()}"
                    self.output_panel.appendPlainText(error_message)
                    self.log_error(error_message, panel='editor')  # Direct to editor's output panel
                finally:
                    self.stop_progress()

            threading.Thread(target=task).start()
        else:
            self.output_panel.appendPlainText("No code to execute.")

    def execute_code(self, code):
        """Executes the provided code in a temporary file with improved error reporting."""
        try:
            # Create a temporary file to save the code for execution
            with tempfile.NamedTemporaryFile(mode='w', suffix='.pyw', delete=False) as tmp_file:
                tmp_file.write(code)
                temp_script_path = tmp_file.name

            self.output_panel.appendPlainText(f"Temporary script created at {temp_script_path}\n")
            self.output_panel.appendPlainText("Executing code...\n")

            # Start a subprocess to run the script and capture both stdout and stderr
            process = subprocess.Popen(
                [self.python_executable, temp_script_path],
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE,
                text=True
            )

            stdout, stderr = process.communicate()

            # Display the standard output (if any)
            if stdout:
                self.output_panel.appendPlainText(stdout)

            # Parse the error output (stderr) for better insights
            if stderr:
                self.output_panel.appendHtml(f"<span style='color: red;'>Error:</span><br>")
                error_lines = stderr.splitlines()

                # Highlight and display each line of the error with suggestions
                for line in error_lines:
                    if "NameError" in line:
                        self.output_panel.appendHtml(f"<span style='color: red;'>{line}</span><br>")
                        if "not defined" in line:
                            suggestion = self.provide_error_suggestion(line)
                            if suggestion:
                                self.output_panel.appendHtml(f"<span style='color: orange;'>Suggestion: {suggestion}</span><br>")
                    else:
                        self.output_panel.appendHtml(f"{line}<br>")

                # Display the full traceback if available
                self.output_panel.appendHtml("\nFull traceback:\n")
                self.output_panel.appendHtml(f"<pre style='color: red;'>{traceback.format_exc()}</pre>")

            # Check the return code to determine if execution was successful
            if process.returncode != 0:
                self.output_panel.appendHtml("<span style='color: red;'>Script execution failed with errors.</span>")
            else:
                self.output_panel.appendPlainText("Script executed successfully.")

        except Exception as e:
            # Capture and display detailed traceback if code execution fails
            error_message = f"Error executing code: {str(e)}\n{traceback.format_exc()}"
            self.output_panel.appendHtml(f"<span style='color: red;'>{error_message}</span>")
            self.log_error(error_message, panel='editor')

        finally:
            # Clean up the temporary script file
            if os.path.exists(temp_script_path):
                try:
                    os.remove(temp_script_path)
                except Exception as cleanup_error:
                    self.output_panel.appendPlainText(f"Failed to remove temporary file: {cleanup_error}")

    def provide_error_suggestion(self, error_line):
        """Provides suggestions based on the error line."""
        suggestions = {
            "FirewallManagementTab": "Did you mean 'ServiceManagementTab'? Check if the class is defined or imported correctly.",
            "SystemAdminToolkit": "Ensure 'SystemAdminToolkit' class is defined or imported in your script.",
        }
        
        for key, suggestion in suggestions.items():
            if key in error_line:
                return suggestion
        return None




    def read_process_output(self, pipe, is_error=False):
        """Reads output from a process pipe and updates the output panel in real time, with optional error styling."""
        for line in io.TextIOWrapper(pipe, encoding="utf-8"):
            self.update_output_panel(line.strip(), is_error=is_error)

    def update_output_panel(self, line, is_error=False):
        """Adds a new line to the output panel with optional color for errors, auto-scrolls to the end."""
        if is_error:
            # Format errors in red for easier identification
            self.output_panel.appendHtml(f"<span style='color: red;'>{line.strip()}</span>")
        else:
            self.output_panel.appendPlainText(line.strip())
        self.output_panel.moveCursor(QTextCursor.End)
        self.output_panel.ensureCursorVisible()

    def stop_script(self):
        # Attempt to terminate the active subprocess, if any
        if self.active_process and self.active_process.poll() is None:
            try:
                self.active_process.terminate()
                self.output_panel.appendPlainText("\nScript terminated.")
            except Exception as e:
                self.output_panel.appendPlainText(f"\nFailed to terminate the process: {e}")
            finally:
                self.active_process = None  # Clear the active process

        # Set active_thread to None to release it
        if self.active_thread:
            self.active_thread = None
            self.output_panel.appendPlainText("Script stopping initiated.")

    def get_pypi_package_name(self, import_name):
        """
        Returns the corresponding PyPI package name for a given import name.
        """
        return IMPORT_TO_PYPI.get(import_name, import_name)

    def extract_imported_packages(self, code):
        """
        Parses the code to find all top-level imported packages using AST.
        """
        packages = set()
        try:
            tree = ast.parse(code)
            for node in ast.walk(tree):
                if isinstance(node, ast.Import):
                    for alias in node.names:
                        package = alias.name.split('.')[0]
                        packages.add(package)
                elif isinstance(node, ast.ImportFrom):
                    if node.module:
                        package = node.module.split('.')[0]
                        packages.add(package)
        except SyntaxError as e:
            # **Changed**: Directing errors from Code Editor to output_panel
            self.output_panel.appendPlainText(f"Syntax error while parsing imports: {e}")
            logging.error(f"Syntax error while parsing imports: {e}")
        return list(packages)

    def is_package_installed(self, package_name):
        """
        Checks if a package is installed by attempting to import it.
        """
        try:
            importlib.import_module(package_name)
            return True
        except ImportError:
            return False

    def check_and_install_missing_packages(self, code):
        """Identifies and installs missing packages, updating the output panel."""
        imported_packages = self.extract_imported_packages(code)
        missing_packages = [pkg for pkg in imported_packages if not self.is_package_installed(pkg)]

        if missing_packages:
            self.output_panel.appendPlainText(f"Detected missing packages: {missing_packages}")
            self.package_output_panel.appendPlainText(f"Installing missing packages: {missing_packages}\n")

            # Start threads for each missing package installation
            for package in missing_packages:
                pypi_package = self.get_pypi_package_name(package)
                install_thread = InstallPackageThread(pypi_package, self.python_executable)
                install_thread.progress_signal.connect(self.handle_install_output)
                install_thread.error_signal.connect(self.handle_install_error)
                self.install_threads.append(install_thread)  # Store the thread reference
                install_thread.start()
        else:
            self.output_panel.appendPlainText("All required packages are already installed.")

    def install_package(self, package_name, async_install=True, target_panel="main"):
        if package_name.strip() and self.python_executable:
            output_panel = self.output_panel if target_panel == "main" else self.package_output_panel

            if async_install:
                # Use a thread for asynchronous installation
                self.install_thread = InstallThread(self.python_executable, package_name)
                self.install_thread.package_installed_signal.connect(lambda line: output_panel.appendPlainText(line))
                self.install_thread.error_signal.connect(lambda error: output_panel.appendPlainText(f"Error: {error}"))
                
                # Start the thread
                output_panel.appendPlainText(f"\nInstalling {package_name} asynchronously...\n")
                self.install_thread.start()
            else:
                # Synchronous installation
                try:
                    output_panel.appendPlainText(f"\nInstalling {package_name} synchronously...\n")
                    install_command = [self.python_executable, "-m", "pip", "install", package_name]
                    result = subprocess.run(install_command, capture_output=True, text=True)
                    output_panel.appendPlainText(result.stdout)
                    if result.stderr:
                        output_panel.appendPlainText(f"Error: {result.stderr}")
                    if result.returncode == 0:
                        output_panel.appendPlainText(f"{package_name} installed successfully.\n")
                    else:
                        output_panel.appendPlainText(f"Failed to install {package_name}.\n")
                except Exception as e:
                    output_panel.appendPlainText(f"Error installing {package_name}: {str(e)}")
        else:
            output_panel.appendPlainText("Please enter a valid package name or set the Python executable.\n")

    def install_requirements(self, requirements_path, existing_packages):
        self.requirements_thread = InstallRequirementsThread(requirements_path, existing_packages)
        self.requirements_thread.progress_signal.connect(self.handle_progress_output)
        self.requirements_thread.error_signal.connect(self.handle_install_error)
        self.requirements_thread.skip_signal.connect(self.handle_install_skip)
        self.requirements_thread.start()

    def handle_progress_output(self, message):
        self.package_output_panel.appendPlainText(message)

    def handle_install_skip(self, message):
        self.package_output_panel.appendPlainText(message)

    def handle_install_output(self, message):
        """
        Handles output messages from installation threads and updates the output panel.
        """
        self.package_output_panel.appendPlainText(message)
        QApplication.processEvents()  # Ensure real-time update in output panel

    def handle_install_error(self, error_message):
        """Logs and displays error messages in the package maintenance output panel."""
        self.package_output_panel.appendPlainText(error_message)  # Display error in maintenance tab
        self.log_error(error_message, panel='package')  # Log to maintenance tab
        logging.error(error_message)
        QApplication.processEvents()  # Update the UI with the new error message

    def list_installed_packages(self):
        """
        Lists installed packages using 'pip list' command.
        """
        def task():
            self.start_progress()
            try:
                self.package_output_panel.clear()
                self.package_output_panel.appendPlainText("Retrieving packages with 'pip list'...\n")
                list_command = [sys.executable, "-m", "pip", "list"]
                result = subprocess.run(list_command, capture_output=True, text=True)
                self.package_output_panel.appendPlainText(result.stdout)
                if result.stderr:
                    self.package_output_panel.appendPlainText(result.stderr)

            except Exception as e:
                self.package_output_panel.appendPlainText(f"Failed to list packages: {e}")
            finally:
                self.stop_progress()

        threading.Thread(target=task).start()

    def upgrade_packages(self):
        """
        Retrieves a list of outdated packages and upgrades each one individually if necessary.
        """
        def task():
            self.start_progress()
            try:
                self.package_output_panel.clear()
                self.package_output_panel.appendPlainText("Retrieving list of outdated packages...\n")

                # Step 1: Retrieve outdated packages with 'pip list --outdated'
                list_outdated_command = [sys.executable, "-m", "pip", "list", "--outdated", "--format=json"]
                result = subprocess.run(list_outdated_command, capture_output=True, text=True)
                
                if result.returncode != 0:
                    self.package_output_panel.appendPlainText("Failed to retrieve list of outdated packages.\n")
                    return

                # Parse the JSON output to get the list of outdated packages
                try:
                    outdated_packages = json.loads(result.stdout)
                except json.JSONDecodeError:
                    self.package_output_panel.appendPlainText("Error decoding output from 'pip list --outdated'.\n")
                    return

                if not outdated_packages:
                    self.package_output_panel.appendPlainText("No outdated packages found.\n")
                    return

                # Step 2: Upgrade each outdated package
                failed_packages = []
                for package_info in outdated_packages:
                    package_name = package_info['name']
                    self.package_output_panel.appendPlainText(f"Upgrading {package_name}...\n")

                    # Upgrade command for the individual package
                    upgrade_command = [sys.executable, "-m", "pip", "install", "--upgrade", package_name]
                    upgrade_result = subprocess.run(upgrade_command, capture_output=True, text=True)

                    if upgrade_result.returncode == 0:
                        # Confirm the upgraded version
                        version_command = [sys.executable, "-m", "pip", "show", package_name]
                        version_result = subprocess.run(version_command, capture_output=True, text=True)
                        new_version = "unknown"
                        for line in version_result.stdout.splitlines():
                            if line.startswith("Version:"):
                                new_version = line.split("Version: ")[-1]
                        self.package_output_panel.appendPlainText(f"{package_name} upgraded to version: {new_version}\n")
                    else:
                        failed_packages.append(package_name)
                        self.package_output_panel.appendPlainText(f"Failed to upgrade {package_name}: {upgrade_result.stderr}\n")

                # Step 3: Summarize the results
                if failed_packages:
                    self.package_output_panel.appendPlainText(f"\nFailed to upgrade the following packages: {', '.join(failed_packages)}\n")
                else:
                    self.package_output_panel.appendPlainText("All packages upgraded successfully.\n")

            except Exception as e:
                self.package_output_panel.appendPlainText(f"Error during upgrade: {e}\n")
            finally:
                self.stop_progress()

        threading.Thread(target=task).start()

    def update_pip(self):
        def task():
            self.start_progress()
            try:
                self.package_output_panel.appendPlainText("\nUpdating pip...\n")
                update_command = [sys.executable, "-m", "pip", "install", "--upgrade", "pip"]
                result = subprocess.run(update_command, capture_output=True, text=True)
                self.package_output_panel.appendPlainText(result.stdout)
                if result.stderr:
                    self.package_output_panel.appendPlainText(result.stderr)
                if result.returncode == 0:
                    self.package_output_panel.appendPlainText("pip updated successfully.")
                else:
                    self.package_output_panel.appendPlainText("Failed to update pip.")
            except subprocess.CalledProcessError as e:
                self.package_output_panel.appendPlainText(f"Failed to update pip: {e}")
            except Exception as e:
                self.package_output_panel.appendPlainText(f"Failed to update pip: {e}")
            finally:
                self.stop_progress()

        threading.Thread(target=task).start()

    def create_standalone_app(self):
        app_name, ok = QInputDialog.getText(self, "App Name", "Enter the name of the app:")
        if not ok or not app_name.strip():
            self.output_panel.appendPlainText("Please enter a valid app name.")
            return
        
        # Prompt user to select output folder
        output_folder = QFileDialog.getExistingDirectory(self, "Select Output Folder for Standalone App")
        if not output_folder:
            self.output_panel.appendPlainText("No output folder selected. Operation cancelled.")
            return
        
        app_folder = os.path.join(output_folder, app_name)
        os.makedirs(app_folder, exist_ok=True)
        self.output_panel.appendPlainText(f"Created app folder at: {app_folder}")
        script_path = os.path.join(app_folder, f"{app_name}.pyw")
        
        try:
            with open(script_path, 'w') as file:
                code = self.code_editor.toPlainText()
                file.write(code)
            self.output_panel.appendPlainText(f"Script file created at {script_path}")
        except Exception as e:
            QMessageBox.critical(self, "Error", f"Failed to create script file: {e}")
            self.output_panel.appendPlainText(f"Failed to create script file: {e}")
            return

        def task():
            self.start_progress()
            try:
                self.output_panel.appendPlainText("\nChecking for PyInstaller...")

                # Install PyInstaller if not already installed
                install_command = [sys.executable, "-m", "pip", "install", "pyinstaller"]
                install_result = subprocess.run(install_command, capture_output=True, text=True)
                self.output_panel.appendPlainText(install_result.stdout)
                if install_result.stderr:
                    self.output_panel.appendPlainText(install_result.stderr)
                if install_result.returncode != 0:
                    self.output_panel.appendPlainText("Failed to install PyInstaller.")
                    return

                self.output_panel.appendPlainText(f"Creating standalone app '{app_name}'...")
                
                # Specify PyInstaller output paths to the selected output folder
                command = [
                    sys.executable, "-m", "PyInstaller",
                    "--distpath", app_folder,
                    "--workpath", os.path.join(app_folder, "build"),
                    "--specpath", os.path.join(app_folder, "spec"),
                    "--onefile", "--noconsole",
                    script_path
                ]
                
                result = subprocess.run(command, capture_output=True, text=True)
                self.output_panel.appendPlainText(result.stdout)
                if result.stderr:
                    self.output_panel.appendPlainText(result.stderr)

                if result.returncode == 0:
                    self.output_panel.appendPlainText("\nCleaning up unnecessary files...")
                    shutil.rmtree(os.path.join(app_folder, "build"), ignore_errors=True)
                    spec_file = os.path.join(app_folder, f"{app_name}.spec")
                    if os.path.exists(spec_file):
                        os.remove(spec_file)
                        self.output_panel.appendPlainText(f"Removed spec file: {spec_file}")
                    if os.path.exists(script_path):
                        os.remove(script_path)
                        self.output_panel.appendPlainText(f"Removed script file: {script_path}")
                    self.output_panel.appendPlainText(f"\nStandalone app '{app_name}' created successfully in {app_folder}")
                else:
                    self.output_panel.appendPlainText(f"Failed to create standalone app: {result.stderr}")
            except subprocess.CalledProcessError as e:
                self.output_panel.appendPlainText(f"Failed to create standalone app: {e}")
            except Exception as e:
                self.output_panel.appendPlainText(f"Failed to create standalone app: {e}")
            finally:
                self.stop_progress()

        threading.Thread(target=task).start()

    def show_documentation(self):
        doc_text = (
            "Welcome to the Portable Python IDE!\n\n"
            "This IDE allows you to write, execute, and manage Python scripts directly from a USB drive "
            "or any portable storage. You can use it to run code, manage Python packages, and even create "
            "standalone applications with PyInstaller. Key features include:\n\n"
            "- Code Editor: A tool for writing and executing Python scripts.\n"
            "- Package Manager: Install, uninstall, and upgrade Python packages.\n"
            "- Real-time Output Panel: Provides a real-time view of code execution output.\n"
            "- Application Maintenance: Manages applications and performs backups.\n"
            "- Create Standalone App: Convert Python scripts into standalone executables.\n"
            "- Export Features: Allows exporting a list of installed packages to an Excel file.\n"
            "- Update pip: Provides an option to update pip to its latest version.\n\n"
            
            "### Debugger Commands ###\n\n"
            "Use these commands in the Debug Console during a debugging session:\n"
            "  - **n**: Next - Executes the current line and moves to the next line in the current function.\n"
            "  - **s**: Step In - Steps into the function on the current line if there's a function call.\n"
            "  - **c**: Continue - Continues execution until the next breakpoint is encountered.\n"
            "  - **q**: Quit - Stops the debugging session and terminates the debugger.\n"
            "  - **p <var>**: Print Variable - Displays the value of the specified variable.\n"
            "  - **l**: List Code - Lists code around the current execution point.\n\n"
            
            "This tool is ideal for developers who need a portable Python environment for development, "
            "debugging, and distribution of Python applications."
        )
        QMessageBox.information(self, "Documentation", doc_text)

    def show_about(self):
        about_text = (
            "Portable Python IDE Version 1.0\n\n"
            "This application is designed to provide a lightweight, portable Python environment for developers. "
            "It allows you to run scripts, manage packages, and even create standalone Python applications "
            "on the go. Simply copy the Python environment to a USB drive, plug it into any machine, and "
            "start coding. The IDE supports key features like syntax highlighting, real-time code execution, "
            "and easy package management. Created to help developers stay productive anywhere.\n\n"
            "For more information, visit our website or check the documentation section."
        )
        QMessageBox.information(self, "About", about_text)

    def start_progress(self):
        self.progress_bar.setVisible(True)

    def stop_progress(self):
        self.progress_bar.setVisible(False)

    def closeEvent(self, event):
        if self.active_process and self.active_process.poll() is None:
            reply = QMessageBox.question(self, 'Exit',
                                         "A script is still running. Do you want to terminate it and exit?",
                                         QMessageBox.Yes | QMessageBox.No, QMessageBox.No)
            if reply == QMessageBox.Yes:
                try:
                    self.active_process.terminate()
                    self.output_panel.appendPlainText("\nScript terminated by user during exit.")
                except Exception as e:
                    self.output_panel.appendPlainText(f"\nFailed to terminate process during exit: {e}")
                event.accept()
            else:
                self.output_panel.appendPlainText("Exit cancelled; script is still running.")
                event.ignore()
        else:
            event.accept()

    def select_python_folder(self):
        """Select the Python installation folder and configure the Python environment based on the selection."""
        try:
            saved_python_path = load_python_path()
            if saved_python_path and saved_python_path.exists():
                # Use previously saved path if valid
                self.python_install_folder = str(saved_python_path)
                self.python_executable = str(saved_python_path / "python.exe")
                self.pythonw_executable = str(saved_python_path / "pythonw.exe")
                self.finalize_ui()
                return

            # Prompt user to select a Python folder
            python_folder = QFileDialog.getExistingDirectory(self, "Select Python Installation Folder")
            if python_folder:
                python_executable = Path(python_folder) / "python.exe"
                if python_executable.exists():
                    # Save path and update environment variables
                    save_python_path(python_folder)
                    self.python_install_folder = python_folder
                    self.python_executable = str(python_executable)
                    self.pythonw_executable = str(Path(python_folder) / "pythonw.exe")
                    os.environ['PYTHONHOME'] = str(python_executable.parent)
                    os.environ['PYTHONPATH'] = str(python_executable.parent / 'Lib' / 'site-packages')
                    print(f"Configured Python environment at {self.python_install_folder}")
                    self.finalize_ui()
                else:
                    QMessageBox.critical(self, "Error", f"No 'python.exe' found in {python_folder}. Please select a valid Python installation.")
                    self.select_python_folder()
            else:
                QMessageBox.critical(self, "Error", "No folder selected. The application will exit.")
                sys.exit(1)
        except Exception as e:
            print(f"Error during Python folder selection: {e}")

    def perform_search(self):
        search_term = self.search_input.text()
        if not self.code_editor.search_text(search_term):
            QMessageBox.information(self, "Search", f"'{search_term}' not found.")

    def perform_replace(self):
        search_term = self.search_input.text()
        replace_text = self.replace_input.text()
        if not self.code_editor.replace_text(search_term, replace_text):
            QMessageBox.information(self, "Replace", f"No selection matches '{search_term}'.")

    def perform_replace_all(self):
        search_term = self.search_input.text()
        replace_text = self.replace_input.text()
        self.code_editor.replace_all(search_term, replace_text)

    def export_requirements_txt(self):
        """
        Exports the list of top-level installed packages to a requirements.txt file.
        Only packages explicitly installed by the user are included, avoiding dependencies.
        """
        try:
            # Define the path for requirements.txt based on the selected drive
            drive_root = self.usb_drive if self.usb_drive else "C:\\"
            ppython_folder = os.path.join(drive_root, "PPython")
            os.makedirs(ppython_folder, exist_ok=True)  # Ensure PPython folder exists

            requirements_path = os.path.join(ppython_folder, "requirements.txt")
            
            # Fetch top-level installed packages using pip
            list_command = [sys.executable, "-m", "pip", "list", "--not-required", "--format=freeze"]
            result = subprocess.run(list_command, capture_output=True, text=True)

            if result.returncode != 0:
                self.package_output_panel.appendPlainText("Failed to retrieve packages for requirements export.\n")
                self.log_error(f"pip list failed with error: {result.stderr}", panel='package')
                return

            # Check if any top-level packages are found
            if not result.stdout.strip():
                self.package_output_panel.appendPlainText("No top-level packages found to export.\n")
                return

            # Sort the requirements alphabetically for better readability
            sorted_requirements = "\n".join(sorted(result.stdout.strip().splitlines()))

            # Save sorted requirements to the defined path
            with open(requirements_path, 'w') as file:
                file.write(sorted_requirements + "\n")  # Ensure the file ends with a newline
            self.package_output_panel.appendPlainText(f"Requirements saved to {requirements_path}\n")

            # Prompt user to save to an additional location, if desired
            file_path, _ = QFileDialog.getSaveFileName(self, "Save requirements.txt As", "", "Text Files (*.txt)")
            if file_path:
                with open(file_path, 'w') as file:
                    file.write(sorted_requirements + "\n")
                self.package_output_panel.appendPlainText(f"Requirements also saved to: {file_path}\n")

        except Exception as e:
            error_message = f"Failed to export requirements.txt: {e}"
            self.package_output_panel.appendPlainText(f"{error_message}\n")
            self.log_error(error_message, panel='package')


    def import_requirements(self):
        """
        Imports packages from a requirements file and installs any missing or outdated ones.
        """
        file_path, _ = QFileDialog.getOpenFileName(self, "Select Requirements File", "", "Text Files (*.txt);;All Files (*)")
        if not file_path:
            self.package_output_panel.appendPlainText("No requirements file selected.")
            return

        try:
            self.package_output_panel.appendPlainText(f"Importing packages from {file_path}...\n")
            self.start_progress()
            QApplication.processEvents()

            # Execute pip install with requirements file
            install_command = [self.python_executable, "-m", "pip", "install", "-r", file_path]
            result = subprocess.run(install_command, capture_output=True, text=True)

            # Display the output
            if result.stdout:
                self.package_output_panel.appendPlainText(result.stdout)
            if result.stderr:
                self.package_output_panel.appendPlainText(result.stderr)

            if result.returncode == 0:
                self.package_output_panel.appendPlainText("Packages installed successfully from requirements.txt.")
            else:
                self.package_output_panel.appendPlainText("Some packages failed to install. Check the output for details.")
                self.log_error(f"Failed to install some packages from requirements.txt:\n{result.stderr}", panel='package')

        except Exception as e:
            self.package_output_panel.appendPlainText(f"Failed to import requirements: {e}")
            self.log_error(f"Failed to import requirements: {e}", panel='package')
        finally:
            self.stop_progress()


    def get_installed_packages(self):
        """
        Retrieves a set of names of all installed packages.
        """
        list_command = [sys.executable, "-m", "pip", "list", "--format=freeze"]
        result = subprocess.run(list_command, capture_output=True, text=True)
        
        if result.returncode != 0:
            self.package_output_panel.appendPlainText("Failed to retrieve installed packages.\n")
            return set()

        installed_packages = set()
        for line in result.stdout.splitlines():
            if "==" in line:
                package_name = line.split("==")[0]
                installed_packages.add(package_name.lower())
        return installed_packages

    def run_debugger(self):
        """Runs the code in the editor with pdb, allowing step-by-step debugging."""
        code = self.code_editor.toPlainText()
        if not code.strip():
            self.output_panel.appendPlainText("No code to debug.")
            return

        def debug_task():
            self.start_progress()
            self.output_panel.clear()
            self.debug_console.clear()
            self.output_panel.appendPlainText("Starting debugger...\n")

            try:
                with tempfile.NamedTemporaryFile(mode='w', suffix='.py', delete=False) as tmp_file:
                    tmp_file.write(code)
                    temp_script_path = tmp_file.name

                # Start pdb with the script
                debug_command = [sys.executable, "-m", "pdb", temp_script_path]
                self.active_process = subprocess.Popen(
                    debug_command,
                    stdin=subprocess.PIPE,
                    stdout=subprocess.PIPE,
                    stderr=subprocess.PIPE,
                    text=True,
                    bufsize=1
                )

                threading.Thread(target=self.read_debug_output, args=(self.active_process,)).start()

            except Exception as e:
                error_message = f"Debugger error: {str(e)}\n{traceback.format_exc()}"
                self.output_panel.appendPlainText(error_message)
                self.log_error(error_message, panel='editor')  # Direct to editor's output panel
            finally:
                self.stop_progress()

        self.active_thread = threading.Thread(target=debug_task)
        self.active_thread.start()

    def read_debug_output(self, process):
        """Reads output from the pdb process and updates the output panel, handles input."""
        def send_input():
            while process.poll() is None:
                if not self.command_in_progress:
                    command = self.debug_console.toPlainText().strip()
                    if command:
                        self.command_in_progress = True
                        try:
                            process.stdin.write(command + "\n")
                            process.stdin.flush()
                            self.debug_console.clear()
                            self.output_panel.appendPlainText(f"Sent command: {command}\n")
                            QApplication.processEvents()
                        except Exception as e:
                            error_message = f"Failed to send command: {e}"
                            self.output_panel.appendPlainText(f"{error_message}\n")
                            self.log_error(error_message, panel='editor')
                time.sleep(0.1)

        threading.Thread(target=send_input).start()

        try:
            for line in io.TextIOWrapper(process.stdout, encoding="utf-8"):
                self.output_panel.appendPlainText(line.strip())
                QApplication.processEvents()
                if "(Pdb)" in line:
                    self.command_in_progress = False  # Allow new command after prompt
        except Exception as e:
            error_message = f"Error reading debugger output: {e}"
            self.output_panel.appendPlainText(f"{error_message}\n")
            self.log_error(error_message, panel='editor')



if __name__ == "__main__":
    app = QApplication(sys.argv)
    app.setStyle(QStyleFactory.create('Fusion'))
    window = MainWindow()
    window.show()
    sys.exit(app.exec_())
