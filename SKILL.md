---
name: sprite-stack
description: Workflow prosedural untuk membuat objek pixel art pseudo-3D dengan teknik sprite stacking secara codely: (1) generate file fisik .png sprite sheet slice dengan script Python, lalu (2) testing dan showcase interaktif di HTML yang me-load file .png tersebut. Aktifkan pada perintah /sprite-stack atau saat user minta buat sprite stack, fake 3D pixel art, atau slice sheet.
---

# Sprite Stack (Codely PNG + HTML Testing Workflow)

Sprite stacking memecah objek 3D menjadi *N* irisan horizontal (`slice`, dari bawah `z = 0` ke atas `z = N - 1`) dalam satu strip gambar horizontal `.png` berukuran `(W * N) × H`. Untuk sprite yang **dianimasikan** (*F* frame), layout sheet menjadi grid `(W * N) × (H * F)` dengan **kolom = slice `z = 0..N-1`** dan **baris = frame animasi `f = 0..F-1`**. Saat dirender, setiap slice diputar dengan sudut yang sama lalu diekstrusi ke atas sehingga tampak seperti objek 3D utuh yang tajam tanpa interpolasi blur.

## Workflow Wajib (2 Langkah)

Setiap kali user meminta pembuatan objek sprite stack:

### Langkah 1: Generate Sprite Sheet `.png` Fisik Secara Codely (Python)
- **Wajib** menghasilkan file `.png` nyata di disk (mis. `slices.png` atau `<nama_objek>_slices.png`), **bukan** sekadar menggambar di memori canvas JS.
- **Default Wajib — Dense `1:1` Isotropic Voxel Slicing (`zScale = 1`)**: gunakan kepadatan vertikal `1:1` antara tinggi slice dan ukuran piksel XY (mis. `N = 28` untuk chibi `24×24`, `N = 80..90` untuk karakter detail `72×64` atau `80×68`). Ini menghasilkan siluet 3D yang halus, detail kurva akurat, dan proporsi yang konsisten di semua sudut kamera.
- **Wajib Ukur Landmark & Proporsi Referensi (Jika Ada Gambar Referensi)**: sebelum menulis koordinat voxel, ukur terlebih dahulu rasio ter-normalisasi (`0.0..1.0` dari telapak kaki ke puncak kepala) untuk:
  - tinggi vertikal tiap landmark (*ankle*, *knee*, *crotch*, *waist*, *chest*, *shoulder*, *chin*, *crown*),
  - rasio kepala-terhadap-tubuh, lebar bahu vs lebar pinggul, serta posisi sendi lengan/kaki/senjata,
  lalu konversikan langsung ke koordinat voxel agar anatomi dan postur tetap setia pada referensi.
- Gunakan Python (`PIL.Image` / Pillow + `numpy` untuk grid besar, atau `zlib` + `struct` stdlib jika Pillow tidak tersedia) untuk menggambar setiap layer `z = 0..N-1`, mode `RGBA` dengan background transparan `(0, 0, 0, 0)`.
- **High-Detail Sculpted Mode** (jika user meminta detail tinggi / artistik, bukan sekadar memperbesar resolusi):
  - **Anatomi Sub-Part & Silhouette Breakup**: bangun sub-otot terpisah (bukan hanya silinder/tabung mulus) ditambah *kerucut/tuft* 3D runcing untuk surai/bulu, cakar melengkung, duri senjata, dan helai kain robek agar siluet tetap kaya saat diputar 360°.
  - **3D Surface Normal & Cavity AO Shader**: simpan grid `mat_grid` (ID material), hitung normal gradien 3D & kepadatan tetangga `3×3×3` (*cavity ambient occlusion*) lewat NumPy, lalu padukan cahaya atas-depan yang sejuk (*cool highlight*) dengan bayangan hangat (*warm shadow shift*) dan kuantisasi langkah warna piksel yang tajam.
- Jalankan script generator (`python generate_slices.py`) dan pastikan file `.png` benar-benar terbentuk di folder target.

```python
# Contoh kerangka generate_slices.py (Pillow / NumPy 1:1 isotropic slicing)
from PIL import Image

W, H, N, F = 24, 24, 28, 4  # F=1 untuk statis, F>1 untuk animasi; zScale=1 (1:1)
sheet = Image.new("RGBA", (W * N, H * F), (0, 0, 0, 0))
px = sheet.load()

for f in range(F):
    for z in range(N):
        for y in range(H):
            for x in range(W):
                # Tentukan warna RGBA voxel pada (x, y, z, f)
                color = get_voxel_color(x, y, z, f, W, H, N)
                if color:
                    px[z * W + x, f * H + y] = color

sheet.save("slices.png")
```

### Pitfalls & Anti-Patterns (Hindari!)
- **Jangan mengurangi jumlah slice (`N = 16..24`) lalu meregangkannya dengan `zScale > 1`** untuk karakter detail: memaksa ketinggian besar dengan sedikit irisan menyebabkan efek tangga vertikal kasar (*stair-stepping / terracing*) dan menurunkan kualitas siluet 3D secara drastis. Hanya gunakan *few-slices* + `zScale > 1` bila user **secara eksplisit** memintanya.
- **Jangan menaruh pixel outline gelap menutupi dinding depan (`+Y`) atau belakang (`-Y`) slice**: pada *sprite stack*, tepi luar 2D tiap slice adalah dinding samping objek 3D; menimpanya dengan garis hitam di sekeliling luar akan menutupi mata/wajah dari depan dan menghitamkan punggung dari belakang. Jika memakai garis *seam* internal antar-komponen (`part_id`), batasi hanya pada piksel interior murni (`up > 0 && down > 0 && left > 0 && right > 0`).

