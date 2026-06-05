# 🚀 Solusi Layar Bawah Tidak Bisa Diklik (Dead Zone / Phantom Window) di Ubuntu/GNOME
# 🚀 Fix for Unclickable Bottom Screen Area (Dead Zone / Phantom Window) on Ubuntu/GNOME

Repositori ini berisi panduan teknis dan patch untuk mengatasi masalah **area bawah layar tidak bisa diklik** saat menggunakan ekstensi **Dash2Dock Animated / Dash2Dock Lite** di GNOME Shell, terutama pada sesi **Wayland** di Ubuntu.

This repository contains a technical guide and patch to fix the **unclickable bottom screen area** issue when using the **Dash2Dock Animated / Dash2Dock Lite** extension on GNOME Shell, especially under **Wayland** sessions on Ubuntu.

---

# 🐛 Deskripsi Masalah
# 🐛 Problem Description

Jika kamu menggunakan ekstensi **Dash2Dock Animated**, kamu mungkin menyadari bahwa area bawah layar (tepat di atas dock) tiba-tiba menjadi:

If you are using the **Dash2Dock Animated** extension, you may notice that the bottom area of the screen (right above the dock) suddenly becomes:

- tidak bisa diklik  
  unclickable
- seperti tertutup jendela transparan  
  blocked by an invisible transparent layer
- memblokir tombol aplikasi di belakangnya  
  prevents clicking apps behind it

Fenomena ini biasa disebut:

This phenomenon is commonly called:

- Dead Zone
- Phantom Window
- Invisible Hitbox
- Ghost Input Region

---

## Gejala
## Symptoms

- Tombol aplikasi di dekat dock tidak merespons klik  
  App buttons near the dock do not respond to clicks
- Tombol browser atau editor seperti VSCode tidak bisa ditekan  
  Browser or editor buttons like VSCode cannot be clicked
- Area kosong di atas dock terasa seperti “tertutup sesuatu”  
  The empty area above the dock feels blocked by something invisible
- Saat animasi hover/magnify dimatikan, masalah langsung hilang  
  The issue disappears when hover/magnify animation is disabled

---

# 🔍 Penyebab Teknis (Root Cause)
# 🔍 Technical Root Cause

Masalah ini disebabkan oleh **reactive input region** atau **bounding box transparan** yang dibuat oleh ekstensi dock untuk mendeteksi hover mouse.

This issue is caused by a **reactive input region** or **transparent bounding box** created by the dock extension to detect mouse hover.

Di dalam `dock.js`, extension menghitung ukuran footprint (`fp`) menggunakan rumus:

Inside `dock.js`, the extension calculates a footprint (`fp`) value using:

```javascript
let magnify = this.extension.animation_magnify * 1.8;
let fp = iconSize * 2 + iconSize * (0.6 * (1 + magnify));
```

Nilai `fp` ini ternyata bukan hanya untuk animasi.

This `fp` value is not only used for animation.

Nilai tersebut juga dipakai sebagai:

It is also used as:

- tinggi actor dock  
  dock actor height
- area collision mouse  
  mouse collision area
- reactive region  
  reactive input region
- allocation box GNOME Shell  
  GNOME Shell allocation box

Akibatnya:

As a result:

```text
Visual Dock Kecil
Tetapi Hitbox Dock Sangat Besar

Small Visual Dock
But Extremely Large Hitbox
```

Area transparan inilah yang memblokir klik mouse pada aplikasi di belakangnya.

This transparent region is what blocks mouse interaction with applications behind it.

---

# ⚠️ Kenapa Masalah Lebih Terasa di Wayland?
# ⚠️ Why Is This Worse on Wayland?

Pada Wayland:

Under Wayland:

- input region lebih strict  
  input regions are stricter
- setiap surface benar-benar memiliki ownership event  
  every surface strictly owns its input events
- actor transparan tetap menerima input mouse  
  transparent actors still receive mouse input

Akibatnya invisible hitbox benar-benar memakan event klik.

As a result, the invisible hitbox fully captures mouse click events.

Sedangkan pada X11, event forwarding masih lebih longgar sehingga bug kadang tidak terlalu terasa.

Meanwhile on X11, event forwarding is more permissive, so the bug is sometimes less noticeable.

---

# 🛠️ Solusi Perbaikan (Patch)
# 🛠️ Patch Solution

## Langkah 1 — Buka Direktori Extension
## Step 1 — Open the Extension Directory

```bash
cd ~/.local/share/gnome-shell/extensions/dash2dock-lite@icedman.github.com
```

Catatan:
Nama folder bisa sedikit berbeda tergantung versi extension.

Note:
The folder name may differ slightly depending on the extension version.

---

# Langkah 2 — Edit `dock.js`
# Step 2 — Edit `dock.js`

Buka file:

Open the file:

```bash
code dock.js
```

atau editor lain seperti:

or use another editor such as:

```bash
nano dock.js
```

---

# Langkah 3 — Perbaiki Nilai Footprint (`fp`)
# Step 3 — Fix the Footprint (`fp`) Value

Cari bagian:

Find this section:

```javascript
// computation derived from animation scale
let magnify = this.extension.animation_magnify * 1.8;
let fp = iconSize * 2 + iconSize * (0.6 * (1 + magnify));
```

Ganti menjadi:

Replace it with:

```javascript
// computation derived from animation scale
let magnify = this.extension.animation_magnify * 1.8;
let fp = iconSize * 0.9 + iconSize * (0.6 * (1 + magnify));
```

---

# 🧠 Kenapa Ini Memperbaiki Bug?
# 🧠 Why Does This Fix the Bug?

Sebelumnya:

Previously:

