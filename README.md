# Antry — Pembagi Jam Akses biar Server Nggak Tumbang

<div align="center">

<img src="assets/logo-ant.png" width="128" alt="Maskot Antry" />

**Login bareng bikin down. Antry bagi jamnya.**

[![Next.js](https://img.shields.io/badge/Next.js-16-000000)](https://nextjs.org)
[![React](https://img.shields.io/badge/React-19-61dafb)](https://react.dev)
[![TypeScript](https://img.shields.io/badge/TypeScript-5-3178c6)](https://www.typescriptlang.org)
[![Drizzle](https://img.shields.io/badge/Drizzle-ORM-c5f74f)](https://orm.drizzle.team)
[![Postgres](https://img.shields.io/badge/Neon-Postgres-00e599)](https://neon.tech)

</div>

---

## Masalahnya

Bukan serangan. Bukan bug. Cuma **500 orang yang membuka situs di detik yang sama**.

Situs kampus, layanan pendaftaran, toko online saat flash sale — semuanya tumbang karena
pengunjung datang bersamaan lalu menekan refresh berulang saat server mulai lambat. Retry
beruntun itu justru menyelesaikan pekerjaan yang sudah dimulai beban awal.

Host tidak punya cara bilang *"jangan sekarang, coba jam 10:30"* tanpa menempel pengumuman di
tempat yang tidak dibaca siapa pun.

## Antry

Antry memotong trafik jadi **sesi waktu yang bisa dipesan**. Host membuat antrean, membagikan
satu tautan, pengunjung memilih hari dan jam — tanpa akun, tanpa aplikasi, tanpa antre di depan
layar.

**500 pengunjung → 24 sesi × 50 kursi.** Server tetap hidup, pengunjung tetap dapat akses, host
punya data siapa datang kapan.

### Fitur

- **Antrean sesi** — slot 30 menit (atau mode harian), kuota per sesi, tautan pendek `/a/[kode]`
- **Rekomendasi jam sepi** — sesi dengan okupansi terendah ditandai `Direkomendasi · jam sepi`,
  lengkap dengan tombol **Pakai jam ini**
- **Bar beban per sesi** — hijau / kuning / merah, plus peringatan saat sesi pilihan hampir penuh
- **Wizard 3 langkah** untuk host — nama & tanggal → jam & kuota → simpan & tautan
- **Heatmap hari × jam** — host langsung lihat jam mana yang menumpuk
- **Daftar peserta + check-in** — kode booking, nama otomatis, penanda kesamaan perangkat
- **Pengumuman per situs** — gangguan, jam ramai, pemeliharaan, dengan laporan dari pengunjung
- **Panel admin** — setujui host baru, tinjau laporan, tarik pengumuman palsu
- **Kode akses per perangkat** — satu perangkat satu booking per antrean, tanpa akun peserta
- **Mobile-first** — alur pilih sesi dirancang untuk dibuka dari WhatsApp di layar kecil

---

## Tampilan

<div align="center">

**Beranda** — cerita singkat, papan status, dan jam WIB yang hidup.

![Beranda](screenshots/01-beranda.webp)

| Jadwal per situs | Wajib dibaca sebelum antri |
| :---: | :---: |
| ![Jadwal situs](screenshots/02-jadwal-situs.webp) | ![Wajib dibaca](screenshots/03-wajib-dibaca.webp) |

**Pilih hari & jam** — rekomendasi jam sepi, bar beban, dan sisa kursi per sesi.

![Pilih sesi](screenshots/04-pilih-sesi.webp)

</div>

| Sesi dipilih | Panel host |
| :---: | :---: |
| ![Sesi dipilih](screenshots/05-sesi-terpilih.webp) | ![Panel host](screenshots/06-panel-host.webp) |

**Daftar antrean** — bar beban per antrean, plus peringatan saat kursi hampir habis.

![Daftar antrean](screenshots/07-daftar-antrean.webp)

**Kelola antrean** — tautan pendek + QR untuk dibagikan, sisa kuota per sesi, daftar peserta.

![Detail antrean](screenshots/08-detail-antrean.webp)

| Statistik & heatmap | Panel admin |
| :---: | :---: |
| ![Statistik](screenshots/09-statistik.webp) | ![Panel admin](screenshots/10-panel-admin.webp) |

<div align="center">

**Di HP** — pengunjung datang dari tautan WhatsApp, jadi alurnya diuji di 390px.

| Beranda | Wajib dibaca | Pilih sesi |
| :---: | :---: | :---: |
| ![Mobile beranda](screenshots/11-mobile-beranda.webp) | ![Mobile wajib dibaca](screenshots/12-mobile-wajib-dibaca.webp) | ![Mobile pilih sesi](screenshots/13-mobile-pilih-sesi.webp) |

</div>

---

## Arsitektur

```mermaid
flowchart TB
    subgraph VISITOR["Pengunjung (tanpa akun)"]
        BOARD["Papan akses publik<br/>status & jadwal"]
        QUEUE["Pilih sesi<br/>/a/[kode]"]
        TICKET["Tiket akses<br/>kode booking"]
    end
    subgraph HOST["Host"]
        WIZARD["Wizard antrean<br/>3 langkah"]
        MANAGE["Kelola antrean<br/>QR · kuota · peserta"]
        STATS["Statistik<br/>heatmap & tren"]
    end
    subgraph ADMIN["Admin"]
        REVIEW["Tinjauan konten<br/>laporan · host baru"]
    end
    subgraph APP["Next.js App Router"]
        ACTION["Server Actions"]
        AUTH["Better Auth<br/>admin · host · viewer"]
    end
    subgraph DATA["Data"]
        PG[("Neon Postgres")]
        DRIZ["Drizzle ORM"]
        PGL[("PGlite<br/>dev lokal")]
    end
    BOARD --> QUEUE --> TICKET
    WIZARD & MANAGE & STATS --> ACTION
    REVIEW --> ACTION
    QUEUE --> ACTION
    ACTION --> AUTH --> DRIZ
    DRIZ --> PG
    DRIZ -.-> PGL
```

Keputusan desain yang penting:

- **Status waktu dihitung dari timestamp, bukan disimpan.** Buka/tutup sesi, "sedang berlangsung",
  dan "selesai" diturunkan saat dibaca — tidak ada cron yang bisa telat jalan.
- **Satu statement SQL untuk mutasi kritis.** Driver HTTP Neon tidak punya transaksi interaktif, jadi
  booking + cek kuota + cek booking ganda ditulis sebagai satu CTE yang atomik.
- **Rekomendasi jam sepi dihitung di klien.** Data slot sudah dikirim ke halaman, jadi rekomendasi
  berubah instan saat pengunjung berpindah hari tanpa request tambahan.
- **PGlite untuk pengembangan.** Postgres asli yang berjalan sebagai file lokal — `npm run dev`
  tidak butuh akun cloud. Kode yang sama jalan di Neon saat produksi.
- **Peserta tanpa akun, tapi tetap terbendung.** Satu perangkat satu booking per antrean lewat kode
  perangkat; fingerprint ringan hanya dipakai sebagai *sinyal* untuk host, bukan bukti kecurangan.

---

## Stack

| Lapisan | Teknologi |
| :--- | :--- |
| Framework | Next.js 16 (App Router, Turbopack) + React 19 |
| Bahasa | TypeScript |
| Basis data | Neon Postgres (serverless HTTP) |
| ORM | Drizzle (`drizzle-kit` untuk migrasi) |
| Autentikasi | Better Auth — email + password, peran `admin` / `host` / `viewer` |
| Dev lokal | PGlite (Postgres asli, tanpa akun) |
| Styling | Tailwind CSS v4 |
| Konten | MDX, dibangun statis |
| Pembagian waktu | Asia/Jakarta, dihitung eksplisit di server |

---

## Status

Proyek ini **berjalan sebagai produk**, dan source code-nya disimpan privat.

Repository ini adalah **showcase**: dokumentasi produk, arsitektur, tampilan, dan keputusan desain.
Isi `src/` tidak dipublikasikan.

Demo: <https://antry-cyan.vercel.app> — antrean contoh di `/a/demo-jam` dan `/a/demo-harian`.

Untuk pertanyaan, demo, atau diskusi teknis: buka issue di repository ini.

---

## English

**Antry** is a session-based access queue for any website. Hosts create a queue, share one short link, and visitors book a time slot instead of arriving at the same second and hammering refresh. No visitor account is needed — a per-device code is enough.

The point is to cut the retry storm: 500 visitors become 24 sessions of 50 seats, and the host gets data on who arrives when. Hosts and admins sign in; the public board, booking flow, and error articles stay open.

The source code is not published; this repository documents the product.

---

<div align="center">

**Dibuat oleh [Kandar Lubis](https://github.com/kandarlubis31)**

</div>

---

## Lisensi

**All Rights Reserved.** Copyright © 2026 Kandar Lubis.

Kode sumber tidak dipublikasikan dan tidak dilisensikan untuk digunakan ulang. Dokumentasi ini
disajikan untuk peninjauan dan evaluasi.
