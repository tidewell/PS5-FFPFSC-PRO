# Running on Linux

This document explains how to run PS5 FFPFSC PRO from source on Linux and
records the fixes that make the interface work there.

## Contents

- [Requirements](#requirements)
- [Installation](#installation)
- [Starting the application](#starting-the-application)
- [Where data is stored](#where-data-is-stored)
- [Linux differences](#linux-differences)
- [Interface fixes](#interface-fixes)
- [Troubleshooting](#troubleshooting)
- [Verification results](#verification-results)
- [What has not been tested](#what-has-not-been-tested)

## Requirements

- Python 3.11 or newer with Tk support.
- A graphical session (X11 or Wayland through XWayland).
- Optional: `unrar` or `7z` for `.rar` files, and a C++ compiler if you want the
  bundled UnRAR extension.

System packages:

| Distribution | Install command |
| --- | --- |
| Fedora | `sudo dnf install python3 python3-tkinter` |
| Debian, Ubuntu | `sudo apt install python3 python3-venv python3-tk` |
| Arch | `sudo pacman -S python tk` |

## Installation

Run these commands from the repository root:

```bash
python3 -m venv .venv
.venv/bin/python -m pip install customtkinter pillow tkinterdnd2 py7zr rarfile cryptography zlib-ng
```

MkPFS does not need a separate install because the repository includes it in
`backend/mkpfs/`.

## Starting the application

```bash
.venv/bin/python PS5_FFPFSC_PRO_v1.3.0.py
```

`RUN.bat` and `BUILD_EXE.bat` are Windows only. The `py -m pip` commands shown in
some in-app messages are Windows commands too. On Linux, use
`.venv/bin/python -m pip`.

## Where data is stored

Settings and logs go to `PS5_FFPFSC_PRO_BIZKUT` under the application data
folder. `APPDATA` is not set on Linux, so the folder is:

```text
~/PS5_FFPFSC_PRO_BIZKUT/
```

| File | Content |
| --- | --- |
| `settings.json` | Saved options |
| `compatibility.json` | Local compatibility data |
| `raw_tool_output.log` | Raw backend output |

The first run wizard appears whenever `settings.json` is missing.

## Linux differences

- Completion sounds use `winsound`, which only exists on Windows, so they are
  disabled.
- Some error messages recommend NTFS and show Windows paths. On Linux, the rule
  to follow is that the output drive must allow files larger than 4 GB, so avoid
  FAT32.

## Interface fixes

### Window maximizing

**Problem.** The What's New dialog maximizes the main window on the first launch
after an update by calling `root.state("zoomed")`. That state only exists on
Windows and macOS. On Linux, Tk raises this error and the dialog does not open:

```text
TclError: bad argument "zoomed": must be iconic, normal, or withdrawn
```

**Fix.** On Linux, `_show_whats_new` sets the `-zoomed` window attribute. Other
platforms keep calling `state("zoomed")`.

### Skip buttons

**Problem.** The Skip button in the What's New dialog and the Skip Tour button in
the feature tour used `hover_color=("transparent", "transparent")`. customtkinter
6.0.0 rejects that value and fails to create the button:

```text
ValueError: transparency is not allowed in tuple color ('transparent', 'transparent'), use 'transparent'
```

**Fix.** Both buttons now use the solid color `#333333` on hover. This fix also
applies to Windows and macOS because the error comes from customtkinter itself.

**Known limitation.** The color is the same in both themes, so it looks dark when
hovering in the light theme.

### Left column scrolling

**Problem.** The left column stacks the queue and all compression options. On a
short window or a small screen, the lower options were cut off with no way to
reach them.

**Fix.** The column content now sits inside a `CTkScrollableFrame`, so it scrolls
when it does not fit. The rest of the layout is unchanged.

## Troubleshooting

| Symptom | Cause | Solution |
| --- | --- | --- |
| `ModuleNotFoundError: No module named 'tkinter'` | Tk bindings missing | Install the system package from [Requirements](#requirements), then recreate the virtual environment |
| `Missing customtkinter` on startup | Packages not installed, or wrong interpreter | Start with `.venv/bin/python` |
| Drag and drop does not work | `tkinterdnd2` missing | Install `tkinterdnd2`; the rest of the application works without it |
| `.rar` extraction fails in the main window | `rarfile` needs an external tool | Install `unrar` or `7z` |
| `cannot import name '_unrar'` in the log | Backend given a `.rar` without the compiled extension | Build the extension from `backend/unrar` |

## Verification results

Environment: Fedora 44, x86_64, Python 3.14.8, Tk 9.0, customtkinter 6.0.0, in a
new virtual environment with the packages from [Installation](#installation).

The application was started so that the What's New path runs, and callback
errors were collected for three seconds:

| Code | Callback errors |
| --- | --- |
| Before the fixes | `TclError: bad argument "zoomed"` |
| After the fixes | None |

The `hover_color` error was reproduced directly with customtkinter 6.0.0.
Packing a test folder and unpacking the result still matched the source after
the fixes.

## What has not been tested

- Other distributions. Only Fedora 44 was used, so the Debian and Arch package
  names above come from those distributions' naming and were not run.
- Wayland without XWayland.
- Drag and drop.
- The light theme.
- The left column scrolling at different window sizes and scaling factors. The
  application only started successfully with it in place.
