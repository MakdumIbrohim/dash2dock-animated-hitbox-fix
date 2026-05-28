# 🚀 Solusi Layar Bawah Tidak Bisa Diklik (Dead Zone / Phantom Window) di Ubuntu/GNOME 

Repositori ini berisi panduan teknis dan perbaikan (*patch*) untuk mengatasi masalah **area bawah layar tidak bisa diklik** saat menggunakan ekstensi **Dash2Dock Animated** (atau Dash2Dock Lite) di GNOME Shell, terutama pada sesi Wayland di Ubuntu.

## 🐛 Deskripsi Masalah

Jika kamu menggunakan ekstensi **Dash to Dock Animated**, kamu mungkin menyadari bahwa area bawah layar (tepat di atas *dock*) tiba-tiba menjadi "kebal klik" (*unclickable*). Kamu tidak bisa mengklik tombol aplikasi, *browser*, atau *tools* lain yang berada di area tersebut.

**Gejala:**
* Ikon atau tombol aplikasi yang berada di dekat *dock* tidak merespons klik (seperti terhalang sesuatu).
* Saat fitur animasi (seperti efek *magnify/hover*) diaktifkan, masalah ini muncul. Jika animasi dimatikan, klik berfungsi normal.

![Area yang tidak terjangkau](image_92fa3c.png)
*Gambar: Ikon pensil di sebelah kanan bawah tidak bisa diklik meskipun secara visual tidak tertutup oleh dock.*

### 🔍 Penyebab Teknis (The Root Cause)

Masalah ini disebabkan oleh pembuatan **"Invisible Bounding Box" (Kotak Transparan)** oleh ekstensi untuk mendeteksi pergerakan kursor (*hover trigger zone*). 

Pada file konfigurasi ekstensi, *developer* menambahkan nilai *padding/margin* yang sangat besar (bisa 2 hingga 3 kali lipat ukuran ikon) agar animasi terlihat mulus dari jarak jauh. Akibatnya, kotak transparan ini meluas ke atas dan memblokir interaksi *mouse* pada aplikasi di bawahnya.

![Area Transparan yang memblokir layar](image_93113d.png)
*Gambar: Garis ungu menunjukkan seberapa besar kotak pemicu (invisible box) yang menutupi layar kamu.*

---

## 🛠️ Cara Memperbaiki (Langkah-langkah Teknis)

Untuk menghilangkan *bug* ini, kita harus memodifikasi nilai perhitungan ukuran *footprint* (`fp`) secara langsung di dalam kode JavaScript ekstensi tersebut.

### Langkah 1: Buka Direktori Ekstensi
Buka terminal dan arahkan ke direktori tempat ekstensi GNOME kamu terinstal:
```bash
cd ~/.local/share/gnome-shell/extensions/dash2dock-lite@icedman.github.com
```
*Catatan: Nama folder mungkin sedikit berbeda tergantung versi yang kamu instal.*

![File Ekstensi](image_93191e.png)

### Langkah 2: Edit File `dock.js`
Buka file `dock.js` menggunakan *text editor* andalanmu (contoh: VS Code, Nano, atau Gedit):
```bash
code dock.js
```

### Langkah 3: Ubah Nilai Footprint (`fp`)
Gunakan fitur pencarian (*Ctrl+F*) dan cari kode fungsi `layout()`. Temukan baris yang berisi deklarasi variabel `fp` (biasanya berada di sekitar baris **1139**).

**Kode Asli:**
```javascript
// computation derived from animation scale
let magnify = this.extension.animation_magnify * 1.8;
let fp = iconSize * 2 + iconSize * (0.6 * (1 + magnify));
```

**Kode Perbaikan:**
Ubah pengali `iconSize * 2` menjadi angka yang jauh lebih kecil, misalnya **`0.9`** atau bahkan **`0.0`** untuk menghilangkan *padding* berlebih tanpa merusak animasi.

```javascript
// computation derived from animation scale
let magnify = this.extension.animation_magnify * 1.8;
let fp = iconSize * 0.9 + iconSize * (0.6 * (1 + magnify)); 
// ATAU gunakan 0.0 jika 0.9 masih dirasa terlalu tinggi
```

> **💡 Tips Eksperimen untuk Pengguna X11:** Agar bisa mengubah ukurannya, pastikan ketik `Alt + F2`, ketik `r`, lalu `Enter` begitu seterusnya sampai mengecil dan menyesuaikan. *(Catatan: Untuk pengguna Wayland, shortcut ini tidak berfungsi, kamu harus melakukan Log Out dan Log In kembali).*

![Bukti Perbaikan Berhasil](image_9df2e6.png)
*Gambar: Bukti penyesuaian kode di VS Code. Garis ungu pemicu animasi kini menempel sempurna dengan dock dan tidak memblokir area layar.*

### Langkah 4: Simpan dan Muat Ulang GNOME Shell (Penting!)
Setelah menyimpan perubahan pada `dock.js`, sistem harus dimuat ulang agar kode baru dieksekusi.

* **Pengguna X11:** Tekan `Alt + F2`, ketik `r`, lalu tekan `Enter`.
* **Pengguna Wayland (Ubuntu Default saat ini):** Kamu **WAJIB melakukan Log Out (Keluar Sesi) dan Log In kembali** ke sistem Ubuntu kamu.

---

## ✅ Hasil

Setelah *login* kembali, area bawah layar kamu akan berfungsi normal 100%, dan animasi *hover/magnify* pada Dash2Dock Animated akan tetap berjalan dengan mulus!

### 🏷️ Tags / Keywords (Untuk Pencarian)
`gnome-shell-extension`, `dash-to-dock-animated`, `dash2dock-lite`, `wayland-bug`, `ubuntu-unclickable-screen`, `phantom-window`, `gnome-40+`, `hitbox-fix`, `javascript-patch`.
