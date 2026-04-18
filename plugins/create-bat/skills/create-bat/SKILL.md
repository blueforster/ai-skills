---
name: create-bat
description: >
  Use this skill whenever the user asks to create, write, or generate a Windows .bat or .cmd launcher file —
  including "one-click start", "double-click to run", "startup script", "Windows batch file", or any request
  to make a .bat that launches an app, starts a server, installs dependencies, or opens a browser.
  This skill enforces all WSL-to-Windows bat authoring rules (CRLF endings, ASCII-only, self-relaunch,
  .cmd trap avoidance, correct label syntax, no goto inside blocks, proper blank-line echoing, delayed
  expansion, and version-string parsing). ALWAYS use this skill when writing .bat files — do not write
  them freehand without these rules.
---

# create-bat Skill

You are writing a Windows `.bat` file that will be authored from WSL/Linux and run on Windows CMD.
This context introduces silent failure modes that are not obvious. Follow every rule below without exception.

## Step 0 — Read the full rules first

Before writing a single line of bat code, read the reference file:

```
references/bat_rules.md
```

It contains 13 rules with the exact failure modes and fixes. All 13 rules apply to every bat file you create.

## Step 1 — Plan the script

Understand what the bat must do:
- What tool/app does it launch?
- What prerequisites must exist (Node.js, Python, a specific folder)?
- Does the user double-click it, or run it from a terminal?
- What working directory should be used?

Answer these before writing code. Ask the user if anything is unclear.

## Step 1A — Auto-detect project type and create install.bat + start.bat

**Trigger:** Whenever create-bat is invoked from a project directory (i.e. the current working directory contains a recognisable project manifest), always create BOTH `install.bat` and `start.bat` in addition to any other bat file the user requested. Do not ask — just do it.

### 1. Detect project type

Check the current working directory for these files (in priority order):

| File present | Project type | Installer | Start command |
|---|---|---|---|
| `package.json` | Node.js / JS | `npm install` | inspect `scripts` field for `dev`, `start`, `serve`, `preview` (prefer `dev`) |
| `requirements.txt` or `pyproject.toml` or `setup.py` | Python | `pip install -r requirements.txt` or `pip install -e .` | look for `main.py`, `app.py`, `run.py`, `manage.py`, `uvicorn`, `flask`, `fastapi`, `streamlit` |
| `Cargo.toml` | Rust | `cargo build --release` | `cargo run` |
| `go.mod` | Go | `go mod download` | `go run .` |
| `pom.xml` | Java/Maven | `mvn install -DskipTests` | `mvn spring-boot:run` or `java -jar target/*.jar` |
| `build.gradle` or `build.gradle.kts` | Java/Gradle | `gradlew build` | `gradlew bootRun` or `java -jar build/libs/*.jar` |
| `Gemfile` | Ruby | `bundle install` | `bundle exec rails server` or `bundle exec ruby app.rb` |
| `composer.json` | PHP | `composer install` | `php -S localhost:8000` or `php artisan serve` |

If multiple manifests exist, list each type found and generate combined install/start logic.

### 2. Inspect package.json for Node.js projects

Read `package.json` to determine:
- Which start script to use (check `scripts.dev` → `scripts.start` → `scripts.serve` → `scripts.preview`, in that order)
- Which port the app uses (scan script values for `--port`, `-p`, `:3000`, `:5173`, `:8080`, etc.)
- Whether to open a browser after starting (open only if a port is found)

### 3. Inspect requirements.txt / pyproject.toml for Python projects

Scan dependencies to determine framework:
- `flask` → `flask run` or `python app.py`
- `fastapi` or `uvicorn` → `uvicorn main:app --reload` or `uvicorn app:app --reload`
- `streamlit` → `streamlit run app.py` (or whatever the main `.py` is)
- `django` → `python manage.py runserver`
- Otherwise → `python main.py` (or `python app.py`, whichever exists)

### 4. Windows path for the project directory

Convert the current WSL path to a Windows path using Python:

```python
import subprocess, os
cwd = os.getcwd()
win_path = subprocess.check_output(['wslpath', '-w', cwd]).decode().strip()
# e.g. cwd = /mnt/c/Users/forst/myapp  ->  win_path = C:\Users\forst\myapp
```

