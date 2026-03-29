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

## Step 3 — Apply all 24 rules while writing

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
1. The full path of the `.bat` file created
2. The verification results (CRLF count, lone LF = 0, non-ASCII = [])
3. How to use it (double-click, or what it does step by step)
