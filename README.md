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
* I altered it to so you can set a set of source & target patches and it will
  calculate & show the 3x3 matrix then. Plus you can bake it as a DCTL that will
  automatically get saved into your LUT folder.

Everything else mirrors the gizmo: a 4-corner pin defining the 6×4 patch grid, a
sample size, per-patch enable/disable, and the *direct* / *no clip* sampling modes.


### Tips

* Works best on **linear / unprocessed** footage, like the original.
* The matrix is a pure 3×3 (no offset/white-balance-only constraint), so it both
  neutralizes and roughly matches exposure across channels.
* "Sample Size" and corner positions are in image pixels and are render-scale
  aware, so proxy/optimized playback samples the same patch areas.

## License

Plugin code: BSD-3-Clause (same as the OpenFX Support library it links).
ColorChecker reference data originates from colour-science.org, as shipped with
the original gizmo.
