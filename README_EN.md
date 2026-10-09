# SmartAxle · Intelligent Vehicle Wheel & Axle Detection System

[简体中文](README.md) | **English**

Camera (or local video) capture → motion detection trigger → wheel detection + multi-object tracking →
axle counting, single/dual tire and vehicle-type classification → save the full-vehicle side mosaic image,
write detection records, and push results to the host system in real time over serial / TCP / HTTP.

## Features

| Feature | Description |
| --- | --- |
| Offline local inference | Built-in OpenVINO (CPU/GPU) with automatic fallback to onnxruntime; consumes `models/*.onnx` directly — no internet required |
| Portable runtime | All data (config / logs / result images) lives next to the program; the whole folder can be copied around. Windows packaging scripts produce an installer-free portable build |
| Detection output | Axle count, single/dual tire sequence (vehicle-type code), wheelbase, full-vehicle side mosaic image |
| Integration | 5 output protocols: Serial protocol 1 / Serial protocol 2 / TCP protocol 1 / HTTP reporting / TCP server |
| Unattended operation | Tray-resident, auto-start on boot, single instance, automatic reconnection on stream loss, no-preview mode (saves CPU) |

---

## 1. Quick Start

### 1.1 Requirements

* Windows 10 / 11 (UI, tray, serial port and auto-start are verified on Windows).
* Python 3.8+ (this project is verified on 3.12).
* One video source: Hikvision IP camera (uses HCNetSDK when available) / generic RTSP camera / local video file.

### 1.2 Install & Run

```powershell
git clone <this repository>
cd axle_vision

# Option 1: one-click script (double-click; creates/reuses .venv and installs everything)
安装依赖.bat   # install-dependencies.bat

# Option 2: manual install
python -m venv .venv
.\.venv\Scripts\pip install -r requirements.txt

# Start the program
.\.venv\Scripts\python main.py
```

On first run the program extracts `models/` and `Icon/`, and generates `configs/config.ini` from built-in
defaults (existing config files are never overwritten).

### 1.3 First-time Setup (3 Steps)

| Step | What to do | Where |
| --- | --- | --- |
| 1 | Fill in camera **username / password / IP**, set Mode to `Camera` | Top-left "Network Camera Info" panel |
| 2 | Draw the detection zone (polygon over the lane), adjust **confidence** if needed | Main view + bottom parameter bar |
| 3 | Choose the **protocol**, fill in the **host IP:port** → click "Test Connection" | Bottom parameter bar |

Then click **Open Camera** to start detection; click **Stop Detection** to end the session (can restart).
No camera at hand? Set Mode to `Local Video` → "Select File" to run the full pipeline on a local video.

### 1.4 Data Directory (Important)

Config, logs and result images are all written to the **program's own directory**; the whole folder
can be copied/migrated as-is:

| Path | Content |
| --- | --- |
| `configs/config.ini` | All settings (includes camera credentials — **do not share or commit**, excluded by `.gitignore` by default) |
| `log/log.txt` | Application log (startup self-check, UI actions, camera status) |
| `log/worker.log` | Detection-thread log (detection, counting, mosaic, **destination of every forwarded message**) |
| `log/record.txt` | Detection records (time, axle count, wheel pattern, vehicle type, wheelbase) |
| `img/YYYYmmddHHMMSS.jpg` | Full-vehicle side mosaic image for each vehicle |

---

