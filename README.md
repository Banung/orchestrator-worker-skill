# Orchestrator–Worker Skills

Dua skill pribadi yang aku buat sendiri untuk kebutuhanku saat menggunakan AI untuk coding. Repository ini menyimpan skill tersebut beserta cara pakainya.

## Isinya

- [orchestrator](orchestrator/SKILL.md): membaca konteks repository, menentukan pendekatan, lalu membuat instruksi kerja (`WORK ORDER`). Perannya read-only, bukan mengedit file atau menjalankan test/build.
- [worker](worker/SKILL.md): mengerjakan instruksi tersebut sesuai scope, melakukan validasi, lalu memberikan laporan (`WORKER REPORT`).

Keduanya dipakai di chat terpisah. Pemindahan instruksi dan laporan dilakukan manual dengan copy-paste, bukan lewat komunikasi otomatis antar-model.

## Cara pakai

1. Aktifkan `orchestrator` di chat pertama dan jelaskan tugas yang ingin dikerjakan.
2. Salin seluruh `WORK ORDER` yang dihasilkan ke chat kedua yang memakai `worker`.
3. Worker mengerjakan tugas dan mengembalikan `WORKER REPORT` berisi hasil, validasi, serta kendala jika ada.
4. Periksa hasilnya sendiri, atau salin laporan kembali ke orchestrator kalau ingin meminta review.

Kalau bukti belum cukup untuk menentukan implementasi, orchestrator bisa memberi tugas `DISCOVERY`: mengumpulkan informasi tanpa mengubah file atau state. Tugas `IMPLEMENTATION` mengizinkan perubahan sesuai batas yang tertulis di work order.

Work order dan laporan memakai bahasa Inggris; percakapan di luar itu mengikuti bahasa pengguna.

## Memasang skill

Salin folder `orchestrator` dan `worker` ke lokasi skill yang dibaca aplikasi yang dipakai. Untuk aplikasi yang mendukung `~/.agents/skills`, jalankan dari root repository ini:

```sh
mkdir -p ~/.agents/skills
cp -R -i orchestrator ~/.agents/skills/
cp -R -i worker ~/.agents/skills/
```

Opsi `-i` meminta konfirmasi sebelum menimpa file yang sudah ada. Muat ulang skill sesuai cara aplikasi tersebut. Lokasi instalasi dan dukungan frontmatter bisa berbeda antar-aplikasi.

## Catatan

Ini instruksi untuk model, bukan pembatas izin tool. Kalau orchestrator perlu benar-benar read-only, tool yang bisa mengubah file atau state tetap perlu dinonaktifkan di aplikasinya. Hasil kerja dan klaim validasi worker juga tetap perlu diperiksa.

