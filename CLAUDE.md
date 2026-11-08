# CLAUDE.md

MagiskOnWSALocal builds a patched Windows Subsystem for Android (WSA) package on Linux: it downloads WSA from the Microsoft Store, injects Magisk (root) and optionally GApps into the Android system images, and outputs a folder the user installs on Windows with `Run.bat` / `Install.ps1`. Licensed AGPL-3.0, derived from LSPosed's project.

## Commands

There is no compile step, test suite, or CI workflow. "Building" means running the pipeline script.

```bash
# Interactive (TUI via whiptail/dialog; installs deps on apt-get/zypper distros)
./scripts/run.sh

# Non-interactive CLI
./scripts/build.sh --help
./scripts/build.sh --arch x64 --release-type retail --magisk-ver stable \
                   --gapps-brand MindTheGapps --root-sol magisk --remove-amazon
./scripts/build.sh --offline --gapps-variant pico --magisk-custom   # uses files already in download/
```

- Needs root/sudo (loop-mounts ext4 images), x86_64 or aarch64 host, Python ≥ 3.7 with `requests`.
- System deps: `setools lzip wine patchelf e2fsprogs aria2 python3 attr unzip`, plus `winetricks msxml6`.
- Downloads go to `download/`, results to `output/` (both gitignored). Delete `download/` to force fresh downloads.
- A full build downloads several GB and mounts filesystems, so it usually **can't run in a sandbox/cloud session**. Validate changes statically instead:

```bash
bash -n scripts/*.sh                    # syntax check shell scripts
shellcheck scripts/*.sh                 # lint (not preinstalled; existing code uses `# shellcheck disable=` directives)
python3 -m py_compile scripts/*.py      # syntax check Python helpers
./scripts/build.sh --help               # arg parsing smoke test (no root needed)
```

## Layout

- `scripts/build.sh`: the whole pipeline: arg parsing → generate download links → aria2c download → extract WSA/Magisk/GApps → expand & mount `system/vendor/product/system_ext.img` → integrate Magisk, GApps and extra packages → merge language resources (`wine64 makepri.exe`) → unmount & shrink → write `Install.ps1` / `Run.bat` (heredocs inside build.sh) → copy to `output/`.
- `scripts/run.sh`: interactive wrapper; installs deps, asks questions, then calls `build.sh` with flags.
- `scripts/generate*Link(s).py`: append aria2 entries to `download/download.list`. `generateWSALinks.py` talks to the Windows Update SOAP API using the templates in `xml/`.
- `scripts/extract*.py`, `fixGappsProp.py`: unpack archives and patch props. `extractWSA.py` passes version info back to bash by writing `KEY=value` lines to the file in `$WSA_WORK_ENV`, which build.sh then `source`s.
- `x64/`, `arm64/`: per-arch files copied into the image (`system/` = extra apps like VpnDialogs, `gapps/` = MindTheGapps overlays).
- `linker/<host-arch>/`: Android bionic linker/libs so `magiskpolicy` can run on the Linux host (via `patchelf`).
- `wine/<host-arch>/makepri.exe`: Windows tool run under wine to rebuild `resources.pri`.

## Code style & conventions

- **Bash**: scripts start with the AGPL header and require bash (not sh). 4-space indent. Quote all expansions and guard deletes with `${VAR:?}`. Append `|| abort` to every step that can fail, because `abort` unmounts and cleans up. Use `$SUDO` (empty when already root) instead of a literal `sudo`. Add new CLI options in all four places: `ARGUMENT_LIST`, the `case` parser, `usage()`, and a `*_MAP` array + `check_list` if the option takes an enum value. Mirror the option in `run.sh`'s prompts.
- **Python**: small standalone scripts that take positional `sys.argv` args (no argparse) and use `pathlib.Path`. Keep them compatible with Python 3.7. Only third-party dependency is `requests`. Print with `flush=True` so output interleaves correctly with bash.
- **Line endings** (`.gitattributes`): `*.sh` → LF, `*.xml` → CRLF. Don't normalize these. Binaries (`*.so`, `*.exe`, `*.apk`) must stay untouched.
- Don't commit anything from `download/` or `output/`.
- Commit messages are short and imperative (e.g. "Add issue template", "Update bug_report.yml").

## Gotchas

- Magisk < 24 does not work on WSA. Aroma OpenGApps is unsupported on x86_64.
- `--magisk-custom` expects `download/magisk-debug.zip`.
- `--gapps-variant` only matters for OpenGApps. MindTheGapps has a single variant.
- build.sh computes each image's new size from the payload it adds (the "Expand images" block) and adds 20000 sectors of headroom. If you add new files to an image, update those calculations too.
- `py_compile` on Python 3.12+ prints `SyntaxWarning: invalid escape sequence '\.'` for regexes in `generateGappsLink.py` / `generateWSALinks.py` (they're non-raw f-strings). This is a known issue and does not break anything.
