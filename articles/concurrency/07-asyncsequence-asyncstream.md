---
title: "`AsyncSequence` & `AsyncStream`: Bridging Dunia Callback"
category: "Concurrency"
status: draft
tags:
  - swift-mastering
  - swift/concurrency
  - article
---

# `AsyncSequence` & `AsyncStream`: Bridging Dunia Callback

> **Kategori:** Concurrency · **Level:** Menengah–Lanjut · **Status:** 🚧 kerangka

## Kenapa topik ini penting
Continuation menangani "satu nilai". Untuk sumber yang menghasilkan **banyak** nilai —
delegate lokasi, socket, notification, input teks — `AsyncStream` adalah jawabannya,
dan memakai continuation di sana adalah bug menunggu terjadi.

## Kerangka isi

### 1. `AsyncSequence` sebagai protokol
- `AsyncIteratorProtocol.next() async throws -> Element?`
- `for await item in seq` dan bagaimana compiler men-desugar-nya
- Operator: `map`, `filter`, `compactMap`, `prefix`, `dropFirst` — semua lazy

### 2. `AsyncStream` vs `AsyncThrowingStream`
```swift
let stream = AsyncStream<CLLocation> { continuation in
    let delegate = LocationDelegate { continuation.yield($0) }
    continuation.onTermination = { _ in manager.stopUpdatingLocation() }
    manager.delegate = delegate
    manager.startUpdatingLocation()
}
```
- `yield`, `finish`, `onTermination`
- **Buffering policy**: `.unbounded` (default, bahaya memory), `.bufferingNewest(n)`,
  `.bufferingOldest(n)` — jelaskan kapan masing-masing benar
- `AsyncStream.makeStream()` (Swift 5.9+) untuk continuation yang perlu disimpan

### 3. Kapan pakai stream, kapan Combine, kapan closure
| | Pakai kalau |
|---|---|
| Closure | Satu pendengar, event sederhana |
| `AsyncStream` | Banyak nilai, konsumsi dengan `for await`, butuh cancellation |
| Combine | Butuh operator komposisi kaya, multicast, backpressure |
| `@Observable` | State UI yang diamati SwiftUI |

### 4. Jebakan
- **Satu iterator**: `AsyncStream` bukan multicast — dua `for await` akan berebut nilai
- Stream yang tidak pernah `finish()` menahan task selamanya
- Buffer `.unbounded` pada sumber cepat = memory growth tak terbatas
- Membatalkan task tidak otomatis menghentikan sumbernya — itu tugas `onTermination`

### 5. Contoh migrasi: debounce search
Ganti `DispatchWorkItem` + `Task` di `SearchViewModel` dengan satu stream teks yang
di-debounce, sehingga hanya ada satu mekanisme pembatalan.

## Cek pemahaman (draft)
1. Kenapa memakai `withCheckedContinuation` untuk delegate lokasi menyebabkan crash?
2. Apa yang terjadi kalau dua task melakukan `for await` pada `AsyncStream` yang sama?
3. Kapan `.bufferingNewest(1)` adalah pilihan yang benar?

## Sumber untuk ditulis
- `swift/stdlib/public/Concurrency/AsyncStream.swift`, `AsyncSequence.swift`
- [SE-0298 — AsyncSequence](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0298-asyncsequence.md)
