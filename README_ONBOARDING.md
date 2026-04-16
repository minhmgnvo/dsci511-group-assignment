 🚆 DSCI 511 Workspace: Onboarding & Developer Guide

Welcome to the team! We are using a modern Python setup for this project to ensure we all have the **exact same** environment and avoid "it works on my machine" bugs.

We use **uv** to manage our Python packages and **DVC** to manage our large data files.

---

## 🟢 Part 1: "Day One" Setup Guide
*Follow these steps in order to get your computer ready.*

### 1. Install `uv`
You need the `uv` tool installed globally on your machine.
* **Mac / Linux:**
  `brew install uv`
* **Windows (PowerShell):**
  `powershell -c "irm https://astral.sh/uv/install.ps1 | iex"`

*(Restart your terminal after installing to ensure the command works).*

### 2. Get the Code
Clone the repository to your computer:
```bash
git clone https://github.com/YOUR_USERNAME/dsci511-group-assignment.git
cd dsci511-group-assignment
```

* **What this does:** It reads the `uv.lock` file and creates a virtual environment (`.venv`) with the **exact** versions of pandas, scikit-learn, etc., that the rest of the team is using.

### 3. VS Code Setup (Crucial!)

To keep this project separate from your other classes, we recommend setting up a **Workspace**.

1. **Open VS Code** and select **File > Open Folder...**
2. Choose the `511-group-assignment` folder you just cloned.
3. **Select the Python Interpreter:**
   * Open any `.py` file (or create a dummy one).
   * Look at the bottom-right corner. If it doesn't say `.venv`, click the Python version.
   * Select **Python 3.xx (Recommended) ./.venv/bin/python**.

4. **Save as Workspace:**
   * Go to **File > Save Workspace As...**
   * Name it: `511-group-assignment.code-workspace`.
   * Save it **inside** the project folder.

**✅ Success:** From now on, whenever you want to work on this project, just open the `511-group-assignment.code-workspace` file.

### 4. Build the Environment (`uv sync`)

This is the magic command. Instead of creating a venv and pip installing manually, just run:

```bash
uv sync
```
## ⚡ Part 2: Daily Workflow (uv & Libraries)

### How to install a new library

**Do NOT use `pip install`.** Use `uv add`. This updates the lockfile so everyone else gets the library too.

```bash
uv add library_name
# Example: uv add matplotlib
```

### ⚠️ How to save that change (Important!)

After running `uv add`, two files on your computer will change: `pyproject.toml` (the list of tools) and `uv.lock` (the specific versions). You **must** commit these changes so the team gets them.

Run these three commands immediately after adding a library:

```bash
# 1. Stage the configuration files
git add pyproject.toml uv.lock

# 2. Save the snapshot
git commit -m "Added matplotlib library"

# 3. Send it to the team
git push
```

### How to sync changes from the team

If you pull from GitHub and someone else added a library, your environment is now out of date. Update it instantly:

```bash
uv sync
```

---
