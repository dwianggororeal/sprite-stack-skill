---
name: sprite-stack
description: Procedural workflow to build pseudo-3D pixel art objects via sprite stacking in code: (1) generate a physical .png slice sprite sheet with a Python script, then (2) test and showcase it interactively in HTML loading that .png directly. Trigger on /sprite-stack or whenever the user requests a sprite stack, fake 3D pixel art, or slice sheet.
---

# Sprite Stack (Codely PNG + HTML Testing Workflow)

Sprite stacking slices a 3D object into *N* horizontal layers (`z = 0` at the sole/ground up to `z = N - 1` at the crown) packed into a horizontal `.png` strip of size `(W * N) × H`. For **animated** sprites (*F* frames, or multiple animation states stacked vertically as `totalRows` rows), the sheet forms a `(W * N) × (H * totalRows)` grid where **columns = slices `z = 0..N-1`** and **rows = animation frames**. At render time, every slice is rotated by the same yaw angle and extruded upward along the camera projection so the object reads as crisp, solid 3D pixel art without blurry interpolation.

## Mandatory 2-Step Workflow

Whenever the user asks to create a sprite-stacked object or character:

### Step 1: Generate a Physical `.png` Sprite Sheet in Code (Python)
- **Mandatory**: produce an actual `.png` file on disk (e.g., `slices.png` or `<object>_slices.png`), **never** draw only in JS canvas memory.
- **Mandatory Default — Dense `1:1` Isotropic Voxel Slicing (`zScale = 1`)**: match vertical slice density `1:1` to the XY pixel grid (e.g., `N = 28` for a `24×24` chibi, `N = 80..90` for detailed `72×64` or `80×68` characters). This preserves smooth 3D silhouettes, accurate limb curvature, and consistent proportions from every camera angle.
- **Reference Landmark & Anatomical Proportion Measurement**: before coding coordinates from a reference image, measure normalized vertical and horizontal landmark ratios (`0.0..1.0` from sole to crown):
  - heights of ankle, knee, crotch, waist, chest, shoulder, chin, and crown,
  - head-to-body ratio, shoulder-to-hip width ratio, and limb joint coordinates,
  then convert them directly into voxel coordinates so anatomy remains faithful to the reference.
- Use Python (`PIL.Image` / Pillow + `numpy` for larger grids, or stdlib `zlib` + `struct` if Pillow is unavailable) to render each layer `z = 0..N-1` in `RGBA` mode with a transparent background `(0, 0, 0, 0)`.
- **High-Detail Sculpted Mode** (when the user asks for artistic detail rather than flat geometry):
  - **Sub-Part Anatomy & Silhouette Breakup**: sculpt distinct sub-muscles and armor plates (never plain tubes) plus 3D pointed cones/tufts for fur, manes, curved claws, weapon spikes, and tattered cloth strips so the 360° silhouette stays expressive.
  - **3D Surface Normal & Cavity AO Shader**: store a material ID grid (`mat_grid`), compute smoothed 3D surface gradient normals and `3×3×3` neighbor cavity ambient occlusion via NumPy, blend cool top-front key highlights with painterly warm shadow shifts, and quantize to crisp pixel-art color steps.
- **Multi-State Articulated Animation Mode** (when generating `idle`, `walk`, `run`, `jump`, `crouch`, etc.):
  - **Fixed-Length 3D Forward-Kinematics (FK) Skeleton**: define fixed bone lengths (`thigh`, `shin`, `foot`, `upper_arm`, `forearm`, `hand`, `spine`) and orient every limb and torso section via orthonormal local frames `(r_vec, f_vec, axis)` so leaning and twisting stay anatomically solid.
  - **Spherical Joint Bridges**: place spherical joint volumes at shoulders, elbows, and knees so `90°+` sprint, jump, and crouch bends never tear or gap.
  - **Automatic Ground-Contact Z-Pinning**: compute the lowest boot/foot sole Z from FK per frame and shift the root height so grounded poses sit squarely on `z = 0`, adding explicit `air_z > 0` lift only on airborne frames (`run` flight / `jump` arc).
  - **Multi-Row Sheet Layout**: stack all states consecutively by row (e.g., `idle` rows `0..5`, `walk` `6..11`, `run` `12..17`, `jump` `18..23`, `crouch` `24..29`).
- Run the generator script (`python generate_slices.py`) and verify the `.png` exists with expected dimensions and non-empty slices.

```python
# Minimal generate_slices.py skeleton (Pillow / 1:1 isotropic slicing)
from PIL import Image

W, H, N, F = 24, 24, 28, 4  # F=1 for static, F>1 for animation; zScale=1 (1:1)
sheet = Image.new("RGBA", (W * N, H * F), (0, 0, 0, 0))
px = sheet.load()

for f in range(F):
    for z in range(N):
        for y in range(H):
            for x in range(W):
                # Compute RGBA voxel color at (x, y, z, f)
                color = get_voxel_color(x, y, z, f, W, H, N)
                if color:
                    px[z * W + x, f * H + y] = color

sheet.save("slices.png")
```