## 2. UI Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│ Network Camera Info   │      Main view (live preview / status)      │
│  Username Password IP │                                             │
│  ☑ Preview            ├─────────────────────────────────────────────┤
│  ☐ Auto-reconnect     │  Result table (time/axles/wheels/type/wb)   │
├────────────┬─────────┼─────────────────────────────────────────────┤
│ Mode        │ Direction│                                            │
│ ○ Camera    │ L→R      │  Result image (vehicle side mosaic)        │
│ ○ LocalVideo│ R→L     │                                             │
│            │ None     │                                             │
│ Controls    ├─────────┤                                             │
│ Open Camera │ ROI box │                                             │
│ Select File │ (polygon)│                                            │
│ Stop        │         │                                             │
├────────────┴─────────┴─────────────────────────────────────────────┤
│ Test(COM) │ Max/Min/Default axles │ Autostart Auto-run │ Protocol  │
│ Forward host / port / Test connection / Status          Sens. Conf. │
└──────────────────────────────────────────────────────────────────────┘
```

The program does **not** force a Qt theme — it follows the Windows system light/dark theme; all
custom-painted screens (waiting / no-preview status / result placeholder) use the dark industrial style
from `utils/splash.py`.

### 2.1 Capture Source & Direction

| Control | Meaning |
| --- | --- |
| Mode `Camera` / `Local Video` | Mutually exclusive: Camera enables "Open Camera" only; Local Video enables "Select File" only |
| Open Camera | Opens the stream by IP + credentials; uses Hikvision HCNetSDK when present, otherwise falls back to RTSP |
| Select File | Runs a local video for offline review (pipeline identical to camera mode) |
| Direction | Left→right / right→left / none. Counting adapts to the direction and decides mosaic slicing orientation |

### 2.2 ROI Box: Rectangle → Polygon

* Interaction: **drag vertices** to reshape, drag inside to **move the whole box**, **double-click an edge**
  to insert a vertex, **right-click a vertex** to delete (minimum 3 points).
* Semantics (important):
  * **The polygon only constrains motion detection** — pixels outside are blacked out before motion scoring
    ("which areas count");
  * **The AI detection region is the polygon's bounding rectangle** (model input must be rectangular), so the
    "top-left / bottom-right" inputs keep showing the bounding rectangle;
  * The axle-counting line is the horizontal center of the bounding rectangle.
* Vertices are saved in `detection/roi_points` (`x1,y1;x2,y2;...`); empty / invalid / fewer than 3 points
  **automatically falls back to the old rectangle behavior** — zero migration for old configs.

### 2.3 Bottom Parameter Bar

| Control | Meaning |
| --- | --- |
| COM + **Test** | Tests whether the selected serial port can be opened; result shown in a message box |
| Max / Min / Default axles | Result filtering and default classification |
| Auto-start / Auto-run | Auto-start writes a registry entry; Auto-run = start detection immediately on launch |
| Protocol | Serial 1 / Serial 2 / TCP protocol 1 / TCP client mode / TCP server mode |
| Sensitivity | Motion detection threshold (1000–5000, default 4000, lower = more sensitive) |
| **Confidence** | Wheel detection threshold (0.3–0.8, **recommended 0.50–0.55 on site**). Lower it to reduce missed dark/distant wheels (missing axles); too low causes false positives and over-counting. **Changes apply immediately while detection is running** |
| Forward address + Test | Host `IP:port` of the receiving system; **changes apply immediately while running** — no restart needed |

### 2.4 Preview / Auto-Reconnect

* **Uncheck "Preview"** = no-preview mode: the live view stops rendering and the main view shows a
  status image; CPU usage drops significantly — ideal for unattended operation. Detection, counting,
  forwarding and disk writes all continue normally.
* **"Auto-reconnect"**: when checked, the program automatically reconnects the camera after stalls or
  stream loss beyond a threshold (`camera/offtime`, default 50000 heartbeats).

### 2.5 Exit

* Closing the window = **hide to tray**; the program keeps running.
* Real exit: right-click the tray icon → **Exit**.
* While running, Task Manager shows **2 processes with the same name** (main process + AI detection
  child process); both should disappear after exit.

---

## 3. Recognition Pipeline

```
Capture thread (camera/video) --frames--> queue --> detection child process (infer+track+count+mosaic)
                                            |-- result queue --> UI table / result image
                                            |-- protocol push --> serial / TCP / HTTP
                                            '-- disk --> log/record.txt, img/*.jpg
