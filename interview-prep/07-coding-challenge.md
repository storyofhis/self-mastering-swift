---
title: "Interview Prep 07 — Coding Challenge"
category: "Interview Prep"
status: draft
tags:
  - swift-mastering
  - interview-prep
  - interview
---

# Interview Prep 07 — Coding Challenge

> **Status:** 🚧 kerangka — daftar soal sudah ada, solusi & pembahasan belum ditulis

## Yang sebenarnya diuji di live coding iOS

Bukan LeetCode Hard. Yang diuji:
1. Apakah kamu menulis Swift **idiomatik** (bukan Java/JS dengan sintaks Swift)
2. Apakah kamu memikirkan edge case tanpa diminta
3. Apakah kamu bicara sambil berpikir
4. Apakah kamu bisa menerima masukan tanpa defensif

## Kategori 1 — Swift idiomatik (paling sering)

1. Implementasikan `debounce` dan `throttle`
2. Implementasikan cache LRU dengan `NSCache`-like API
3. Implementasikan retry dengan exponential backoff + jitter
4. Kelompokkan array berdasarkan key → `Dictionary(grouping:by:)`
5. Ratakan array bersarang tanpa `flatMap` bawaan
6. Implementasikan tipe COW sederhana *(lihat
   [artikel COW](../articles/swift-language/02-copy-on-write-internals.md))*
7. Implementasikan `Result`-like enum dengan `map`/`flatMap`
8. Parse response JSON heterogen dengan lossy decoding *(lihat
   [artikel networking](../articles/ios-uikit/03-networking-urlsession-codable.md))*

## Kategori 2 — Concurrency

9. Unduh N URL dengan konkurensi maksimum M *(pola bounded task group)*
10. Buat image loader yang men-dedup request in-flight *(actor + `Task` di cache)*
11. Konversi delegate berbasis callback jadi `AsyncStream`
12. Implementasikan `withTimeout` untuk operasi async

## Kategori 3 — UIKit langsung di depan interviewer

13. Buat `UITableView` dengan search + debounce dari nol
14. Buat custom view dengan Auto Layout programmatic
15. Perbaiki bug gambar ketuker saat scroll *(diberi kode yang sudah bug)*
16. Temukan retain cycle di potongan kode yang diberikan

## Kategori 4 — Algoritma (kalau perusahaannya besar)

17. Two pointers, sliding window, binary search
18. Traversal tree/graph (BFS/DFS)
19. Hash map untuk deduplikasi/counting
20. Sorting dengan comparator kustom

Fokus di 17–20 hanya kalau kamu melamar ke perusahaan yang memang memakai soal
algoritma. Untuk kebanyakan perusahaan iOS, kategori 1–3 jauh lebih mungkin.

## Aturan saat mengerjakan

- **Bicara sebelum menulis.** "Saya akan pakai dictionary untuk mapping ID ke item
  supaya lookup-nya O(1)."
- **Tulis signature dulu**, isi belakangan.
- **Sebutkan edge case sambil menulis**: array kosong, duplikat, nilai negatif,
  Unicode, pembatalan.
- **Jangan diam lebih dari 20 detik.** Kalau buntu, katakan apa yang kamu pikirkan.
- **Setelah selesai, uji sendiri** dengan satu contoh kecil, keras-keras.

## Yang harus ditulis
Solusi lengkap + pembahasan kompleksitas + varian follow-up untuk tiap nomor.