### Pitfalls & Anti-Patterns (Avoid!)
- **Do not cut slice count (`N = 16..24`) and stretch `zScale > 1`** for detailed characters unless explicitly requested: stretching thick slices causes coarse vertical stair-stepping/terracing and degrades 3D silhouette quality.
- **Do not wrap outer 2D slice perimeters in a black outline**: in sprite stacking, outer 2D slice edges are the 3D side, front (`+Y`), and back (`-Y`) walls—outlining them in black hides eyes/faces from the front and blacks out the back. Keep any internal `part_id` seam lines strictly on interior pixels (`up > 0 && down > 0 && left > 0 && right > 0`).

### Step 2: Testing & Showcase in HTML (Load Real `.png` + Lossless Hybrid 3D Renderer)
- Create `index.html` in the same folder loading the generated `.png` file (`const img = new Image(); img.src = 'slices.png';`).
- **Coupled 3D Orthographic Projection (`Pitch` `-90°..+90°`)**: never squash slices independently with a fixed vertical lift, or the object will stretch/flatten awkwardly when tilted. Couple them via `pitch`:
  - `sinP = Math.sin(rad)` and `cosP = Math.cos(rad)`.
  - Pivot around the vertical midpoint `zMid = (N - 1) / 2` with vertical offset `(z - zMid) * space * zScale * cosP * zoom` (default `zScale = 1`).
  - When `pitch < 0` (viewing from underneath), reverse the painter's order loop (`z = N - 1` down to `0`).
- **Lossless Hybrid Renderer + Dirty-State Caching**:
  - **Dirty-state signature**: skip redrawing whenever `(curKey, curAnimKey, sheetRow, angle, pitch, space, zoom, fillGaps, panX, panY)` has not changed so HD sprites stay at 60 FPS.
  - **Fast Sub-Step Path (`|sinP| * zoom >= 1.25`)**: extrude each slice vertically across `steps = fillGaps ? Math.max(1, Math.ceil(sliceStepPx)) : 1` sub-steps (ordered by `pitch` sign) to eliminate horizontal scanline gaps.
  - **Low-Pitch Row-Blitting Path (`|sinP| * zoom < 1.25`)**: at shallow pitch (`0°..15°`) or low zoom, squashing slices directly causes nearest-neighbor row-skipping that drops front facial details. Rotate each slice un-squashed onto an offscreen `rotCanvas` at adaptive supersample `ss = W >= 48 ? 1 : 2`, then blit row-by-row back-to-front (`rMin..rMax`) using row height `voxelRowH = fillGaps ? (absSinP / ss + cosP * space * zScale) * zoom : Math.max(1, (absSinP / ss) * zoom)` — singularity-free at `0°`.
- Include the raw `.png` strip preview (`<img src="slices.png">`) and interactive controls (`Auto-spin`, `Fill Gaps`, `Angle`, `Pitch`, `Spacing`, `Zoom`, plus `Play Anim`, `FPS`, `Frame`, and state selector buttons when `anims` has multiple states like `Idle / Walk / Run / Jump / Crouch`).
- **Mouse Viewport Controls on `<canvas>`**: **Left-drag** to Rotate (`angle`) & Pitch (`pitch`), **Right-drag / Middle-drag / Shift+Left-drag** to Pan (`panX`, `panY` with `contextmenu` prevented), **Mouse Wheel** to Zoom (`wheel` with `preventDefault`), and **Double-click** / Reset button to restore camera defaults.
- **`file://` Compatibility**: never call `getImageData` on local images loaded in HTML so users opening `index.html` directly via `file://` never hit CORS tainted-canvas errors—stick to `ctx.drawImage(...)`.