### Langkah 2: Testing & Showcase di HTML (Load `.png` Asli + Lossless Hybrid 3D Renderer)
- Buat `index.html` di folder yang sama yang memuat file `.png` fisik hasil Langkah 1 (`const img = new Image(); img.src = 'slices.png';`).
- **Wajib Proyeksi 3D Ortografis Berpasangan (`Pitch` `-90°..+90°`)**: jangan menggeser `squash` secara terpisah dengan tinggi `Z` konstan, karena objek akan tampak melar/gepeng saat kamera dimiringkan ke atas/bawah. Gunakan sudut `pitch`:
  - `sinP = Math.sin(rad)` dan `cosP = Math.cos(rad)`.
  - Pusatkan pivot di tengah tinggi objek: `zMid = (N - 1) / 2`, dengan offset vertikal `(z - zMid) * space * zScale * cosP * zoom` (default `zScale = 1`).
  - Jika `pitch < 0` (melihat dari bawah), balik urutan *painter's order* loop `z` (dari `N - 1` turun ke `0`).
- **Wajib Lossless Hybrid Renderer + Dirty-State Caching**:
  - **Dirty-state signature**: lewatkan redraw apabila `(curKey, animFrame, angle, pitch, space, zoom, fillGaps, panX, panY)` tidak berubah agar resolusi HD tetap mulus 60 FPS.
  - **Fast Sub-Step Path (`|sinP| * zoom >= 1.25`)**: lakukan sub-step vertikal `steps = fillGaps ? Math.max(1, Math.ceil(sliceStepPx)) : 1` dengan arah `dy` mengikuti tanda `pitch` agar tidak ada celah *scanline*.
  - **Low-Pitch Row-Blitting Path (`|sinP| * zoom < 1.25`)**: pada *pitch* rendah (`0°..15°`) atau zoom kecil, transformasi *squash* langsung akan menjatuhkan baris piksel depan (*row-skipping*). Putar slice ke buffer offscreen `rotCanvas` beresolusi adaptif `ss = W >= 48 ? 1 : 2` tanpa *squash*, lalu blit baris demi baris dari belakang ke depan (`rMin..rMax`) dengan tinggi baris `voxelRowH = fillGaps ? (absSinP / ss + cosP * space * zScale) * zoom : Math.max(1, (absSinP / ss) * zoom)` — bebas singularitas di `0°`.
- Tampilkan juga preview strip `.png` asli (`<img src="slices.png">`) dan kontrol interaktif (`Auto-spin`, `Fill Gaps`, `Angle`, `Pitch`, `Spacing`, `Zoom`, serta kontrol animasi `Play`, `FPS`, dan `Frame` bila `F > 1`).
- **Interaksi Mouse di Preview Canvas**: dukung manipulasi kamera langsung di `<canvas>`—**Left-drag** untuk Rotate (`angle`) & Pitch (`pitch`), **Right-drag / Middle-drag / Shift+Left-drag** untuk Pan (`panX`, `panY` dengan `contextmenu` dicegah), **Mouse Wheel** untuk Zoom (`wheel` dengan `preventDefault`), serta **Double-click** / tombol reset untuk mengembalikan posisi kamera ke default.
- Catatan penting: karena `index.html` kadang dibuka langsung lewat `file://`, jangan gunakan `getImageData` pada gambar lokal agar tidak terkena blokir CORS `tainted canvas`—cukup gunakan `ctx.drawImage(...)` yang 100% aman di `file://`.

```html
<script>
const rotCanvas = document.createElement('canvas');
const rctx = rotCanvas.getContext('2d');
let lastRenderSig = '';

// Di dalam loop requestAnimationFrame setelah cur.img.complete:
const sig = `${curKey}|${animFrame}|${angle.toFixed(4)}|${pitch.toFixed(2)}|${space}|${zoom}|${fillGaps}|${panX}|${panY}`;
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
        ctx.drawImage(cur.img, z * cur.w, animFrame * cur.h, cur.w, cur.h, -cur.w / 2, -cur.h / 2, cur.w, cur.h);
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
      rctx.drawImage(cur.img, z * cur.w, animFrame * cur.h, cur.w, cur.h, -cur.w / 2, -cur.h / 2, cur.w, cur.h);
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

## Lampiran: Referensi Engine & AI View Sheet (Opsional)

Jika gambar berasal dari 4 tampilan datar (top, side, front, back) di https://www.spritecook.ai/sprite-stack-generator atau ingin dipasang di Godot / Roblox:

### Prompt 4 Tampilan (Opsional jika pakai AI Image Generator)
```
Orthographic model sheet of one small <OBJEK DAN WARNA>, four views in a 2x2 grid. Top-left: TOP view looking straight down, object pointing DOWN. Top-right: SIDE view, object pointing LEFT. Bottom-left: FRONT view. Bottom-right: BACK view. Strictly orthographic, no perspective, no tilt, no three-quarter angle. Same scale in every view so length, width and height match between views. Crisp clean pixel art with a 1 px dark outline around each view, solid flat color fills, simple shading, no soft edges, no shadows, no ground, no text, no labels, no grid lines, generous empty space between views. Low-resolution chunky pixel art, each view about 32 pixels.
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
