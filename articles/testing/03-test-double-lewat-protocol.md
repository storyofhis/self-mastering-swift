---
title: "Test Double: Fake, Stub, Spy lewat Protocol"
category: "Testing"
status: draft
tags:
  - swift-mastering
  - swift/testing
  - article
---

# Test Double: Fake, Stub, Spy lewat Protocol

> **Kategori:** Testing · **Level:** Menengah · **Status:** 🚧 kerangka

## Kerangka isi

### 1. Istilah yang sering tertukar
| | Untuk |
|---|---|
| **Dummy** | Mengisi parameter, tidak dipakai |
| **Stub** | Mengembalikan jawaban tetap |
| **Spy** | Stub + mencatat apa yang dipanggil |
| **Fake** | Implementasi sederhana yang benar-benar bekerja (in-memory store) |
| **Mock** | Punya ekspektasi yang diverifikasi |

Di Swift, `struct` fake sederhana biasanya cukup — framework mocking jarang sepadan.

### 2. Protokol sebagai seam
```swift
protocol MusicSearching { ... }              // ✅ ada di project movie
final class LibraryStore { ... }             // ❌ tidak ada protokolnya
```
- Utang konkret: `SearchViewModel.toggleSave` tidak bisa diuji tanpa `UserDefaults` nyata
- Perbaikan: `protocol LibraryStoring` + `extension LibraryStore: LibraryStoring {}`

### 3. Berapa banyak protokol itu terlalu banyak
- Protokol untuk setiap kelas = "eleven files that all forward the same call"
- Aturan: buat seam di **boundary I/O** (jaringan, disk, jam, lokasi, notifikasi),
  bukan di setiap kolaborasi internal

### 4. Menyuntikkan waktu & keacakan
```swift
protocol Clock { var now: Date { get } }
init(clock: Clock = SystemClock())
```
Ini seam yang paling sering dilupakan dan paling sering menyebabkan test flaky.

### 5. Fixture yang enak dipakai
```swift
extension Track {
    static func fixture(id: Int = 1, title: String = "Song",
                        durationMillis: Int? = 225_000) -> Track { ... }
}
```
Default value di mana-mana; test hanya menyebut yang relevan bagi test itu.

## Cek pemahaman (draft)
1. Kenapa fake in-memory sering lebih baik dari mock framework?
2. Di mana kamu meletakkan seam dan di mana tidak?
3. Kenapa `Date()` langsung di dalam kode produksi membuat test rapuh?
