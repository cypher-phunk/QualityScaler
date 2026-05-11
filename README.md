<div align="center">
    <img src="Assets/logo.png" width="175">
    <br><br>
    <b>QualityScaler</b> — AI image &amp; video upscaler for Linux
    <br><br>
    <img src="https://github.com/user-attachments/assets/98187c2e-0b10-4856-b34e-89a8df4fbfbe">
</div>

---

> **This is a Linux fork of [QualityScaler by Djdefrag](https://github.com/Djdefrag/QualityScaler).** The original is Windows-only. This fork ports it to Linux as a self-contained AppImage, replacing DirectML with CUDA (onnxruntime-gpu) for NVIDIA GPU acceleration. All credit for the original application goes to Djdefrag.

## What is QualityScaler?

QualityScaler uses AI models to upscale and de-noise photos and videos — up to 4× resolution increase with detail enhancement. Everything runs locally; no internet connection is required after download.

**Supported AI models:** BSRGANx2, BSRGANx4, RealESRGANx4, RealESR-Gx4, MSharpx4

---

## Downloading and Running

1. Download the latest `QualityScaler-*-x86_64.AppImage` from the [Releases](../../releases) page.
2. Make it executable and run:

```bash
chmod +x QualityScaler-*-x86_64.AppImage
./QualityScaler-*-x86_64.AppImage
```

That's it. The AppImage is fully self-contained — Python, CUDA libraries, ffmpeg, exiftool, and AI models are all bundled. No system packages or GPU drivers beyond your existing NVIDIA driver are required.

### System Requirements

| | Minimum |
|---|---|
| OS | Linux x86_64 |
| RAM | 8 GB |
| GPU | NVIDIA with ≥ 4 GB VRAM, driver 450+ |
| GPU Architecture | Pascal (GTX 10xx) or newer |
| FUSE | libfuse2 (see below if missing) |

> **GPU note:** The bundled CUDA libraries target CUDA 11 / cuDNN 8, which supports Pascal (GTX 1080, compute capability 6.1) and all newer NVIDIA architectures. AMD and Intel GPUs are not supported; the app will fall back to CPU, which is significantly slower.

> **CPU fallback:** If no compatible NVIDIA GPU is detected, inference runs on CPU automatically. Expect much longer processing times.

### FUSE not available?

On some systems (Docker, WSL, older kernels) FUSE may not be available. Use this flag instead:

```bash
./QualityScaler-*-x86_64.AppImage --appimage-extract-and-run
```

---

## Tips

- **Tile size:** Controls how much VRAM is used per inference pass. Lower it if you see out-of-memory errors. 200×200 is safe for a GTX 1080 with 8 GB VRAM; you can increase it on cards with more VRAM.
- **Input resize:** Downscaling the input before upscaling (e.g. 50%) reduces VRAM usage and speeds up processing at the cost of some quality.
- **Output path:** Defaults to the same folder as the input files. Click the output path field to change it.
- **Video resume:** If a video upscale is stopped mid-way, restarting with the same settings resumes from where it left off.

---

## Building from Source

### Prerequisites

- Linux x86_64
- Python 3.11 (`python3.11 --version`)
- `wget` and `curl`
- AI model files in `AI-onnx/` (see below)
- `appimagetool` in the repo root (see below)

**Install Python 3.11 if needed:**

```bash
# Fedora
sudo dnf install python3.11

# Ubuntu / Debian
sudo apt install python3.11 python3.11-venv python3.11-tk
```

**Download appimagetool** (one-time):

```bash
wget "https://github.com/AppImage/AppImageKit/releases/download/continuous/appimagetool-x86_64.AppImage"
chmod +x appimagetool-x86_64.AppImage
```

**Download AI models** and place all `.onnx` files in `AI-onnx/`:
- [AI models (gofile.io)](https://gofile.io/d/b4Ds9u)

### Build

```bash
git clone https://github.com/cypher-phunk/QualityScaler
cd QualityScaler

python3.11 -m venv .venv
source .venv/bin/activate

bash build-appimage.sh
```

This produces `QualityScaler-<version>-x86_64.AppImage` in the project root.

The build script handles everything automatically:
- Downloads a static `ffmpeg` binary into `Assets/` on first run
- Downloads a static `exiftool` binary into `Assets/` on first run
- Installs all Python dependencies including CUDA libraries
- Runs PyInstaller to bundle the app
- Copies CUDA libs into the AppImage so it's self-contained
- Packages everything with `appimagetool`

> **Python 3.11 is required.** Python 3.12+ has an f-string syntax change that breaks the build. Python 3.14 (Fedora 43 default) does not have onnxruntime-gpu 1.18.x wheels.

### Run without packaging (for development)

```bash
source .venv/bin/activate
python3.11 QualityScaler.py
```

---

## Releasing

Releases are built automatically by GitHub Actions. To publish a new release:

1. Update `VERSION` in `build-appimage.sh`
2. Tag and push:

```bash
git tag v2026.x
git push origin v2026.x
```

The workflow (`.github/workflows/release.yml`) builds the AppImage on `ubuntu-22.04` and attaches it to a GitHub Release.

---

## Contributing

Contributions are welcome. A few things worth knowing before you start:

### Architecture

QualityScaler is a single-file Python application (`QualityScaler.py`) with a customtkinter GUI. The AI inference runs via onnxruntime using `.onnx` model files.

### Key constraints

- **onnxruntime-gpu is pinned to 1.18.1.** This is the last version that uses cuDNN 8, which is required for Pascal GPUs (GTX 10xx, sm_61). Versions 1.19+ use cuDNN 9, which dropped Pascal support. Do not upgrade without testing on Pascal hardware or adding a GPU capability check.

- **CUDA libraries use the `cu11` suffix.** onnxruntime-gpu 1.18.1 links against CUDA 11 (`libcublasLt.so.11`, `libcudart.so.11.0`, etc.). The `nvidia-*-cu11` pip packages provide these and are bundled into the AppImage at build time.

- **Python 3.11 only for building.** See note above.

- **All subprocess calls must use `shell=False` (bool) and `stdin=subprocess_DEVNULL`.** The codebase previously had `shell="False"` (a non-empty string, which Python treats as truthy and therefore `shell=True`). This caused ffmpeg to receive no arguments. All subprocess calls have been corrected.

- **`iconbitmap()` does not work on Linux.** Tkinter's `iconbitmap()` only accepts `.xbm` format on Linux. The app uses `iconphoto()` with `logo.png` on non-Windows platforms.

- **File dialogs use native system dialogs where available.** The `_native_file_dialog()` function tries `zenity` (GNOME), then `kdialog` (KDE), then falls back to tkinter. Do not replace this with a plain `filedialog` call.

- **CUDA memory arena.** The ONNX session is configured with `arena_extend_strategy: "kSameAsRequested"` and a 6 GB cap. The default `kNextPowerOfTwo` strategy causes VRAM to double on each allocation, exhausting the GPU on multi-image batches. Do not remove these options.

### Running tests

There are no automated tests at this time. To validate a change, build the AppImage and test manually with a sample image and video.

### Commit style

Follow the existing commit messages: `QualityScaler <version>` for releases, descriptive messages for individual changes.

---

## How it works

QualityScaler is written entirely in Python, frontend to backend.

| Component | Role |
|---|---|
| [onnxruntime-gpu](https://github.com/microsoft/onnxruntime) | AI inference via CUDA (NVIDIA) |
| [customtkinter](https://github.com/TomSchimansky/CustomTkinter) | GUI |
| [OpenCV](https://github.com/opencv/opencv) | Image and video processing |
| [PyInstaller](https://github.com/pyinstaller/pyinstaller) | Binary bundling |
| [ffmpeg](https://ffmpeg.org/) | Video frame extraction and encoding (bundled static binary) |
| [exiftool](https://exiftool.org/) | Metadata preservation (bundled static binary) |

---

## Features

- Elegant and easy to use GUI
- Image and video upscaling
- Multiple GPU support
- Compatible images: jpg, png, tif, bmp, webp, heic
- Compatible video: mp4, webm, mkv, flv, gif, avi, mov, mpg, qt, 3gp
- Automatic image tiling to manage GPU VRAM
- Resize input/output independently before and after upscaling
- Blending/interpolation between original and upscaled output
- Video upscaling stop & resume
- Hardware accelerated video encoding (x264 fallback if codec unavailable)
- Fully offline — no telemetry, no internet required

---

## Examples

#### Video

![original](https://user-images.githubusercontent.com/32263112/209139620-bdd028f8-d5fc-40de-8f3d-6b80a14f8aab.gif)

https://user-images.githubusercontent.com/32263112/209139639-2b123b83-ac6e-4681-b94a-954ed0aea78c.mp4

#### Images

![test](https://user-images.githubusercontent.com/32263112/166690007-f1601487-7b94-4f2c-b4e2-436bc189a26e.png)

![ORIGINAL](https://user-images.githubusercontent.com/32263112/226847190-e4dbda21-8896-456d-8120-3137f3d2ac62.png)

![Bsrgan x4](https://user-images.githubusercontent.com/32263112/168884625-c869baee-4cca-4a33-bdad-b65d9c29889d.png)

---

## Credits

Original application by [Djdefrag](https://github.com/Djdefrag/QualityScaler).

AI models:
- [BSRGAN](https://github.com/cszn/BSRGAN)
- [Real-ESRGAN](https://github.com/xinntao/Real-ESRGAN)
- [MSharp](https://github.com/isl-org/MiDaS)
