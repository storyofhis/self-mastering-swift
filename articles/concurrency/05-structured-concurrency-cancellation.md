---
title: "Structured Concurrency: `async let`, `TaskGroup`, dan Cancellation yang Benar"
category: "Concurrency"
status: draft
tags:
  - swift-mastering
  - swift/concurrency
  - article
---

# Structured Concurrency: `async let`, `TaskGroup`, dan Cancellation yang Benar

> **Kategori:** Concurrency · **Level:** Menengah–Lanjut · **Status:** 🚧 kerangka

## Kenapa topik ini penting
Ini bagian yang paling sering dipakai sehari-hari dan paling sering ditanyakan dalam
bentuk soal desain ("unduh 500 gambar").

## Kerangka isi

### 1. Apa yang membuat concurrency "terstruktur"
- Setiap child task punya induk; induk tidak selesai sebelum semua anak selesai
- Pembatalan mengalir ke bawah secara otomatis
- Error dari satu anak membatalkan saudaranya
- Bandingkan dengan `Task {}` yang **tidak** terstruktur

### 2. `async let`
- Child task mulai saat **dideklarasikan**, bukan saat di-`await`
- Jumlah tetap dan diketahui saat compile
- Contoh dari project `movie`: `HomeViewModel.load()` dengan dua shelf paralel
- Jebakan: `async let` yang tidak pernah di-`await` → dibatalkan implisit di akhir scope

### 3. `withTaskGroup` / `withThrowingTaskGroup`
- Jumlah dinamis, hasil dikumpulkan lewat `for await`
- `group.next()` vs `for try await result in group`
- `withDiscardingTaskGroup` untuk pekerjaan tanpa hasil (Swift 5.9+)

### 4. Bounded concurrency — pola yang harus dihafal
```swift
for _ in 0..<maxConcurrent { addTaskIfAvailable() }
while let result = try await group.next() {
    collect(result)
    addTaskIfAvailable()          // isi ulang satu setiap kali satu selesai
}
```
Sebutkan angkanya: 4–6 untuk HTTP (sesuai `httpMaximumConnectionsPerHost`).

### 5. Cancellation secara mendalam
- Kooperatif: `cancel()` hanya menyalakan flag
- `Task.isCancelled` vs `try Task.checkCancellation()`
- **Cek di blok `catch`** — `URLError.cancelled` yang tidak difilter menyebabkan
  pesan error berkedip di UI
- `withTaskCancellationHandler` untuk membersihkan resource non-Swift
- Cancellation dan `defer`; Swift 6.4 mengizinkan `await` di `defer`

### 6. Task priority & priority escalation
- Kenapa `Task(priority:)` bukan jaminan
- Priority inversion dan bagaimana runtime menaikkan prioritas task yang ditunggu

## Cek pemahaman (draft)
1. Kapan `f()` mulai jalan di `async let x = f()`?
2. Kenapa `for url in urls { Task { ... } }` bukan solusi yang baik?
3. Apa yang terjadi pada child task kalau induknya melempar?

## Sumber untuk ditulis
- [SE-0304 — Structured concurrency](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)
- `swift/stdlib/public/Concurrency/TaskGroup.swift`, `AsyncLet.swift`
- Project `movie`: `HomeViewModel.load`, `SearchViewModel.performSearch`