```html
<script>
const rotCanvas = document.createElement('canvas');
const rctx = rotCanvas.getContext('2d');
let lastRenderSig = '';

// Inside requestAnimationFrame once cur.img.complete is true:
const sheetRow = curAnimRow + animFrame;
const sig = `${curKey}|${curAnimKey}|${sheetRow}|${angle.toFixed(4)}|${pitch.toFixed(2)}|${space}|${zoom}|${fillGaps}|${panX}|${panY}`;
if (sig !== lastRenderSig) {
  lastRenderSig = sig;
  const rad = pitch * Math.PI / 180, sinP = Math.sin(rad), cosP = Math.cos(rad), absSinP = Math.abs(sinP);
  const zMid = (cur.n - 1) / 2, zScale = cur.zScale || 1;
  const sliceStepPx = space * zScale * cosP * zoom;
  const zStart = pitch >= 0 ? 0 : cur.n - 1, zEnd = pitch >= 0 ? cur.n : -1, zStep = pitch >= 0 ? 1 : -1;

  ctx.setTransform(1, 0, 0, 1, 0, 0);
  ctx.imageSmoothingEnabled = false;
  ctx.clearRect(0, 0, view.width, view.height);

  if (absSinP * zoom >= 1.25) {
    const steps = fillGaps ? Math.max(1, Math.ceil(sliceStepPx)) : 1;
    const dyStart = pitch >= 0 ? 0 : steps - 1, dyEnd = pitch >= 0 ? steps : -1, dyStep = pitch >= 0 ? 1 : -1;
    for (let z = zStart; z !== zEnd; z += zStep) {
      const zOffset = (z - zMid) * sliceStepPx;
      for (let dy = dyStart; dy !== dyEnd; dy += dyStep) {
        const sy = cy - (zOffset + (dy - (steps - 1) / 2));
        ctx.setTransform(cos * zoom, sin * sinP * zoom, -sin * zoom, cos * sinP * zoom, cx, sy);
        ctx.drawImage(cur.img, z * cur.w, sheetRow * cur.h, cur.w, cur.h, -cur.w / 2, -cur.h / 2, cur.w, cur.h);
      }
    }
  } else {
    const ss = cur.w >= 48 ? 1 : 2;
    const rotSize = Math.ceil(Math.hypot(cur.w, cur.h)) * ss;
    if (rotCanvas.width !== rotSize || rotCanvas.height !== rotSize) {
      rotCanvas.width = rotCanvas.height = rotSize;
      rctx.imageSmoothingEnabled = false;
    }
    const rotHalf = rotSize / 2, rowStepPx = (sinP / ss) * zoom;
    const voxelRowH = fillGaps ? (absSinP / ss + cosP * space * zScale) * zoom : Math.max(1, (absSinP / ss) * zoom);
    const drawW = Math.round((rotSize / ss) * zoom), drawX = Math.round(cx - drawW / 2);
    const boundHalf = ((Math.abs(sin) * cur.w + Math.abs(cos) * cur.h) / 2) * ss;
    const rMin = Math.max(0, Math.floor(rotHalf - boundHalf - 1));
    const rMax = Math.min(rotSize - 1, Math.ceil(rotHalf + boundHalf + 1));

    for (let z = zStart; z !== zEnd; z += zStep) {
      rctx.setTransform(1, 0, 0, 1, 0, 0);
      rctx.clearRect(0, 0, rotSize, rotSize);
      rctx.setTransform(cos * ss, sin * ss, -sin * ss, cos * ss, rotHalf, rotHalf);
      rctx.drawImage(cur.img, z * cur.w, sheetRow * cur.h, cur.w, cur.h, -cur.w / 2, -cur.h / 2, cur.w, cur.h);
      const zBaseY = cy - (z - zMid) * sliceStepPx;
      for (let r = rMin; r <= rMax; r++) {
        const sy = zBaseY + (r + 0.5 - rotHalf) * rowStepPx;
        const y0 = Math.round(sy - voxelRowH / 2), y1 = Math.round(sy + voxelRowH / 2);
        ctx.drawImage(rotCanvas, 0, r, rotSize, 1, drawX, y0, drawW, Math.max(1, y1 - y0));
      }
    }
  }
}
</script>
```

---

## Appendix: Engine & 4-View Sheet Reference (Optional)

When sourcing slices from flat orthographic views (https://www.spritecook.ai/sprite-stack-generator) or exporting into Godot:

### 4-View Orthographic Prompt (Optional for AI Image Generators)
```
Orthographic model sheet of one small <OBJECT AND COLORS>, four views in a 2x2 grid. Top-left: TOP view looking straight down, object pointing DOWN. Top-right: SIDE view, object pointing LEFT. Bottom-left: FRONT view. Bottom-right: BACK view. Strictly orthographic, no perspective, no tilt, no three-quarter angle. Same scale in every view so length, width and height match between views. Crisp clean pixel art with a 1 px dark outline around each view, solid flat color fills, simple shading, no soft edges, no shadows, no ground, no text, no labels, no grid lines, generous empty space between views. Low-resolution chunky pixel art, each view about 32 pixels.
```

### Godot 4 (`sprite_stack.gd`)
```gdscript
extends Node2D
@export var sheet: Texture2D
@export var slices := 24
@export var turn := 0.0
@export_range(0.1, 1.0) var squash := 0.75
@export var spacing := 1.0

func _process(_d): queue_redraw()

func _draw():
	if sheet == null: return
	var w := float(sheet.get_width()) / slices
	var h := float(sheet.get_height())
	var rect := Rect2(-floorf(w / 2.0), -floorf(h / 2.0), w, h)
	var spin := Transform2D(turn, Vector2.ZERO)
	for z in slices:
		var lift := Transform2D(Vector2(1, 0), Vector2(0, squash), Vector2(0, -roundf(z * spacing)))
		draw_set_transform_matrix(lift * spin)
		draw_texture_rect_region(sheet, rect, Rect2(z * w, 0, w, h))
```
