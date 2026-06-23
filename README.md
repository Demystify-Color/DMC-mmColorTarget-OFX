# mmColorTarget — OpenFX (OFX) port

An OpenFX port of Marco Meyer's **mmColorTarget** Nuke gizmo, for use in
**DaVinci Resolve** (Studio) and probably also other OFX hosts (Natron, Fusion, Vegas, Nuke,…) – haven't tested this yet though.

ALSO THE WINDOWS INSTALLER ISN'T TESTED. PLEASE REPORT BACK IF IT DOESN'T WORK.

Check out my website (https://www.demystify-color.com) if you want to learn more about all things Colorgrading, Film Profiling & Color Science!
Or if you just like this port, please consider supporting me: https://buymeacoffee.com/nicofink ☕


It matches the colors of an image to a **MacBeth ColorChecker** reference by
sampling the 24 patches and solving a least-squares **3×3 color matrix**, exactly
as the original gizmo does:

```
X = lstsq(SRC, TGT)        # SRC, TGT are the Nx3 sampled patch values
out_rgb = in_rgb · X       # applied per pixel (== Nuke ColorMatrix M = transpose(X))
```

Original gizmo: https://www.nukepedia.com/gizmos/colour/mmcolortarget
Author's write-up: https://www.marcomeyer-vfx.de/posts/mmcolortarget-nuke-gizmo/

---

## What's different from the Nuke version (and why)

The Nuke gizmo has two image inputs and a *Calculate Matrix* button that bakes a
separate `ColorMatrix` node. OFX has no "create a node" button concept, and
Resolve effectively gives a plugin a single image input. So this port:

* **Solves and applies the matrix live, every frame** — no button, no baked node.
  Drag the corner-pin onto the chart and the correction appears immediately.
* **Defaults to "Use Reference Colorspace as Target"** (the gizmo's *ignoreTarget*
  mode) — the dominant Resolve workflow: neutralize a shot with a chart to known
  reference values. 38 reference colorspaces are embedded (sRGB, Rec.709, Rec.2020,
  ACES, ALEXA/RED/Sony gamuts, etc.), generated from the same colour-science.org
  data as the gizmo's `mmColorTarget_colorspaces.json`.
* Also accepts an **optional second "Target" clip** to match one image to another.
  Connect it via the node's OFX input (Color page: right-click node → *Add OFX
  Input*), then turn **Use Reference Colorspace as Target** off.

Everything else mirrors the gizmo: a 4-corner pin defining the 6×4 patch grid, a
sample size, per-patch enable/disable, and the *direct* / *no clip* sampling modes.

---

## Requirements

* **DaVinci Resolve Studio** (the free edition does not load OFX plugins), or any
  other OFX 1.4 host.
