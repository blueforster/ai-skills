# Windows Batch File (.bat) Writing Guide
# (For files authored in WSL2 / Linux and run on Windows CMD)

---

## 1. CRLF Line Endings — ALWAYS Required

**Problem:** Files written from WSL/Linux have LF (`\n`) endings. Windows CMD requires CRLF (`\r\n`) for `goto` label jumps to work. With LF endings, `goto` silently terminates the script instead of jumping.

**Rule:** After every edit to a .bat file from WSL, convert to CRLF:

```python
data = open('file.bat', 'rb').read()
data = data.replace(b'\r\n', b'\n').replace(b'\n', b'\r\n')
open('file.bat', 'wb').write(data)
```

---

## 2. ASCII Only — No Unicode

**Problem:** CMD crashes immediately on double-click if the .bat file contains any non-ASCII characters, even with `chcp 65001` at the top. This includes:
- Box-drawing chars: `╔ ║ ╚ ┌ │ └`
- Em-dash / en-dash: `— –`
- Any accented letters, CJK characters, emoji

**Rule:** Use only plain ASCII in .bat files. Replace decorative characters:
- `—` → `--` or just remove
- Box borders → `=` or `-`

---

## 3. Double-Click Auto-Close

**Problem:** Explorer launches `.bat` with `cmd /c`, which closes the window immediately when the script exits — even on error.

**Fix:** Add self-relaunch at the very top of the script:

```bat
@echo off
setlocal EnableDelayedExpansion
if "%~1"=="" (
    cmd /k ""%~f0" run"
    exit /b
)
```

This re-opens the script inside a persistent `cmd /k` window when double-clicked.

---

## 4. .cmd File Trap — The Silent Killer

**Problem:** On Windows, many tools are `.cmd` wrappers (e.g. `npm.cmd`, and sometimes `pip.cmd`). When a `.cmd` file internally calls `exit` (not `exit /b`), it terminates the **entire parent batch session**, not just itself. The script silently stops mid-execution with no error message.

**Affected commands (confirmed):**
- `npm --version` → kills parent script
- `npm install` → kills parent script after finishing
- `pip show torch` → kills parent script (when pip resolves to pip.cmd)
- Any `.cmd` file that uses bare `exit`

**Rules:**
- Never call `npm --version` to check npm. Use `where npm >nul 2>&1` instead.
- Never call `pip <subcommand>` directly when checking package state. Use `python -m pip <subcommand>` — this runs pip through python.exe, bypassing any .cmd wrapper.
- When in doubt, use `where <tool> >nul 2>&1` (checks PATH existence without running the tool).

```bat
:: BAD - kills parent script
npm --version >nul 2>&1

:: GOOD - only checks PATH, never runs npm.cmd
where npm >nul 2>&1

:: BAD - kills parent script after finishing
npm install

:: GOOD - npm runs in child CMD; its exit does not affect parent
cmd /c "npm install"

:: BAD
pip show torch >nul 2>&1

:: GOOD
python -m pip show torch >nul 2>&1
```

---

## 5. goto Labels — Syntax Rules

**Problem:** `::label` looks like a label but is actually a comment. `goto :label` cannot find it and **silently terminates** the script.

**Rule:** Labels must use a single colon at the start of the line, with nothing before them:

```bat
:: BAD - this is a comment, not a label
::my_label

:: GOOD
:my_label
```

Also: if `goto` cannot find the target label (e.g. due to LF endings — see rule 1), the script silently exits. Always verify labels exist AND that the file has CRLF.

---

## 6. Avoid goto Inside if/else Blocks

**Problem:** `goto` inside a parenthesized `( )` block is unreliable in CMD, especially combined with `setlocal EnableDelayedExpansion`. It can exit the script instead of jumping.

**Rule:** Prefer `if/else` nesting over `goto` for flow control inside a block:

```bat
:: RISKY
if not errorlevel 1 (
    set "DONE=1"
    goto :next_section
)

:: SAFER
if not errorlevel 1 (
    set "DONE=1"
) else (
    :: do the install work here
)
:: falls through naturally — no goto needed
```

---

## 7. Commands That Hang

**Problem:** Some commands hang indefinitely in certain environments, making the script appear frozen:

| Command | Problem | Alternative |
|---------|---------|-------------|
| `nvidia-smi` (run) | Hangs if driver issues | `if exist "path\nvidia-smi.exe"` |
| `wmic path win32_VideoController \| findstr nvidia` | wmic hangs on some systems | Ask user directly with `set /p` |
| `python -c "import torch"` | Hangs on slow CUDA init | `python -m pip show torch` |

