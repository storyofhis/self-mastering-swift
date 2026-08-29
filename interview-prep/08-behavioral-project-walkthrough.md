---
title: "Interview Prep 08 — Behavioral & Project Walkthrough"
category: "Interview Prep"
status: draft
tags:
  - swift-mastering
  - interview-prep
  - interview
---

# Interview Prep 08 — Behavioral & Project Walkthrough

> **Status:** 🚧 kerangka — struktur sudah ada, cerita pribadi harus kamu isi sendiri

## Kenapa bagian ini sering menentukan

Kandidat dengan jawaban teknis setara akan dibedakan di sini. Dan berbeda dari
soal teknis, ini **bisa disiapkan sepenuhnya** — jawabannya tidak berubah.

## Format STAR

| | Isi | Porsi waktu |
|---|---|---|
| **S**ituation | Konteks secukupnya | 15% |
| **T**ask | Apa tanggung jawabmu spesifiknya | 15% |
| **A**ction | Apa yang **kamu** lakukan (bukan "kami") | 55% |
| **R**esult | Hasil, sebisa mungkin dengan angka | 15% |

Kesalahan paling umum: menghabiskan 70% waktu di Situation.

## Project walkthrough — struktur 5 menit

Latih dengan project `movie` (atau project apa pun yang kamu kerjakan sendiri):

1. **Konteks (20 detik)** — "App tiga tab, klien iTunes Search API, UIKit
   programmatic, MVVM, async/await."
2. **Satu keputusan arsitektur + alternatif yang ditolak (90 detik)** —
   "MVVM, bukan VIPER. VIPER memberi Router untuk navigasi, dan itu memang gap saya —
   tapi lima file per layar untuk tiga layar dengan satu edge navigasi tidak sepadan."
3. **Satu masalah teknis nyata + solusinya (90 detik)** —
   "iTunes `/lookup` mencampur entri album ke dalam daftar lagu. Alih-alih
   special-case, saya buat wrapper yang men-decode tiap elemen individual dan
   membuang yang gagal, jadi satu entri rusak tidak menjatuhkan layar."
4. **Harga yang saya bayar (60 detik)** —
   "`LibraryStore` tidak di balik protokol, jadi belum bisa di-fake di test.
   Itu hal pertama yang akan saya perbaiki."
5. **Apa yang saya pelajari (30 detik)**

Siapkan juga versi 15 menit yang memperdalam poin 3.

## Bank pertanyaan behavioral

Tulis satu cerita spesifik untuk masing-masing. Satu cerita boleh dipakai untuk
beberapa pertanyaan.

1. Ceritakan bug tersulit yang pernah kamu perbaiki. Bagaimana kamu menemukannya?
2. Ceritakan saat kamu tidak setuju dengan keputusan teknis rekan/atasan.
3. Ceritakan saat kamu membuat kesalahan yang berdampak ke user.
4. Ceritakan saat kamu harus belajar teknologi baru dengan cepat.
5. Ceritakan trade-off yang kamu ambil karena tenggat waktu, dan bagaimana kamu
   membayar utang teknisnya.
6. Ceritakan saat kamu menerima code review yang keras.
7. Ceritakan saat kamu harus menjelaskan hal teknis ke orang non-teknis.
8. Apa yang membuatmu bangga dari kode yang kamu tulis?
9. Bagaimana kamu memutuskan kapan berhenti mengoptimasi?
10. Kenapa kamu ingin pindah / kenapa perusahaan ini?

## Pertanyaan yang HARUS kamu ajukan balik

Ini dinilai. Pertanyaan yang bagus menunjukkan kamu memikirkan pekerjaannya:

- Bagaimana proses code review di tim ini?
- Berapa lama dari commit ke produksi? Bagaimana rilisnya?
- Bagaimana tim memutuskan menambah dependency?
- Sudah di Swift 6 language mode? Kalau belum, apa yang menahan?
- Berapa persentase waktu untuk fitur baru vs perawatan?
- Apa hal yang paling membuat frustrasi engineer di tim ini sekarang?

Pertanyaan terakhir itu sering memberi jawaban paling jujur.

## Yang harus kamu isi sendiri
Tulis satu paragraf STAR untuk setiap nomor 1–10 di file terpisah, lalu latih
mengucapkannya keras-keras. Menulis saja tidak cukup — yang diuji adalah
mengucapkannya dengan lancar.