* To build: a C++14 compiler and a checkout of the OpenFX SDK
  (https://github.com/AcademySoftwareFoundation/openfx).

---

## Build

> There is **no prebuilt binary** in this package — OFX plugins are native code and
> must be compiled on (or for) the target OS. On macOS the steps below produce a
> single **universal** bundle that runs on both Apple Silicon and Intel.

### macOS (Apple Silicon + Intel, universal) — recommended

One-time: install Xcode Command Line Tools.

```bash
xcode-select --install
```

Then, from the `mmColorTarget` folder, just run the script. It clones the OpenFX
SDK automatically the first time and builds a universal binary:

```bash
./build_macos.sh            # build only
./build_macos.sh --install  # build, then copy to /Library/OFX/Plugins (asks for password)
```

Verify it's universal:

```bash
lipo -info mmColorTarget.ofx.bundle/Contents/MacOS/mmColorTarget.ofx
# -> Architectures in the fat file: ... are: x86_64 arm64
```

Prefer `make` directly? It builds universal by default on macOS:

```bash
git clone --depth 1 https://github.com/AcademySoftwareFoundation/openfx.git
make OFX_PATH=./openfx                       # universal (arm64 + x86_64)
make OFX_PATH=./openfx ARCHS="-arch arm64"   # single-arch, if you ever want it
sudo make install OFX_PATH=./openfx
```

If Resolve refuses to load it because the bundle isn't signed, clear the quarantine
attribute that macOS adds to downloaded files:

```bash
sudo xattr -dr com.apple.quarantine /Library/OFX/Plugins/mmColorTarget.ofx.bundle
```

(Optionally code-sign with your own identity:
`codesign --force --deep -s - /Library/OFX/Plugins/mmColorTarget.ofx.bundle`.)

### Linux

```bash
git clone --depth 1 https://github.com/AcademySoftwareFoundation/openfx.git
cd mmColorTarget
make OFX_PATH=../openfx
sudo make install OFX_PATH=../openfx
```

### Windows / any platform (CMake)

```bash
cmake -B build -DOFX_PATH=/path/to/openfx
cmake --build build --config Release
# build/mmColorTarget.ofx.bundle  ← copy to the OFX plugin folder (below)
```

The CMake build also targets macOS universal (`arm64;x86_64`, deployment 11.0) by
default.

---

## Install

Copy `mmColorTarget.ofx.bundle` to the host's OFX plugin directory:

| OS      | Path                                                  |
|---------|-------------------------------------------------------|
| Linux   | `/usr/OFX/Plugins`                                    |
| macOS   | `/Library/OFX/Plugins`                                |
| Windows | `C:\Program Files\Common Files\OFX\Plugins`           |

Then restart Resolve. The plugin appears in the **OpenFX** library under
**Color → mmColorTarget**. (In *Video & GPU* preferences you can *Auto Update*
the OFX list.)

---

## Usage

1. Apply **mmColorTarget** to the node/clip that contains the color chart.
2. In the viewer, drag the four **corner handles** so the magenta quad lines up
   with the chart and the 24 green sample boxes sit inside the 24 patches.
   (Top-Left handle → dark-skin patch corner; the grid is row-major from there.)
   Adjust **Sample Size** so each box stays well inside its patch.
3. Pick how to define the target:
   * **Use Reference Colorspace as Target** (default): choose a **Reference
     Colorspace**. The image is neutralized to those values.
   * Or turn it off and feed a **Target** image via *Add OFX Input* to match the
     source chart to the target chart (corner-pin the target too).
4. Optionally disable individual **Patches** (e.g. clipped or glared ones), switch
   **Sampling Method** to *no clip* to ignore 0/1-clipped pixels in the averages,
   and use **Mix** to dial the correction strength.

### Tips

* Works best on **linear / unprocessed** footage, like the original.
* The matrix is a pure 3×3 (no offset/white-balance-only constraint), so it both
  neutralizes and roughly matches exposure across channels.
* "Sample Size" and corner positions are in image pixels and are render-scale
  aware, so proxy/optimized playback samples the same patch areas.

---

## Files

```
mmColorTarget/
├── src/
│   ├── mmColorTarget.cpp     # the plugin (effect, processor, overlay, factory)
│   ├── mmct_core.h           # shared geometry + least-squares solver
│   └── colorchecker_data.h   # 38 embedded reference colorspaces (auto-generated)
├── test/
│   └── e2e_test.cpp          # end-to-end check: rotated chart → solve → neutralize
├── Makefile                  # Linux + macOS (universal) build
├── build_macos.sh            # one-command macOS universal build/install
├── CMakeLists.txt            # cross-platform build
├── Info.plist                # macOS bundle metadata
└── README.md
```

## Validation

The core was checked two ways:

* the 3×3 least-squares solver matches `numpy.linalg.lstsq` to ~1e-8, and
* `test/e2e_test.cpp` paints a **rotated** synthetic ColorChecker (each patch =
  reference run through a known camera matrix), then runs the plugin's own
  geometry + sampling + solve and confirms it recovers the matrix and neutralizes
  the chart back to the reference (errors ~1e-8). Build/run:

```bash
g++ -std=c++14 -O2 -o test/e2e_test test/e2e_test.cpp && ./test/e2e_test
```

## License

Plugin code: BSD-3-Clause (same as the OpenFX Support library it links).
ColorChecker reference data originates from colour-science.org, as shipped with
the original gizmo.