**Rule:** Never run a potentially-slow detection command silently. Either use `if exist` (file presence only), `where` (PATH lookup only), or ask the user directly.

---

## 8. Blank Lines in Output

**Problem:** `echo.` occasionally outputs a literal `.` character if a file named `.` exists in the current directory. `echo(` is more reliable but can confuse CMD's parser inside `( ) > file` redirect blocks.

**Rules:**
- Standalone blank line: use `echo(`
- Inside `( ) > file` redirect blocks: use `echo.`
- Never use `echo ` (echo followed by a space) — it prints `ECHO is off` or similar

```bat
:: standalone blank line
echo(

:: inside a file redirect block
(
    echo Line 1
    echo.
    echo Line 2
) > output.txt
```

---

## 9. python -c Quoting Issues

**Problem:** `python -c "code"` in CMD has fragile quote parsing. Semicolons, parentheses, or special characters inside the quoted code string can cause CMD to misparse the line and try to run part of the string as a separate command.

**Rule:** Never use `python -c "multi-statement code"` in batch files. Use one of these instead:

```bat
:: Option A: temp script file
echo import sys > "%TEMP%\check.py"
echo sys.exit(0) >> "%TEMP%\check.py"
python "%TEMP%\check.py" >nul 2>&1

:: Option B: python -m (for pip/module tasks)
python -m pip show torch >nul 2>&1

:: Option C: simple single import (OK if no special chars)
python -c "import torch" >nul 2>&1
```

---

## 10. errorlevel Checking

**Rule:** CMD's `if errorlevel` is "greater-than-or-equal", not "equal":

```bat
:: TRUE if errorlevel >= 1 (i.e. any error)
if errorlevel 1 ( echo failed )

:: TRUE if errorlevel == 0 (success)
if not errorlevel 1 ( echo success )

:: Check for specific code (e.g. 2)
if errorlevel 2 ( echo code is 2 or higher )
if not errorlevel 3 ( echo code is less than 3 )
```

---

## 11. Variable Expansion Inside Blocks

**Rule:** With `setlocal EnableDelayedExpansion`, use `!VAR!` (not `%VAR%`) to read variables that were set **inside** the same `( )` block:

```bat
setlocal EnableDelayedExpansion
set "RESULT=0"
if 1==1 (
    set "RESULT=1"
    echo inside block: !RESULT!   <- correct: shows 1
    echo inside block: %RESULT%   <- wrong: shows 0 (parsed at block entry)
)
echo after block: !RESULT!        <- correct: shows 1
```

---

## 12. for /f Version String Parsing

**Problem:** Version strings like `v18.0.0` — if you split by delimiter `v`, the first token is empty.

**Rule:** Strip the leading `v` with string substitution instead:

```bat
:: BAD
for /f "delims=v tokens=1" %%V in ('node --version') do set "VER=%%V"
:: VER is empty because "v18.0.0" split by "v" gives "" as first token

:: GOOD
for /f %%V in ('node --version') do set "NODE_VER=%%V"
set "NODE_MAJOR=!NODE_VER:~1!"   :: strips leading "v" -> "18.0.0"

:: EVEN BETTER: let the tool check its own version
node -e "process.exit(parseInt(process.version.slice(1))>=18?0:1)" >nul 2>&1
```

---

## 13. Avoid Special Characters in echo Output

**Problem:** Characters like `--`, `...`, `—`, or trailing periods in `echo` lines can confuse users who interpret them as script errors, or in rare cases trigger encoding issues.

**Rule:** Keep echo output simple and unambiguous:
- No trailing `...` on status lines
- No `--` (use a space or nothing)
- No em-dashes `—`
- No trailing period `.` on status messages

```bat
:: BAD
echo  Checking for NVIDIA GPU...
echo  No NVIDIA GPU detected -- installing CPU version.

:: GOOD
echo  Checking for NVIDIA GPU...
echo  No NVIDIA GPU detected, installing CPU version
```

---

## 14. Paths with Spaces Must Be Quoted

**Problem:** Any path containing spaces will be split into separate tokens by CMD, causing the command to fail silently or with a confusing error.

**Rule:** Always double-quote every path used in `cd`, `if exist`, `start`, `for /f`, and variable expansions.

```bat
:: BAD - fails if path is "C:\My Projects\app"
cd %APP_DIR%
if exist %APP_DIR%\package.json

:: GOOD
cd /d "%APP_DIR%"
if exist "%APP_DIR%\package.json"
```

---