```

| Stage | Implementation |
| --- | --- |
| Video input | Hikvision SDK (optional, needs HCNetSDK) / RTSP / local video file |
| Vehicle arrival | Frame-difference motion detection + sensitivity threshold + ROI polygon mask; recognized after consecutive motion frames |
| Wheel detection | **OpenVINO** inference (fallback to onnxruntime), ONNX models in `models/`; ROI crop → resize 320 → NMS → confidence filter |
| Tracking | SORT (Kalman prediction + IOU matching; alive tracks are output, no min_hits filtering) |
| Axle counting | Trigger line = vertical center of ROI; wheel centers crossing the line are counted left/right separately |
| Single/dual tires | Left/right wheels on the same axle are paired to build the single/dual sequence (e.g. "single dual dual") |
| Vehicle type | Single/dual sequence + wheelbase heuristics → vehicle-type code |
| Full mosaic | Slice width = **inter-frame vehicle displacement** (median of all visible wheel displacements, smoothed by a rolling median over the last 7 frames); frames are stitched horizontally into the side mosaic |
| Persist | `log/record.txt` + `img/YYYYmmddHHMMSS.jpg` |
| Push | 5 protocols (see section 5) |

> **Imaging and axle count share the same root**: a wheel that is not detected is neither counted nor
> included in the mosaic. "Mosaic missing an axle" and "axle count short by one" are the same problem —
> check the confidence threshold and ROI first.

### Vehicle-Type Codes

| Code | Wheel pattern | Code | Wheel pattern |
| --- | --- | --- | --- |
| 12 | S D | 125 | S D D D |
| 112 | S S D | 152 | S D D D |
| 115 | S S D D | 1222 | S D D D |
| 122 | S D D | 127 / 155 / 1522 | S D D D D |
| 15 | S D D (short wheelbase) | 11222 / 1125 | S S D D D |
| 157 | S D D D D D | 1127 / 1155 / 11522 | S S D D D D |

`S` = single tire, `D` = dual tires (one axle each).

---

## 4. Configuration (`configs/config.ini`)

| Section | Key | Meaning |
| --- | --- | --- |
| camera | usr / pwd / ip | Camera account and address |
| camera | com | Serial port number (protocols 0/1) |
| camera | direction | Driving direction: 0 left→right, 1 right→left |
| camera | start_up / auto_run | Start on boot / start detection on launch |
| camera | offtime | Idle/stalled heartbeats before auto-reconnect (0 = never) |
| camera | serial_protocol | Output protocol: 0 serial 1, 1 serial 2, 2 TCP, 3 HTTP, 4 TCP server |
| camera | forward_host / forward_port | Forwarding target address/port (read live by the child process) |
| camera | is_display | Show preview (0 = no-preview mode) |
| camera | is_rest | Auto-reconnect on stall/stream loss (UI label "Auto-reconnect") |
| camera | rtsp_url | Custom RTSP URL (empty = built from ip/usr/pwd) |
| detection | left_x / left_y / right_x / right_y | ROI bounding rectangle (1920×1080 reference) |
| detection | roi_points | Polygon ROI vertices `x1,y1;x2,y2;...`; empty/invalid falls back to rectangle |
| detection | max_num / min_num / defau_num | Axle count upper / lower / default value |
| detection | delay | Extra per-frame delay (throttling) |
| detection | fluency | Frame skipping: run inference every N frames, track-only otherwise |
| detection | vis_threshold | Wheel detection confidence threshold (default 0.8, **recommended 0.50–0.55**) |
| detection | sensitivity | Motion detection sensitivity |
| detection | backend | Inference backend: auto (default, OpenVINO first) / openvino / onnxruntime |
| detection | ov_device | OpenVINO device: CPU (default) / GPU / AUTO |

> Defaults are maintained in one place only: `utils/config.py::DEFAULT_CONFIG` (UI initial values and
> config backfill defaults are derived from it).

---

## 5. Output Protocols (`service/protocols.py`)

| Protocol | Channel | Frame |
| --- | --- | --- |
| 0 Serial protocol 1 | Serial | `A5 5A <axle count HEX> 5A A5` |
| 1 Serial protocol 2 | Serial | `65 6B 0<camera> 0<direction> 0<axles> [01\|02]... 0A 0D` |
| 2 TCP protocol 1 | TCP client | Hex string of the same frame as serial protocol 2 |
| 3 HTTP reporting | HTTP client | `POST /identify/axles`, JSON: `camera / camers_ip / kkinds / report_type / wheels` |
| 4 TCP server | Listening port | Pushes `A5 5A <axle count> 5A A5` once the host connects |

* **The axle-count field always carries the determined axle count** (not the wheel-pattern length). If the
  wheel pattern is one byte short (handheld video shake), the frame still reports the determined count and
  `log/worker.log` writes a WARNING for the possibly-short pattern.
* Frame length = 6 + wheel-pattern bytes; `0A 0D` is the terminator. HTTP mode uses a 3-second timeout, no retry.
* Byte-level specification, sample frames, integration steps, test scripts and acceptance cases:
  `docs/第三方系统对接文档.md` (Chinese).
* Troubleshooting: every send logs a `转发目标 host:port` (forward target) line in `log/worker.log`.

---

## 6. Project Layout

```
axle_vision/
├── main.py                 Entry point: main window, tray, capture/detection subprocess orchestration, protocol push
├── requirements.txt        Runtime dependencies
├── build.spec              PyInstaller build script (onedir)
├── 安装依赖.bat            One-click dependency install
├── 一键打包.bat            One-click portable build + installer
├── configs/config.ini      Config (auto-generated on first run; contains camera credentials, do not commit)
├── models/                 ONNX wheel detection models (model*.onnx, tyre_640.onnx)
├── Icon/                   Icons (custom drawn)
├── img/                    Result image output
├── log/                    log.txt / worker.log / record.txt
├── out/                    Self-check & diagnostics artifacts (screenshots, mosaics, probe CSV/logs)
├── ui/
│   ├── main_window.py      UI definition and widget bindings
│   └── video_display.py    Video display widget (scaled rendering + polygon ROI interaction + drawing)
├── service/
│   ├── camera.py           Capture threads: VideoFileCapture / RtspCapture / HikCapture
│   ├── hiksdk.py           Hikvision SDK wrapper (optional, falls back to RTSP)
│   ├── detector.py         ONNX inference + motion detection (polygon mask) + post-processing
│   ├── tracker.py          SORT multi-object tracking
│   ├── counter.py          Axle counting, tire pairing, vehicle-type classification
│   ├── worker.py           Detection child-process main loop (inference, mosaic, results, forwarding)
│   └── protocols.py        Serial / TCP / HTTP protocol push
├── utils/                  Qt compatibility, config I/O (incl. cross-process shared strings), logging,
│                           drawing, auto-start, runtime path resolution, dark status art
├── installer/
│   ├── SmartAxle.iss        Inno Setup installer script (AppVersion / AppId defined here)
│   └── version_info.txt     exe version resource (file/product version)
├── docs/                   Documentation (third-party integration spec, etc., in Chinese)
├── tools/                  md2docx.py (Markdown→Word), docx2pdf.ps1 (Word→PDF) and other helper scripts
└── tests/                  Self-checks, regression and on-site diagnostic scripts
```

---

## 7. Packaging (Optional)

For distribution, the scripts produce a portable folder and a Windows installer:

| Script (double-click) | Purpose |
| --- | --- |
| `安装依赖.bat` | Creates/reuses `.venv` and installs runtime deps + `pyinstaller` (versions locked to the verified environment: PySide6 binding, no PyQt5 / scipy) |
| `一键打包.bat` | Cleans old artifacts → PyInstaller portable folder → Inno Setup installer exe (about 3–6 minutes) |

Manual two-step build (Chinese filenames fail when piped):

```powershell
.\.venv\Scripts\python.exe -m PyInstaller build.spec --noconfirm --clean
& "$env:LOCALAPPDATA\Programs\Inno Setup 6\ISCC.exe" installer\SmartAxle.iss
```

* Without Inno Setup installed, the script skips the installer and outputs the portable folder only.
* **Why not onefile**: a onefile build extracts ~360 MB to `%TEMP%\_MEIxxxxxx` at every start and deletes
  it on exit; if any file is locked (detection subprocess, OpenVINO DLLs, antivirus scan) it fails with
  `Failed to remove temporary directory` and leaves residue. The folder build extracts nothing and writes
  no temp dirs — measured cold start to UI: **1.7 s**.
* The installer does **not** bundle `configs/config.ini` (camera credentials, ROI calibration, protocol
  addresses are site parameters): the program generates it from built-in defaults on first run, and
  upgrades never overwrite a tuned config.
* Packaging note: **close the running program before deleting `dist`** (locked files make the delete fail
  silently and PyInstaller fails afterwards).

---

## 8. Inference Backends (OpenVINO / onnxruntime)

`[detection] backend`: `auto` (default, OpenVINO first with automatic fallback) / `openvino` (forced,
still falls back to onnxruntime) / `onnxruntime` (forced).
`[detection] ov_device`: `CPU` (default) / `GPU` / `AUTO`; unavailable devices fall back to CPU.

Measured locally (`models/model.onnx`, 320×320 input, median of 200–300 runs):

| Threads | OpenVINO | onnxruntime |
| --- | --- | --- |
| 1 | 6.15 ms | 6.17 ms |
| 2 | 3.71 ms | 3.53 ms |
| 4 | **2.13 ms** | 3.01 ms |

* Both backends produce identical outputs (max absolute error ≈ 1e-6) with the same end-to-end wheel boxes.
  Reproduce: `python tests/compare_backends.py --threads 4`.
* On the same machine `ov_device=GPU` is slower than CPU for this small model (memory transfer overhead
  outweighs compute), so `CPU` is the default; try `GPU` / `AUTO` with larger inputs (e.g.
  `models/tyre_640.onnx`) or on weak CPUs.

---

## 9. Tests & Diagnostics

| Script | Purpose |
| --- | --- |
| `smoke_test.py` | Qt binding, all-module import, offscreen UI render (`out/ui_preview.png`), drawing & classification functions |
| `functional_test.py` | Generates a synthetic video → real capture thread + detection subprocess (model-accuracy independent) |
| `result_test.py` | Fake detector drives the full chain: tracking → counting → vehicle type → mosaic → persist → queue |
| `compare_backends.py` | Backend numerical consistency, per-frame performance, end-to-end detection comparison |
| `bench_infer.py` | Single-backend inference benchmark (`--backend` / `--threads` / `--device`) |
| `video_probe.py` | **Main field diagnostic**: feeds a video through the real capture→detection→mosaic→settlement pipeline, logging per-frame motion triggers, every wheel id & center x, mosaic width and results; outputs the mosaic and `out/vp_frames.csv`; supports `--vis` / `--sens` comparisons |
| `probe_config_mode.py` | Config persistence + Camera/Local Video exclusivity regression (auto-backup/restores the real `config.ini`) |
| `probe_roi_polygon.py` | Polygon ROI full-chain regression: parsing / dragging / adding / deleting points / mask / config round-trip |
| `probe_protocol_bytes.py` | Byte & frame assertions for the 5 protocols (axle count = determined count) |
| `probe_forward_live.py` | Forwarding end-to-end: real detection subprocess + peer service, verifies "address change applies while running" |
| `probe_share_host.py` | Cross-process shared-string regression |
| `probe_exit.py` | Whether child process/threads are cleaned up on tray exit |
| `probe_ui_reset.py` | UI reset, status text and late frames on stop / natural end |
| `probe_tracker.py` / `probe_counting.py` / `probe_ui.py` / `screenshot_ui.py` / `debug_capture.py` | Tracking, counting, GUI visibility, UI screenshots, capture-queue probes |

```powershell
.\.venv\Scripts\python.exe tests\smoke_test.py
.\.venv\Scripts\python.exe tests\video_probe.py --vis 0.5
```

> When investigating "missing / extra axles / mosaic" issues: **close the running program first**
> (the program writes `config.ini` in real time, so two probe runs would otherwise race on config).

---

## 10. Troubleshooting

| Symptom | Check first | Fix |
| --- | --- | --- |
| Results never reach the peer | `转发目标 host:port` (forward target) lines & errors in `log/worker.log` | Protocol type must match the peer; click "Test Connection"; address changes apply **without restart** (hot reload supported) |
| Axle count is one short | `log/worker.log` + mosaic | Usually a **missed wheel**: raise threshold 0.8 → 0.50–0.55; make sure the ROI covers the lane; below 0.45 over-counting was observed |
| Mosaic ghosting / gaps | `out/vp_frames.csv` (`video_probe`) | Mosaic width derives from inter-frame displacement; lane changes / camera shake distort it; a fixed camera mount helps |
| Vehicle-type code inconsistent with wheel pattern | Wheel-pattern length | Handheld/shaky video can drop one wheelbase character; the **axle count and frame are still correct**; fixed-camera footage pending retest |
| Open Camera fails | `log/log.txt` + UI message box | The box shows the IP and cause; verify credentials, IP reachability and open ports; RTSP is used without the SDK |
| Test button shows a box but doesn't change color | — | By design: serial test results are reported via a message box, the button keeps no color state |
| Window closed but program still runs | Tray icon | Close = hide to tray; right-click tray → "Exit" for a real exit |
| Processes remain after exit | Task Manager | 0 expected after normal exit; 2 same-name processes while running is normal (main + detection child) |
| `Failed to remove temporary directory` on exit | — | Legacy onefile issue, eliminated in the folder build; old `%TEMP%\_MEI*` residue can be cleaned |
| Code changes have no effect | Packaging | If the site runs the packaged build, it must be rebuilt with `一键打包.bat` |

---

## 11. Known Limitations

* The model is a wheel/axle detector; accuracy depends on lighting, camera angle and mounting height —
  the ROI must be calibrated per site.
* The mosaic may misalign on lane changes or camera shake — inherent to this class of algorithms.
* Main validation footage is handheld tracking video (960×544); fixed industrial camera tuning is pending.
* Serial / TCP / HTTP forwarding must be integrated with site equipment (protocols 0/1 need a hardware
  serial port).
* UI, auto-start and serial features are Windows-oriented and unverified on other platforms.

---

## 12. Contributing

Issues and pull requests are welcome:

1. Run `tests/smoke_test.py` before submitting; for pipeline changes please attach `video_probe.py`
   comparison results.
2. New config keys must be added to `utils/config.py::DEFAULT_CONFIG` and to section 4 of both READMEs.
3. Protocol changes must update `tests/probe_protocol_bytes.py` and the integration doc.

## 13. License

This project is released under the [MIT License](LICENSE). Third-party components (OpenVINO,
onnxruntime, PySide6, OpenCV, etc.) remain under their own licenses; the license of the model weights in
`models/` is subject to the note in the repository.
