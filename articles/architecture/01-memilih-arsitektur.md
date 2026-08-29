---
title: "MVC → MVVM → VIPER → TCA: Memilih Berdasarkan Biaya, Bukan Tren"
category: "Architecture"
status: draft
tags:
  - swift-mastering
  - ios/architecture
  - article
---

# MVC → MVVM → VIPER → TCA: Memilih Berdasarkan Biaya, Bukan Tren

> **Kategori:** Architecture · **Level:** Menengah–Lanjut · **Status:** 🚧 kerangka

## Tesis artikel ini
Tidak ada arsitektur yang "lebih baik". Ada arsitektur yang **biayanya dibayar** pada
ukuran tim dan kompleksitas tertentu. Tugasmu adalah menyebut titik itu, bukan memilih
pemenang.

## Kerangka isi

### 1. Kriteria yang dipakai membandingkan
- Jumlah file per fitur
- Kemudahan menulis test tanpa UI
- Berapa orang bisa bekerja paralel tanpa konflik
- Berapa lama orang baru bisa produktif
- Apa yang rusak saat fitur bertambah

### 2. MVC (Apple-style)
- Kenapa ia dituduh "Massive View Controller" — dan kenapa itu sering salah alamat
  (masalahnya bukan MVC, tapi tidak adanya lapisan lain sama sekali)
- Kapan MVC masih jawaban yang benar: app kecil, layar sederhana, tim satu orang

### 3. MVVM
- Yang dibeli: logika bisa diuji tanpa UI
- Yang tidak diselesaikan: **navigasi** (gap yang jujur dicatat di
  `DECISIONS.md` project `movie`)
- Binding: closure vs Combine vs `@Observable` — dan kapan naik tingkat

### 4. VIPER
- Router memberi navigasi rumah yang eksplisit
- Harga: ~5 file per layar. Untuk 3 layar dengan 1 edge navigasi, itu
  "eleven files that all forward the same call untouched"
- Kapan dibayar: tim besar, banyak layar, navigasi bercabang

### 5. Clean Architecture
- Dependency Rule dalam bentuk mini: ViewModel bergantung pada protokol, bukan
  implementasi — inilah yang project `movie` **pertahankan**
- Yang ditolak: entity per ring, mapper, use case layer
- Harga yang diakui: `Track` melayani dua peran (wire model + domain model), jadi
  perubahan JSON terlihat oleh semua ViewModel

### 6. TCA
- Unidirectional data flow, state yang bisa di-*replay*, test yang sangat kuat
- Harga: kurva belajar, ukuran binary, dan kamu terikat pada library pihak ketiga

### 7. Cara menjawab "arsitektur apa yang kamu pakai?" di interview
Jangan sebut nama. Sebut **keputusan + alternatif yang ditolak + harga**.

## Sumber untuk ditulis
- Project `movie`: `DECISIONS.md`, `REFLECTION.md`
