# sprite-stack

A procedural skill for building pseudo-3D pixel art objects and articulated characters via **sprite stacking** directly in code (Python + HTML Canvas), with zero external generator dependencies.

## 2-Step Workflow

1. **Generate Physical `.png` Slice Sheets (Python)**
   - Uses Pillow / NumPy to render bottom-to-top horizontal slices (`z = 0..N-1`) into a transparent RGBA strip `(W * N) × H`, or a multi-row grid `(W * N) × (H * totalRows)` for animated sprites.
   - Defaults to dense **1:1 isotropic voxel slicing** (`zScale = 1`) and normalized reference landmark measurement (`0.0..1.0`) for accurate anatomy and proportions.
   - Supports **High-Detail Sculpted Mode** (sub-muscle sculpting, 3D silhouette breakup tufts/claws/cloth, and vectorized 3D surface-normal + `3×3×3` cavity-AO shading) and **Multi-State FK Animation** (fixed-bone 3D Forward Kinematics, spherical joint bridges, automatic ground-contact Z-pinning, and state rows like `idle`, `walk`, `run`, `jump`, `crouch`).

2. **Interactive `file://`-Safe HTML 3D Showcase**
   - Loads the physical `.png` directly via `ctx.drawImage(...)` (works safely from `file://` without CORS tainted-canvas errors).
   - **Coupled 3D Orthographic Pitch (`-90°..+90°`)**: pairs `sin(pitch)` & `cos(pitch)` centered on the object's vertical midpoint with full top/bottom painter's ordering.
   - **Lossless Hybrid 3D Renderer + Anti-Gap Extrusion**: uses dirty-state signature caching, fast vertical sub-steps at steep pitch, and adaptive pre-rotated back-to-front row-blitting at shallow pitch (`0°..15°`) so front details never drop out and scanline gaps never appear.
   - **Mouse Viewport Controls**: Left-drag to Rotate & Pitch, Right/Middle/Shift-drag to Pan, Mouse Wheel to Zoom, and Double-click to Reset.

## Installation & Usage

Clone into your global Antigravity skills folder:

```bash
git clone https://github.com/dwianggororeal/sprite-stack-skill.git ~/.gemini/config/skills/sprite-stack
```

Or on Windows PowerShell:

```powershell
git clone https://github.com/dwianggororeal/sprite-stack-skill.git "$env:USERPROFILE\.gemini\config\skills\sprite-stack"
```

Invoke with `/sprite-stack` in chat or ask to generate a sprite stack / fake 3D pixel art model from a prompt or reference image.