## 15. `cd /d` for Cross-Drive Navigation

**Problem:** Plain `cd "D:\path"` silently does nothing if the current drive is `C:`. CMD's `cd` only changes the directory within the current drive.

**Rule:** Always use `cd /d "path"` to change both drive and directory at once.

```bat
:: BAD - silently stays on C: if script is run from C:
cd "D:\my-project"

:: GOOD
cd /d "D:\my-project"

:: Also works with variables
cd /d "%APP_DIR%"
```

---

## 16. Quoted `set` to Prevent Contamination

**Problem:** `set VAR=value` — any trailing space on the line becomes part of the value.
`set VAR =value` — sets the variable named `VAR ` (with a trailing space), not `VAR`.
Both bugs are invisible in the source and cause subtle mismatches later.

**Rule:** Always use the quoted form `set "VAR=value"`. It prevents trailing spaces and also safely handles values that contain `&`, `|`, `<`, `>` characters.

```bat
:: BAD - trailing space after "folder" becomes part of APP_DIR
set APP_DIR=C:\my folder

:: BAD - sets "VAR " not "VAR"
set VAR =hello

:: GOOD
set "APP_DIR=C:\my folder"
set "VAR=hello"

:: To clear a variable safely
set "APP_DIR="
```

---

## 17. `rem` not `::` Inside Blocks

**Problem:** `::` is shorthand for a label comment. Inside a `( )` block, CMD tries to parse it as a label, which causes a syntax error or silently breaks the block on some CMD versions.

**Rule:** Use `rem` for comments inside any `( )` block. Reserve `::` for top-level comments only.

```bat
:: BAD - causes errors inside ( ) in some CMD versions
if exist "%DIR%" (
    :: do something
    echo Found
)

:: GOOD
if exist "%DIR%" (
    rem do something
    echo Found
)
```

---

## 18. `exit /b` vs Bare `exit`

**Problem:** Bare `exit` with no argument closes the **entire CMD window**, which terminates the parent `cmd /k` session and loses any previous terminal history. This is almost never what you want inside a launcher script.

**Rule:**
- Use `exit /b` at the end of a script or subroutine — it returns control to the caller.
- Use `exit /b 0` for success, `exit /b 1` for failure.
- Bare `exit` is only appropriate at the very top of the self-relaunch block (where `exit /b` is already used correctly).

```bat
:: BAD - closes the whole CMD window
goto :end
:end
exit

:: GOOD - exits current script, caller continues
goto :end
:end
exit /b

:: Subroutine return
:my_function
rem ... do work ...
exit /b 0
```

---

## 19. Redirect Order: `>nul 2>&1`

**Problem:** `2>&1 >nul` is a common mistake. It redirects stderr to the current stdout (the terminal) first, then redirects stdout to nul — but stderr still goes to the terminal.

**Rule:** Always write `>nul 2>&1` (stdout to nul first, then stderr follows stdout).

```bat
:: BAD - stderr still visible on terminal
some-command 2>&1 >nul

:: GOOD - both stdout and stderr silenced
some-command >nul 2>&1

:: Same rule applies for file redirection
some-command > output.txt 2>&1
```

---

## 20. `%ERRORLEVEL%` Pseudo-Variable Trap

**Problem:** `%ERRORLEVEL%` is a dynamic pseudo-variable, but if any earlier code ran `set "ERRORLEVEL=0"` (even accidentally), it becomes a regular variable that shadows the real errorlevel and always returns the stored value.

**Rule:** Never test errorlevel with `if "%ERRORLEVEL%"=="0"`. Always use the `if errorlevel N` syntax, which reads the real errorlevel register regardless of any variable named ERRORLEVEL.

```bat
:: BAD - can be fooled by a prior "set ERRORLEVEL=0"
if "%ERRORLEVEL%"=="0" echo success
if "%ERRORLEVEL%" NEQ "0" echo failed

:: GOOD - always reads the real errorlevel register
if not errorlevel 1 echo success
if errorlevel 1 echo failed
```

---

## 21. `start ""` Title Argument for Files and URLs

**Problem:** `start` treats the first double-quoted argument as the **window title**, not as the target. So `start "http://localhost:5173"` opens a CMD window titled "http://localhost:5173" and launches nothing. Same trap for `.exe` paths.

**Rule:** Always pass an empty title `""` as the first argument when the target is quoted.