Use `win_path` as `APP_DIR` in both bat files.

### 5. install.bat — template

```python
content = (
    "@echo off\r\n"
    "setlocal EnableDelayedExpansion\r\n"
    "if \"%~1\"==\"\" (\r\n"
    "    cmd /k \"\"%~f0\" run\"\r\n"
    "    exit /b\r\n"
    ")\r\n"
    "\r\n"
    "set \"APP_DIR=<WIN_PATH>\"\r\n"
    "\r\n"
    "echo ========================================\r\n"
    "echo  Installing dependencies\r\n"
    "echo ========================================\r\n"
    "echo(\r\n"
    "\r\n"
    # --- tool existence check (e.g. node, python) ---
    "where <tool> >nul 2>&1\r\n"
    "if errorlevel 1 (\r\n"
    "    set \"ABORT=1\"\r\n"
    "    echo  [FAIL] <Tool> not found. Install it first.\r\n"
    ")\r\n"
    "if \"!ABORT!\"==\"1\" goto :end\r\n"
    "\r\n"
    # --- install command (wrapped in cmd /c for npm; python -m pip for pip) ---
    "echo  Installing packages\r\n"
    "start /wait \"\" /D \"%APP_DIR%\" cmd /c \"<install command>\"\r\n"
    "if errorlevel 1 (\r\n"
    "    set \"INSTALL_FAILED=1\"\r\n"
    "    echo  [FAIL] Install failed.\r\n"
    ")\r\n"
    "if \"!INSTALL_FAILED!\"==\"1\" goto :end\r\n"
    "\r\n"
    "echo(\r\n"
    "echo  Done. Dependencies installed.\r\n"
    "goto :done\r\n"
    "\r\n"
    ":end\r\n"
    "echo(\r\n"
    "echo  Install failed. Check errors above.\r\n"
    "echo(\r\n"
    "pause >nul\r\n"
    "\r\n"
    ":done\r\n"
)
```

### 6. start.bat — template

```python
content = (
    "@echo off\r\n"
    "setlocal EnableDelayedExpansion\r\n"
    "if \"%~1\"==\"\" (\r\n"
    "    cmd /k \"\"%~f0\" run\"\r\n"
    "    exit /b\r\n"
    ")\r\n"
    "\r\n"
    "set \"APP_DIR=<WIN_PATH>\"\r\n"
    "\r\n"
    "echo ========================================\r\n"
    "echo  Starting <Project Name>\r\n"
    "echo ========================================\r\n"
    "echo(\r\n"
    "\r\n"
    # --- optional: check dependencies are installed ---
    # e.g. for Node: if not exist "%APP_DIR%\node_modules\." -> prompt to run install.bat
    "if not exist \"%APP_DIR%\\<deps_marker>\" (\r\n"
    "    set \"ABORT=1\"\r\n"
    "    echo  [WARN] Dependencies not installed. Run install.bat first.\r\n"
    ")\r\n"
    "if \"!ABORT!\"==\"1\" goto :end\r\n"
    "\r\n"
    # --- start server in new window ---
    "echo  Starting server\r\n"
    "start \"<Project Name>\" /D \"%APP_DIR%\" cmd /k \"<start command>\"\r\n"
    "\r\n"
    # --- optional browser open after delay (only if port known) ---
    "ping -n 4 127.0.0.1 >nul 2>&1\r\n"
    "start \"\" \"http://localhost:<PORT>\"\r\n"
    "\r\n"
    "echo  Server started. Browser opening\r\n"
    "goto :done\r\n"
    "\r\n"
    ":end\r\n"
    "echo(\r\n"
    "echo  Could not start. Check errors above.\r\n"
    "echo(\r\n"
    "pause >nul\r\n"
    "\r\n"
    ":done\r\n"
)
```