```text
fp terlalu besar
↓
Dock actor terlalu tinggi
↓
Invisible hitbox meluas ke atas
↓
Klik mouse terblokir

fp too large
↓
Dock actor becomes too tall
↓
Invisible hitbox expands upward
↓
Mouse clicks get blocked
```

Sesudah diperkecil:

After reducing it:

```text
Hitbox menyesuaikan ukuran dock
↓
Area transparan mengecil
↓
Klik mouse kembali normal

Hitbox matches dock size
↓
Transparent area shrinks
↓
Mouse clicks work normally again
```

---

# ⚠️ Masalah Baru: Animasi Bounce / Magnify Terpotong
# ⚠️ New Problem: Bounce / Magnify Animation Gets Clipped

Setelah `fp` diperkecil, biasanya muncul efek samping:

After reducing `fp`, a side effect usually appears:

- icon bounce terpotong  
  bounce animation gets clipped
- magnify kepotong bagian atas  
  magnify effect gets cut off
- animasi tidak bebas keluar dock  
  animation cannot overflow outside the dock

Penyebabnya adalah:

This happens because of:

```javascript
clip_to_allocation: true,
```

pada constructor utama dock.

inside the main dock constructor.

GNOME Shell akan memotong semua child actor yang keluar dari ukuran parent container.

GNOME Shell clips every child actor that renders outside the parent container size.

Karena sekarang ukuran dock lebih kecil, icon animasi jadi keluar dari container lalu di-clip compositor GNOME.

Since the dock size is now smaller, animated icons overflow outside the container and get clipped by the GNOME compositor.

---

# ✅ Solusi Tambahan (Penting)
# ✅ Additional Important Fix

Cari bagian constructor:

Find the constructor section:

```javascript
super._init({
```

Temukan:

Find:

```javascript
clip_to_allocation: true,
```

Ubah menjadi:

Replace it with:

```javascript
clip_to_allocation: false,
```

---

# 🔥 Kenapa Ini Penting?
# 🔥 Why Is This Important?

Dengan `clip_to_allocation: false`:

With `clip_to_allocation: false`:

- icon boleh render keluar container  
  icons can render outside the container
- bounce animation tidak terpotong  
  bounce animation no longer gets clipped
- magnify kembali smooth  
  magnify becomes smooth again
- dock tetap punya hitbox kecil  
  the dock still keeps a small hitbox
- dead zone tetap hilang  
  dead zones remain fixed

---

# ✅ Kombinasi Patch Terbaik
# ✅ Best Patch Combination

Gunakan dua patch berikut sekaligus:

Use these two patches together:

## Patch 1 — Kecilkan Footprint
## Patch 1 — Reduce Footprint

```javascript
let fp = iconSize * 0.9 + iconSize * (0.6 * (1 + magnify));
```

## Patch 2 — Disable Clipping
## Patch 2 — Disable Clipping

```javascript
clip_to_allocation: false,
```

---

# 🎯 Hasil Akhir
# 🎯 Final Result

Dengan kombinasi ini:

With this combination:

✅ Dead zone hilang  
✅ Dead zones removed

✅ Area bawah layar bisa diklik lagi  
✅ Bottom screen area becomes clickable again

✅ Hover animation tetap smooth  
✅ Hover animation remains smooth

✅ Bounce animation tidak kepotong  
✅ Bounce animation no longer clipped

✅ Magnify tetap berjalan normal  
✅ Magnify works correctly

✅ Tidak ada invisible phantom window  
✅ No more invisible phantom window

---

# 🔄 Reload GNOME Shell
# 🔄 Reload GNOME Shell

## Pengguna X11
## X11 Users

Tekan:

Press:

```text
Alt + F2
```

lalu ketik:

then type:

```text
r
```

kemudian Enter.

then press Enter.

---

## Pengguna Wayland
## Wayland Users

Wayland tidak mendukung restart GNOME Shell menggunakan `Alt + F2`.

Wayland does not support restarting GNOME Shell using `Alt + F2`.

Kamu WAJIB:

You MUST:

```text
Log Out → Log In kembali

Log Out → Log In again
```

agar perubahan JS extension dimuat ulang.

so the extension changes can be reloaded.

---

# 🧪 Tips Eksperimen
# 🧪 Experiment Tips

Jika dock masih terasa terlalu besar atau terlalu kecil:

If the dock still feels too large or too small:

Coba ubah:

Try adjusting:

```javascript
iconSize * 0.8
iconSize * 0.9
iconSize * 1.0
iconSize * 1.1
```

Biasanya nilai paling stabil ada di:

The most stable values are usually around:

```text
0.9 sampai 1.1

0.9 to 1.1
```

---

# 🧠 Analisis Teknis Tambahan
# 🧠 Additional Technical Analysis

Masalah ini adalah contoh klasik:

This issue is a classic example of:

```text
Visual Region ≠ Interactive Region
```

pada GUI modern berbasis compositor.

in modern compositor-based GUIs.

Extension menggunakan satu variabel (`fp`) untuk:

The extension uses a single variable (`fp`) for:

- render size
- hover trigger
- collision area
- allocation box

Padahal idealnya:
render area dan input area dipisah.

Ideally, render area and input area should be separated.

Karena itu patch ini bekerja dengan:

That is why this patch works by:

1. mengecilkan input footprint  
   reducing the input footprint

2. mengizinkan visual overflow keluar actor  
   allowing visual overflow outside the actor

---

# 🏷️ Keywords

`gnome-shell-extension`  
`dash-to-dock-animated`  
`dash2dock-lite`  
`wayland-bug`  
`ubuntu-unclickable-screen`  
`phantom-window`  
`hitbox-fix`  
`javascript-patch`  
`clip_to_allocation`  
`gnome-shell-deadzone`  
`dock-hover-bug`  
`invisible-hitbox`
