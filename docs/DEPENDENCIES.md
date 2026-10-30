# Dependencies

This document lists everything PS5 FFPFSC PRO depends on, explains how each
dependency is loaded, records what changed in the latest update, and describes
how to repeat the update.

## Contents

- [Overview](#overview)
- [Vendored dependencies](#vendored-dependencies)
- [Python packages](#python-packages)
- [How MkPFS is located at runtime](#how-mkpfs-is-located-at-runtime)
- [RAR extraction paths](#rar-extraction-paths)
- [What changed from MkPFS 0.0.9 to 1.1.0](#what-changed-from-mkpfs-009-to-110)
- [What changed from UnRAR 7.21 to 7.23](#what-changed-from-unrar-721-to-723)
- [How to update MkPFS](#how-to-update-mkpfs)
- [How to update UnRAR](#how-to-update-unrar)
- [Verification results](#verification-results)
- [What has not been tested](#what-has-not-been-tested)

## Overview

| Name | Version | Kind | Location |
| --- | --- | --- | --- |
| MkPFS | 1.1.0 | Vendored Python package | `backend/mkpfs/` |
| UnRAR | 7.23 | Vendored C++ source | `backend/unrar/src/` |
| customtkinter | 6.0.0 | PyPI package | Python environment |
| Pillow | 13.0.0 | PyPI package | Python environment |
| tkinterdnd2 | 0.7.0 | PyPI package | Python environment |
| py7zr | 1.1.4 | PyPI package | Python environment |
| rarfile | 4.5 | PyPI package | Python environment |
| cryptography | 50.0.2 | PyPI package | Python environment |
| zlib-ng | 1.0.0 | PyPI package | Python environment |
| pypresence | 4.6.2 | PyPI package | Python environment |
| isal | 1.8.0 | PyPI package, pulled in by `mkpfs` | Python environment |

The PyPI versions are the versions the project was last tested with. Only
`mkpfs` is pinned. `RUN.bat` and `BUILD_EXE.bat` install the other packages
without a version, so they always receive the newest release.

Use Python 3.11 or newer. Pillow 13 requires 3.11, and `mkpfs` only falls back
to a package that is not among its dependencies (`backports.strenum`) on
versions older than 3.11, even though its metadata allows 3.9. Testing was done
on Python 3.14.8.

## Vendored dependencies

Vendored code is copied into the repository without local edits. This keeps
every update a plain file replacement and keeps the upstream diff readable.

### MkPFS

- Upstream: <https://github.com/PSBrew/MkPFS>
- Package: <https://pypi.org/project/mkpfs/>
- License: GPL-3.0
- Source of the current copy: the `mkpfs-1.1.0-py3-none-any.whl` wheel

The `backend/mkpfs/gui/` subpackage comes with the upstream package. The
application does not import it, because `backend/cli.py` only calls the
`python -m mkpfs` command line.

### UnRAR

- Upstream: <https://www.rarlab.com/rar_add.htm>
- Source of the current copy: `unrarsrc-7.2.7.tar.gz`
- License: the UnRAR license, included in the upstream tarball as `license.txt`

The rarlab file name does not match the product version. The tarball number is
a build counter, so always read `version.hpp` inside the archive:

| Tarball | `version.hpp` | Status |
| --- | --- | --- |
| `unrarsrc-7.2.6.tar.gz` | 7.21 | Previous vendored copy |
| `unrarsrc-7.2.7.tar.gz` | 7.23 | Current vendored copy, latest stable |
| `unrarsrc-7.3.2.tar.gz` | 7.30 beta 2 | Not used, beta |

A value other than `0` in `RARVER_BETA` means the source is a beta. The
download page lists the newest build, which can be a beta, so do not assume the
linked file is stable.

## Python packages

| Package | Needed by | Purpose |
| --- | --- | --- |
| customtkinter | Main script, `backend/gui.py` | User interface |
| Pillow | Main script | Images in the user interface |
| tkinterdnd2 | Main script | Drag and drop, optional |
| py7zr | Main script | `.7z` extraction, optional if a `7z` or `7za` command exists |
| rarfile | Main script | `.rar` extraction, needs an external tool, see below |
| cryptography | MkPFS | AES encryption in `backend/mkpfs/pfs.py` |
| zlib-ng | MkPFS | Preferred compression backend |
| pypresence | Nothing at this time | Listed in `RUN.bat`, not imported by the code |
| isal | MkPFS metadata | Optional backend, never selected by this application |

Install commands:

```bash
python -m pip install customtkinter pillow tkinterdnd2 py7zr rarfile cryptography zlib-ng
```

This list is enough to run the application from source because `backend/mkpfs`
is bundled. It was tested in a new virtual environment without `mkpfs`, `isal`
or `pypresence` installed. Installing the `mkpfs` package is only needed for the
fallback described in the next section.

## How MkPFS is located at runtime

`_locate_mkpfs()` in `backend/cli.py` tries these options in order and uses the
first one that applies:

1. Frozen build: internal bundle through `--mkpfs-internal`.
2. Bundled package: `backend/mkpfs/`, run as `python -m mkpfs` from `backend/`.
3. Sibling folder of the project that contains `mkpfs/__main__.py`.
4. An `mkpfs` command on `PATH`.
5. Automatic `pip install mkpfs==1.1.0`.

Option 2 applies to every normal source checkout, so a different `mkpfs` version
installed in the Python environment or on `PATH` has no effect on the
application. The startup log confirms the version in use:

```text
[INFO] Using bundled MkPFS package at .../backend
[INFO] MkPFS: MkPFS 1.1.0
```

A packaged executable starts the backend with a Python interpreter found on the
system (`get_backend_python_command()` in the main script). That interpreter
needs `cryptography` and `zlib-ng`. Without `zlib-ng`, MkPFS 1.1.0 falls back to
the standard library `zlib`.

## RAR extraction paths

The project has two independent ways to read `.rar` files.

| Caller | Library | Requirement |
| --- | --- | --- |
| Main script, `ArchiveExtractor._rar` | `rarfile` | An external tool such as `unrar` or `7z` |
| `backend/cli.py` given a `.rar` path | `backend/unrar` | The compiled `_unrar` extension |

`backend/unrar` is not compiled by the repository, and the compiled file is
ignored by `backend/.gitignore`. `backend/cli.py` runs with `backend/` as the
first entry of `sys.path`, so `import unrar` always finds this folder. A copy
installed with `pip install ./backend/unrar` is therefore shadowed, and without
a compiled extension the backend reports:

```text
[ERROR] RAR extraction failed: cannot import name '_unrar' from 'unrar'
```

Build the extension and place it next to the sources. Run this from the
repository root. It needs a C++ compiler and the Python development headers.

```bash
python -m pip install setuptools wheel
python -m pip wheel ./backend/unrar --no-deps -w build/unrar-wheel
python -m zipfile -e build/unrar-wheel/unrar-*.whl build/unrar-wheel/x
cp build/unrar-wheel/x/unrar/_unrar*.so backend/unrar/
```

The result is specific to one Python version and platform. Rebuild it after a
Python upgrade or a UnRAR update.

## What changed from MkPFS 0.0.9 to 1.1.0

The update spans the upstream 1.0.0 and 1.1.0 releases. The items below come
from comparing the two packages file by file.

### Compatibility with this application

| Area | Result |
| --- | --- |
| Flags passed by `backend/cli.py` | All still exist with the same meaning |
| Progress bar format matched by `PROGRESS_RE` | Unchanged |
| Backend log output | Identical on the test game, apart from the version number |
| `-V` version output | Unchanged format |

No change to application logic was necessary.

### Behavior changes

- **exFAT cluster size.** The automatic cluster size is now always 64 KiB. It
  used to be 32 KiB unless the average file size was at least 1 MiB. Upstream
  made this change because ShadowMountPlus needs 64 KiB allocation units for its
  LVD and BFS fast path. New images of games with many small files can therefore
  differ from older ones.
- **Compression backend.** A new `--compression-backend` flag accepts `auto`,
  `zlib-ng`, `zlib` and `isal`. The default `auto` tries zlib-ng, then the
  standard library `zlib`. This application does not pass the flag, so it uses
  zlib-ng when installed.
- **ISA-L warning.** ISA-L is never selected automatically. Upstream reports that
  it can produce PFSC blocks the PS5 hardware decompressor rejects, followed by a
  kernel panic shortly after launch ([PSBrew/MkPFS#132](https://github.com/PSBrew/MkPFS/issues/132)).
  Do not pass `--compression-backend isal` by hand unless you accept that risk.
- **Default compression level.** The upstream default went from 9 to 7. This has
  no effect here because the application always passes `--compression-level`,
  and its own default stays at 9.
- **File scanning.** A new `gather.py` module replaces `Path.rglob` when
  collecting files.

### New features the application does not use

- `mkpfs batch` for converting a whole directory.
- `--ampr-skip-regen-if-exists` and `--ampr-force-regen` on `pack folder`. The
  default stays the same: `ampr_emu.index` is regenerated when
  `fakelib/libSceAmpr.sprx` is present.
- `mkpfs tree` on a plain folder.
- The `mkpfs-gui` desktop interface in `backend/mkpfs/gui/`.

## What changed from UnRAR 7.21 to 7.23

- `dll.hpp` raises `RAR_DLL_VERSION` from 9 to 11 and adds `ERAR_USER_LIMIT`,
  `ROADOF_SHARED` and the `MaxDiskWrite`, `MaxAllocate` and `MaxRuntime` fields.
- These fields replace part of a reserved block in `RAROpenArchiveDataEx`, and
  `backend/unrar/_unrar.cpp` zero-initializes that structure, so the limits stay
  disabled and the wrapper needs no change.

## How to update MkPFS

1. Read the upstream release notes.

2. Download and unpack the wheel outside the repository:

   ```bash
   python -m pip download --no-deps mkpfs==X.Y.Z -d /tmp/mkpfs-new
   python -m zipfile -e /tmp/mkpfs-new/mkpfs-X.Y.Z-py3-none-any.whl /tmp/mkpfs-new/unpacked
   ```

3. Replace the bundled copy:

   ```bash
   rm -rf backend/mkpfs
   cp -r /tmp/mkpfs-new/unpacked/mkpfs backend/mkpfs
   ```

4. Update every pin:

   | File | Location |
   | --- | --- |
   | `PS5_FFPFSC_PRO_v1.3.0.py` | `MKPFS_VERSION` |
   | `backend/cli.py` | `pip install` call and the message after it in `_locate_mkpfs()` |
   | `RUN.bat` | `pip install` line |
   | `BUILD_EXE.bat` | `pip install` line |

5. Compare the command line and log output against the old version:

   - Flags used by `pack_folder_single_pass()` and `compress_file_to_ffpfsc()`.
   - `Progress.step()` in `backend/mkpfs/pbar.py` against `PROGRESS_RE`.
   - The strings matched in `CLIWorker._handle_line()`: `image created
     successfully`, `building exfat image from`, `running post-pack`,
     `running post-create check` and `build summary`.

6. Run the checks in [Verification results](#verification-results).

## How to update UnRAR

1. Download the candidate tarball and confirm that `RARVER_BETA` in `version.hpp`
   is `0`.

2. Copy only the files that are already vendored, so tarball project files stay
   out of the repository:

   ```bash
   for f in $(ls backend/unrar/src); do cp /tmp/unrar-new/unrar/"$f" backend/unrar/src/"$f"; done
   ```

3. Compare `dll.hpp` with the previous version and update
   `backend/unrar/_unrar.cpp` if the API changed. Compare the upstream `makefile`
   source list with `UNRAR_SOURCES` in `backend/unrar/setup.py`.

4. Rebuild the extension as described in
   [RAR extraction paths](#rar-extraction-paths).

5. Extract sample archives and compare the result with a system `unrar`.

## Verification results

Environment: Fedora 44, x86_64, Python 3.14.8, Tk 9.0.

The test source was a small synthetic game folder with `eboot.bin`,
`sce_sys/param.json`, compressible data, random data and
`fakelib/libSceAmpr.sprx`. Unless noted, each check ran from a clean checkout of
the branch in a new virtual environment that contained only the packages from
the install command above.

| Check | Result |
| --- | --- |
| `backend/cli.py` packs a folder with `--verify` | Exit code 0, 0 warnings, 0 errors |
| `backend/cli.py` packs an `.exfat` with block size and level set | Exit code 0 |
| `ampr_emu.index` generated and included in the image | Passed |
| `mkpfs unpack --deep` output compared with the source folder | Identical |
| Unpacked bytes, MkPFS 0.0.9 compared with 1.1.0 | Same, 9,205,625 |
| Image created with 0.0.9 read by 1.1.0 | Passed |
| Application parses the progress output into stages | Passed |
| Upstream 1.1.0 test suite run against `backend/mkpfs` | 505 passed, 1 skipped |
| `backend/tests` | 1 passed, 3 skipped (no `sample.rar` in the repository) |
| Compiled UnRAR 7.23 extracts three RAR3 and RAR5 samples | Same as system `unrar` |
| `backend/cli.py` extraction step on a `.rar` after the documented build | Passed |

## What has not been tested

- Mounting or playing a new image on a PS5.
- A real game, or any source larger than a few megabytes.
- Windows and macOS.
- `BUILD_EXE.bat`, `RUN.bat` and PyInstaller builds.
- Password protected archives.

Keep an older image of a game until the new one is confirmed on the console.