```bat
:: BAD - CMD opens a new window titled "http://localhost:5173" and runs nothing
start "http://localhost:5173"
start "C:\Program Files\app.exe"

:: GOOD - empty title, then the actual target
start "" "http://localhost:5173"
start "" "C:\Program Files\app.exe"

:: start with /D working-dir also needs ""
start "My App" /D "%APP_DIR%" cmd /k "npm run dev"
rem ^^^^ title is "My App", /D sets working dir — this is intentional
```

---

## 22. `call` for Other .bat Files and Subroutines

**Problem:** Invoking another `.bat` file or a label directly (without `call`) transfers execution permanently — the calling script never resumes after the callee finishes.

**Rule:**
- `call other.bat` — runs another bat file and returns here when done.
- `call :label` — runs an internal subroutine and returns here when done.
- End every subroutine with `exit /b` (not `goto :eof` alone) so control returns cleanly.

```bat
:: BAD - "setup.bat" runs, then script ends; "echo Done" is never reached
setup.bat
echo Done

:: GOOD
call setup.bat
echo Done

:: Internal subroutine
call :check_node
echo After check

:check_node
where node >nul 2>&1
if errorlevel 1 ( echo Node not found )
exit /b
```

---

## 23. `if exist` — Directory vs File Check

**Problem:** `if exist "%DIR%"` matches **both files and directories** with that name. If a file named `node_modules` (no extension) exists, the check passes incorrectly. Also, the check does not distinguish between "this is a directory" and "this is a file".

**Rule:** Append `\.` to the path when checking for a directory. The `.` refers to the directory entry itself, which only exists inside directories.

```bat
:: BAD - matches a file or directory named "node_modules"
if exist "%APP_DIR%\node_modules" (

:: GOOD - only matches a directory
if not exist "%APP_DIR%\node_modules\." (
    rem directory is missing, run npm install
)

:: Same for checking any directory
if not exist "%APP_DIR%\." (
    echo ERROR: folder not found: %APP_DIR%
)
```

---

## 24. `>nul` not `>/dev/null`

**Problem:** When authoring `.bat` files from WSL or any Unix-like environment, it is easy to accidentally write the Linux null-device redirect `>/dev/null 2>&1`. Windows CMD does not have a `/dev/null` device. It tries to write to the literal path `C:\dev\null` (or whichever drive is current). If `C:\dev\` does not exist, CMD prints:

```
The system cannot find the path specified.
```

This error message appears on the terminal, pollutes the output, and can also corrupt `%ERRORLEVEL%` in some CMD versions — causing subsequent `if errorlevel 1` checks to misbehave.

**Rule:** Always use `>nul 2>&1` (Windows CMD null device) in `.bat` files. Never use `>/dev/null`.

```bat
:: BAD - writes to C:\dev\null which does not exist
where node >/dev/null 2>&1
ping -n 6 127.0.0.1 >/dev/null 2>&1

:: GOOD
where node >nul 2>&1
ping -n 6 127.0.0.1 >nul 2>&1
```

**How to catch it:** When verifying the finished file with Python, search for the byte string `/dev/null`:

```python
data = open('file.bat', 'rb').read()
assert b'/dev/null' not in data, "Found Linux /dev/null — replace with >nul"
```

---

## Checklist Before Saving a .bat File

- [ ] All characters are ASCII only (no Unicode)
- [ ] File converted to CRLF after every edit
- [ ] Self-relaunch block at top (if double-click launch is needed)
- [ ] All labels use single `:` (not `::`)
- [ ] No `.cmd` tools called directly for version checks — use `where` or `python -m`
- [ ] No `python -c "complex code"` — use temp file or `python -m`
- [ ] No piped commands that might hang (`wmic`, `nvidia-smi`)
- [ ] `if/else` nesting preferred over `goto` inside blocks
- [ ] `echo(` for blank lines (standalone), `echo.` inside redirect blocks
- [ ] `!VAR!` used inside blocks when `EnableDelayedExpansion` is active
- [ ] All paths are double-quoted
- [ ] `cd /d` used instead of `cd` for cross-drive navigation
- [ ] `set "VAR=value"` quoted form used throughout
- [ ] `rem` used for comments inside `( )` blocks (not `::`)
- [ ] `exit /b` used at script end (not bare `exit`)
- [ ] Redirects written as `>nul 2>&1` (not `2>&1 >nul`)
- [ ] `if errorlevel 1` used (not `if "%ERRORLEVEL%"=="1"`)
- [ ] `start ""` used before any quoted path/URL
- [ ] `call` used when invoking other .bat files or subroutines
- [ ] Directory existence checked with `\."` suffix
- [ ] No `>/dev/null` anywhere — use `>nul 2>&1` (Windows CMD null device)
