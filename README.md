# 🚀 Solusi Layar Bawah Tidak Bisa Diklik (Dead Zone / Phantom Window) di Ubuntu/GNOME

Repositori ini berisi panduan teknis dan patch untuk mengatasi masalah **area bawah layar tidak bisa diklik** saat menggunakan ekstensi **Dash2Dock Animated / Dash2Dock Lite** di GNOME Shell, terutama pada sesi **Wayland** di Ubuntu.

---

# 🐛 Deskripsi Masalah

Jika kamu menggunakan ekstensi **Dash2Dock Animated**, kamu mungkin menyadari bahwa area bawah layar (tepat di atas dock) tiba-tiba menjadi:

- tidak bisa diklik
- seperti tertutup jendela transparan
- memblokir tombol aplikasi di belakangnya

Fenomena ini biasa disebut:

- Dead Zone
- Phantom Window
- Invisible Hitbox
- Ghost Input Region

---

## Gejala

- Tombol aplikasi di dekat dock tidak merespons klik
- Tombol browser atau editor seperti VSCode tidak bisa ditekan
- Area kosong di atas dock terasa seperti “tertutup sesuatu”
- Saat animasi hover/magnify dimatikan, masalah langsung hilang

---

# 🔍 Penyebab Teknis (Root Cause)

Masalah ini disebabkan oleh **reactive input region** atau **bounding box transparan** yang dibuat oleh ekstensi dock untuk mendeteksi hover mouse.

Di dalam `dock.js`, extension menghitung ukuran footprint (`fp`) menggunakan rumus:

```javascript
let magnify = this.extension.animation_magnify * 1.8;
let fp = iconSize * 2 + iconSize * (0.6 * (1 + magnify));
```

Nilai `fp` ini ternyata bukan hanya untuk animasi.

Nilai tersebut juga dipakai sebagai:

- tinggi actor dock
- area collision mouse
- reactive region
- allocation box GNOME Shell

Akibatnya:

```text
Visual Dock Kecil
Tetapi Hitbox Dock Sangat Besar
```

Area transparan inilah yang memblokir klik mouse pada aplikasi di belakangnya.

---

# ⚠️ Kenapa Masalah Lebih Terasa di Wayland?

Pada Wayland:

- input region lebih strict
- setiap surface benar-benar memiliki ownership event
- actor transparan tetap menerima input mouse

Akibatnya invisible hitbox benar-benar memakan event klik.

Sedangkan pada X11, event forwarding masih lebih longgar sehingga bug kadang tidak terlalu terasa.

---

# 🛠️ Solusi Perbaikan (Patch)

## Langkah 1 — Buka Direktori Extension

```bash
cd ~/.local/share/gnome-shell/extensions/dash2dock-lite@icedman.github.com
```

Catatan:
Nama folder bisa sedikit berbeda tergantung versi extension.

---

# Langkah 2 — Edit `dock.js`

Buka file:

```bash
code dock.js
```

atau editor lain seperti:

```bash
nano dock.js
```

---

# Langkah 3 — Perbaiki Nilai Footprint (`fp`)

Cari bagian:

```javascript
// computation derived from animation scale
let magnify = this.extension.animation_magnify * 1.8;
let fp = iconSize * 2 + iconSize * (0.6 * (1 + magnify));
```

Ganti menjadi:

```javascript
// computation derived from animation scale
let magnify = this.extension.animation_magnify * 1.8;
let fp = iconSize * 0.9 + iconSize * (0.6 * (1 + magnify));
```

---

# 🧠 Kenapa Ini Memperbaiki Bug?

Sebelumnya:

```text
fp terlalu besar
↓
Dock actor terlalu tinggi
↓
Invisible hitbox meluas ke atas
↓
Klik mouse terblokir
```

Sesudah diperkecil:

```text
Hitbox menyesuaikan ukuran dock
↓
Area transparan mengecil
↓
Klik mouse kembali normal
```

---

# ⚠️ Masalah Baru: Animasi Bounce / Magnify Terpotong

Setelah `fp` diperkecil, biasanya muncul efek samping:

- icon bounce terpotong
- magnify kepotong bagian atas
- animasi tidak bebas keluar dock

Penyebabnya adalah:

```javascript
clip_to_allocation: true,
```

pada constructor utama dock.

GNOME Shell akan memotong semua child actor yang keluar dari ukuran parent container.

Karena sekarang ukuran dock lebih kecil, icon animasi jadi keluar dari container lalu di-clip compositor GNOME.

---

# ✅ Solusi Tambahan (Penting)

Cari bagian constructor:

```javascript
super._init({
```

Temukan:

```javascript
clip_to_allocation: true,
```

Ubah menjadi:

```javascript
clip_to_allocation: false,
```

---

# 🔥 Kenapa Ini Penting?

Dengan `clip_to_allocation: false`:

- icon boleh render keluar container
- bounce animation tidak terpotong
- magnify kembali smooth
- dock tetap punya hitbox kecil
- dead zone tetap hilang

---

# ✅ Kombinasi Patch Terbaik

Gunakan dua patch berikut sekaligus:

## Patch 1 — Kecilkan Footprint

```javascript
let fp = iconSize * 0.9 + iconSize * (0.6 * (1 + magnify));
```

## Patch 2 — Disable Clipping

```javascript
clip_to_allocation: false,
```

---

# 🎯 Hasil Akhir

Dengan kombinasi ini:

✅ Dead zone hilang  
✅ Area bawah layar bisa diklik lagi  
✅ Hover animation tetap smooth  
✅ Bounce animation tidak kepotong  
✅ Magnify tetap berjalan normal  
✅ Tidak ada invisible phantom window  

---

# 🔄 Reload GNOME Shell

## Pengguna X11

Tekan:

```text
Alt + F2
```

lalu ketik:

```text
r
```

kemudian Enter.

---

## Pengguna Wayland

Wayland tidak mendukung restart GNOME Shell menggunakan `Alt + F2`.

Kamu WAJIB:

```text
Log Out → Log In kembali
```

agar perubahan JS extension dimuat ulang.

---

# 🧪 Tips Eksperimen

Jika dock masih terasa terlalu besar atau terlalu kecil:

Coba ubah:

```javascript
iconSize * 0.8
iconSize * 0.9
iconSize * 1.0
iconSize * 1.1
```

Biasanya nilai paling stabil ada di:

```text
0.9 sampai 1.1
```

---

# 🧠 Analisis Teknis Tambahan

Masalah ini adalah contoh klasik:

```text
Visual Region ≠ Interactive Region
```

pada GUI modern berbasis compositor.

Extension menggunakan satu variabel (`fp`) untuk:

- render size
- hover trigger
- collision area
- allocation box

Padahal idealnya:
render area dan input area dipisah.

Karena itu patch ini bekerja dengan:

1. mengecilkan input footprint
2. mengizinkan visual overflow keluar actor

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