**Key rules for these templates:**
- `node_modules\.` for Node dependency check (Rule 23 — directory check)
- `start /wait "" /D "%APP_DIR%" cmd /c "npm install"` for install (Rule 4 — .cmd trap)
- `python -m pip install -r requirements.txt` for Python (Rule 4 — pip.cmd trap)
- `start "Title" /D "%APP_DIR%" cmd /k "<cmd>"` for server (stays open)
- Browser: `start "" "http://localhost:<PORT>"` only when port is known (Rule 21 — empty title)
- Omit browser open for non-web projects (CLI tools, Rust binaries, etc.)

### 7. Write both files in the project directory

```python
import subprocess, os

cwd = os.getcwd()
win_path = subprocess.check_output(['wslpath', '-w', cwd]).decode().strip()

# Write install.bat
install_path = os.path.join(cwd, 'install.bat')
with open(install_path, 'wb') as f:
    f.write(install_content.encode('ascii'))

# Write start.bat
start_path = os.path.join(cwd, 'start.bat')
with open(start_path, 'wb') as f:
    f.write(start_content.encode('ascii'))

print(f"install.bat -> {install_path}")
print(f"start.bat   -> {start_path}")
```

Verify both files with the Step 5 assertions (CRLF, no lone LF, no non-ASCII, no `/dev/null`).

## Step 2 — Write the bat content as a Python string

Never write the bat file as a text file from WSL — that produces LF endings which silently break `goto`.
Always write it as a Python bytes literal using `\r\n` so every line ends with CRLF.

**Template to adapt for every bat file:**

```python
content = (
    "@echo off\r\n"
    "setlocal EnableDelayedExpansion\r\n"
    # Self-relaunch block (Rule 3) — keeps window open on double-click
    "if \"%~1\"==\"\" (\r\n"
    "    cmd /k \"\"%~f0\" run\"\r\n"
    "    exit /b\r\n"
    ")\r\n"
    "\r\n"
    # ... your logic here, one \r\n per line ...
    "\r\n"
    ":end\r\n"
    "echo(\r\n"
    "pause >nul\r\n"
    "\r\n"
    ":done\r\n"
)
```

## Step 3 — Apply all 24 rules while writing (applies to install.bat and start.bat too)

Quick checklist — verify each before moving to Step 4:

| # | Rule | What to check |
|---|------|---------------|
| 1 | CRLF endings | Every string ends with `\r\n`, never `\n` alone |
| 2 | ASCII only | No CJK, no box-drawing chars, no em-dashes, no accents |
| 3 | Self-relaunch | `if "%~1"==""` block at top with `cmd /k` |
| 4 | .cmd trap | `where node` not `node --version`; `npm` calls wrapped in `cmd /c "..."` or `start /wait "" /D "..." cmd /c "..."`; use `python -m pip` not `pip` |
| 5 | Label syntax | Labels use `:label` (one colon), never `::label` |
| 6 | No goto in blocks | `goto` only at the top level, never inside `( )` — use `set "FLAG=1"` inside the block, then `if "!FLAG!"=="1" goto :label` outside |
| 7 | No hanging commands | Never run `nvidia-smi`, `wmic`, or `python -c "import torch"` — use `if exist` or `where` instead |
| 8 | Blank lines | `echo(` for standalone blank lines; `echo.` only inside `( ) > file` redirect blocks |
| 9 | python -c quoting | Never use `python -c "multi-statement"` — write a temp `.py` file or use `python -m` |
| 10 | errorlevel | `if errorlevel 1` means >=1 (failure); `if not errorlevel 1` means ==0 (success) |
| 11 | !VAR! in blocks | With `EnableDelayedExpansion`, use `!VAR!` (not `%VAR%`) for vars set inside `( )` |
| 12 | Version strings | Use `node -e "process.exit(...>=18?0:1)"` not `for /f "delims=v"` |
| 13 | Echo content | No trailing `...`, no `--`, no `—`, no trailing `.` on status lines |
| 14 | Quoted paths | Double-quote every path in `cd`, `if exist`, `start`, `for /f`, variable expansions |
| 15 | cd /d | Always `cd /d "%PATH%"` — plain `cd` silently stays on current drive |
| 16 | Quoted set | Always `set "VAR=value"` — prevents trailing spaces and special-char contamination |
| 17 | rem in blocks | Use `rem` for comments inside `( )` — `::` causes syntax errors inside blocks |
| 18 | exit /b | Use `exit /b` (not bare `exit`) — bare `exit` closes the whole CMD window |
| 19 | Redirect order | Write `>nul 2>&1` not `2>&1 >nul` — wrong order leaves stderr visible |
| 20 | ERRORLEVEL var | Use `if errorlevel 1` not `if "%ERRORLEVEL%"=="1"` — var form can be shadowed |
| 21 | start "" title | `start "" "path"` — empty first arg required; quoted path alone sets window title |
| 22 | call for .bat | Use `call other.bat` or `call :label` — bare invocation transfers execution permanently |
| 23 | Directory check | Use `if exist "%DIR%\."` to check directory (not file); plain `if exist "%DIR%"` matches both |
| 24 | `>nul` not `>/dev/null` | `/dev/null` does not exist on Windows CMD — use `>nul 2>&1`; verify with `assert b'/dev/null' not in data` |

