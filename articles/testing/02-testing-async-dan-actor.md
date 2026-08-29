---
title: "Menguji Kode Async & Actor"
category: "Testing"
status: draft
tags:
  - swift-mastering
  - swift/testing
  - article
---

# Menguji Kode Async & Actor

> **Kategori:** Testing · **Level:** Menengah–Lanjut · **Status:** 🚧 kerangka

## Kerangka isi

### 1. Test async itu mudah; test **timing** yang sulit
- `@Test func f() async throws` — langsung bisa `await`
- Masalah nyata: kode yang menyelesaikan pekerjaan lewat callback/`Task` tak terstruktur

### 2. Masalah konkret dari project `movie`
```swift
func load() {
    Task { ... onStateChange?(.loaded) }    // tidak ada cara menunggunya dari test
}
```
Test harus "menunggu" tanpa `sleep` yang rapuh. Tiga solusi:
1. Buat method `async` dan panggil `Task` hanya di call site (paling bersih)
2. Ekspos `Task` yang dibuat supaya test bisa `await`-nya
3. Confirmation/expectation (`confirmation { }` di Swift Testing)

### 3. Test double untuk protokol async
```swift
struct FakeMusicService: MusicSearching {
    var albums: [Album] = []
    var error: Error?
    var delay: Duration = .zero
    func searchAlbums(term: String, limit: Int) async throws -> [Album] {
        try await Task.sleep(for: delay)
        if let error { throw error }
        return albums
    }
}
```
- Menguji cancellation: fake yang menunggu, lalu `task.cancel()`

### 4. Menguji actor
- Reentrancy tidak deterministik → test yang menjalankan N task bersamaan
  dan memeriksa invariant, bukan urutan
- `withTaskGroup` untuk membangkitkan konkurensi nyata di test

### 5. Menguji `@MainActor`
- `@Test @MainActor func ...`
- Kenapa mencampur isolation di test menghasilkan error yang membingungkan

### 6. Alat bantu
- Thread Sanitizer di scheme test
- `-warn-concurrency` untuk menemukan masalah sebelum runtime

## Cek pemahaman (draft)
1. Kenapa `Task.sleep` di dalam test adalah code smell?
2. Bagaimana kamu menguji bahwa request dibatalkan saat user mengetik lagi?
3. Bagaimana menguji bahwa cache actor tidak melakukan unduhan ganda?
