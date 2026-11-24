# sprite-stack

Skill prosedural untuk membangun objek pixel art pseudo-3D dengan teknik **sprite stacking** secara *codely* (Python + HTML Canvas), tanpa bergantung pada generator eksternal.

## Alur Kerja 2 Langkah

1. **Generate Sprite Sheet `.png` Fisik (Python)**
   - Menggunakan Pillow / NumPy untuk merender irisan horizontal *bottom-to-top* (`z = 0..N-1`) ke dalam strip RGBA transparan `(W * N) × H`, atau grid `(W * N) × (H * F)` untuk sprite beranimasi (*F* frame).
   - Default memakai irisan isotropik padat **1:1** (`zScale = 1`) dan pengukuran rasio landmark referensi agar proporsi anatomi akurat.
   - Mendukung **High-Detail Sculpted Mode**: pemodelan sub-otot + *silhouette breakup* (cakar, bulu/surai runcing, pita, duri) + *3D Surface Normal & Cavity AO Shader*.

2. **Testing & Showcase Interaktif di HTML (`file://`-safe)**
   - Memuat file `.png` fisik secara langsung via `ctx.drawImage(...)` (aman dibuka langsung lewat `file://` tanpa error CORS `tainted canvas`).
   - **Proyeksi Ortografis 3D (`Pitch` `-90°..+90°`)**: pasangan `sin(pitch)` & `cos(pitch)` terpusat di tengah tinggi objek dengan *painter's order* penuh dari atas maupun bawah.
   - **Lossless Hybrid Renderer + Anti-Gap Extrusion**: sub-step vertikal saat pitch tinggi dan *pre-rotated back-to-front row-blitting* saat pitch rendah (`0°..15°`) agar tidak ada baris wajah yang hilang maupun celah *scanline*, lengkap dengan *dirty-state caching*.
   - **Kontrol Mouse**: Left-drag untuk Rotate & Pitch, Right/Middle/Shift-drag untuk Pan, Mouse Wheel untuk Zoom, Double-click untuk Reset.

## Instalasi & Pemakaian

Salin atau clone folder ini ke direktori global skill Antigravity:

```bash
git clone https://github.com/dwianggororeal/sprite-stack-skill.git ~/.gemini/config/skills/sprite-stack
```

Atau di Windows PowerShell:

```powershell
git clone https://github.com/dwianggororeal/sprite-stack-skill.git "$env:USERPROFILE\.gemini\config\skills\sprite-stack"
```

Panggil di chat dengan `/sprite-stack` atau minta pembuatan *sprite stack* / *fake 3D pixel art* dari ide maupun gambar referensi.