## Step 4 — Write the file using Python binary mode

Run this Python snippet to write the file — **never use the Write tool or shell redirection** for bat files:

```python
import sys

content = (
    # your assembled content here
)

output_path = '/mnt/c/Users/.../your-script.bat'  # adjust path

with open(output_path, 'wb') as f:
    f.write(content.encode('ascii'))

print(f"Written: {len(content)} bytes")
```

> Use `.encode('ascii')` — if any character is non-ASCII, this will raise an error immediately, catching Rule 2 violations before they reach Windows.

## Step 5 — Verify before reporting done

After writing, always run this verification and report the results to the user:

```python
data = open(output_path, 'rb').read()

crlf   = data.count(b'\r\n')
lf_only = data.count(b'\n') - crlf
non_ascii = [hex(c) for c in data if c > 127]

print(f"CRLF pairs : {crlf}")
print(f"Lone LF    : {lf_only}   <-- must be 0")
print(f"Non-ASCII  : {non_ascii} <-- must be []")

assert lf_only == 0,    "FAIL: lone LF found — goto will break"
assert non_ascii == [], f"FAIL: non-ASCII bytes found: {non_ascii}"
print("OK: file is safe to run on Windows CMD")
```

Only report success after both assertions pass.

## Common patterns

### Check if a tool exists (Rule 4 & 7)
```bat
where node >nul 2>&1
if errorlevel 1 (
    set "ABORT=1"
    echo  [FAIL] Node.js not found. Install from https://nodejs.org/
)
if "!ABORT!"=="1" goto :end
```

### Run npm install safely (Rule 4)
```bat
:: npm.cmd uses bare "exit" which kills parent — wrap in cmd /c
start /wait "" /D "%APP_DIR%" cmd /c "npm install"
if errorlevel 1 (
    set "INSTALL_FAILED=1"
    echo  [FAIL] npm install failed.
)
if "!INSTALL_FAILED!"=="1" goto :end
```

### Check Node version >= 18 (Rule 12)
```bat
node -e "process.exit(parseInt(process.version.slice(1))>=18?0:1)" >nul 2>&1
if errorlevel 1 (
    set "ABORT=1"
    echo  [FAIL] Node.js 18 or higher required.
)
if "!ABORT!"=="1" goto :end
```

### Start a dev server in a new window and open browser
```bat
start "My App Server" /D "%APP_DIR%" cmd /k "npm run dev"
ping -n 6 127.0.0.1 >nul 2>&1
start "" "http://localhost:3000"
```

### Error/done exit pattern (Rule 5 & 6)
```bat
:: ... main logic above, using "goto :end" only at top level ...
goto :done

:end
echo(
echo  Startup failed. Check the errors shown above.
echo(
pause >nul

:done
```

## Output

When done, tell the user:
1. The full paths of every `.bat` file created (the originally-requested file plus `install.bat` and `start.bat` when auto-generated)
2. The verification results for each file (CRLF count, lone LF = 0, non-ASCII = [])
3. How to use each file:
   - `install.bat` — double-click (or run from CMD) to install all dependencies
   - `start.bat` — double-click (or run from CMD) to start the app; opens the browser automatically if a port was detected
   - Any other bat file — its specific purpose
