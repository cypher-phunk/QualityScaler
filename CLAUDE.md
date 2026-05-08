# QualityScaler — Claude Context

Linux fork of [QualityScaler by Djdefrag](https://github.com/Djdefrag/QualityScaler).
Original is Windows-only. This fork ports it to Linux and packages it as an AppImage,
replacing DirectML with CUDA (onnxruntime-gpu) for NVIDIA GPU acceleration.

---

## Build the AppImage

```bash
python3.11 -m venv .venv --clear
source .venv/bin/activate
bash build-appimage.sh
./QualityScaler-2026.3-x86_64.AppImage
```

**Python must be 3.11.** 3.12+ breaks the f-string backslash on line 2377.
3.14 (system default on Fedora 43) doesn't have onnxruntime-gpu 1.18.x wheels at all.

`build-appimage.sh` handles everything automatically:
- Downloads static `ffmpeg` and `exiftool` binaries into `Assets/` on first run
- Installs Python deps from `requirements.txt`
- Runs PyInstaller via `QualityScaler.spec`
- Copies CUDA libs from nvidia pip packages into `AppDir/usr/lib/cuda/`
- Packages with `appimagetool-x86_64.AppImage` (must exist in repo root)

`appimagetool` must be present manually — download once:
```bash
wget "https://github.com/AppImage/AppImageKit/releases/download/continuous/appimagetool-x86_64.AppImage"
chmod +x appimagetool-x86_64.AppImage
```

AI model `.onnx` files must be in `AI-onnx/` before building (they're gitignored due to size).
Download link is in `README.md`.

---

## Releasing to GitHub

Push a tag — GitHub Actions does the rest:
```bash
git tag v2026.x
git push origin v2026.x
```

Workflow: `.github/workflows/release.yml` — builds on `ubuntu-22.04`, attaches the AppImage
to a GitHub Release. Users download, `chmod +x`, run. No installs needed.

---

## Why things are the way they are

### onnxruntime-gpu pinned to 1.18.1 + cu11 packages
The dev machine has a GTX 1080 (Pascal, compute capability sm_61).
cuDNN 9 (used by onnxruntime-gpu 1.19+) dropped Pascal support — sm_70 minimum.
1.18.1 is the last release using cuDNN 8, which still supports sm_61.
It links against CUDA 11 (`libcublasLt.so.11`, `libcudart.so.11.0`, etc.),
so all nvidia packages use the `cu11` suffix.

### CUDA libs bundled in the AppImage
`build-appimage.sh` copies `*.so*` from every `nvidia/*/lib/` dir in the venv's
site-packages into `AppDir/usr/lib/cuda/`. `AppRun` prepends that dir to
`LD_LIBRARY_PATH` so onnxruntime finds them without any host install.
Required packages in `requirements.txt`:
- `nvidia-cuda-runtime-cu11`
- `nvidia-cudnn-cu11`
- `nvidia-cublas-cu11`
- `nvidia-cufft-cu11`
- `nvidia-curand-cu11`

### CUDA memory arena
`arena_extend_strategy: "kSameAsRequested"` is set in `_load_inferenceSession()`.
Default `kNextPowerOfTwo` causes VRAM to double on each allocation, exhausting
the 8GB on multi-image batches. Also capped at 6GB and using `HEURISTIC` conv search.
If OOM still occurs, reduce tile size in the UI (try 200×200).

### PIL.ImageTk
System Pillow on Fedora is headless (no Tk support). Fix: build inside the `.venv`
where the PyPI Pillow wheel includes ImageTk. `PIL._tkinter_finder` and `PIL.ImageTk`
are in `hiddenimports` in `QualityScaler.spec`.

### iconphoto instead of iconbitmap
Tkinter's `iconbitmap()` only accepts `.xbm` on Linux, not `.ico`.
`QualityScaler.py` uses `iconphoto()` with `logo.png` on non-Windows.

### Native file dialogs
`_native_file_dialog()` in `QualityScaler.py` tries: zenity → kdialog → tkinter fallback.
On Fedora GNOME, zenity is pre-installed and gives the native GTK file chooser.

### ffmpeg and exiftool
Both bundled as static binaries in `Assets/`. `build-appimage.sh` downloads them
automatically if not present. The code uses `find_by_relative_path("Assets/ffmpeg")`
on Linux (same pattern as the existing Windows `.exe` paths).
exiftool version is fetched dynamically from `https://exiftool.org/ver.txt`.

### subprocess calls
All subprocess calls use `shell=False` (bool, not string) and `stdin=subprocess_DEVNULL`.
The original code had `shell="False"` (a truthy string = shell=True), which caused
ffmpeg to receive no arguments and put the terminal in a broken state showing `~` characters.
