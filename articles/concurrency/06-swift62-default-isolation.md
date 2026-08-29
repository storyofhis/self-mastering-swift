---
title: "Swift 6.2: Default Actor Isolation, `nonisolated(nonsending)`, `@concurrent`"
category: "Concurrency"
status: draft
tags:
  - swift-mastering
  - swift/concurrency
  - article
---

# Swift 6.2: Default Actor Isolation, `nonisolated(nonsending)`, `@concurrent`

> **Kategori:** Concurrency · **Level:** Lanjut · **Status:** 🚧 kerangka

## Kenapa topik ini penting
Ini perubahan yang membuat Swift 6 layak diadopsi untuk app UI. Kalau kamu bisa
membicarakannya, kamu terdengar seperti orang yang mengikuti bahasa secara aktif.

## Kerangka isi

### 1. Masalah sebelum 6.2
- App UI pada dasarnya single-threaded, tapi Swift 6 memaksa menandai `@MainActor`
  di mana-mana → anotasi yang bising dan mudah lupa
- Fungsi `async` nonisolated diam-diam melompat ke global executor, membuat kode
  yang tampak sekuensial ternyata berpindah thread

### 2. Default actor isolation (`-default-isolation MainActor`)
- Seluruh module default `@MainActor`; kamu menandai yang **keluar**, bukan yang masuk
- Cara mengaktifkan di Xcode build settings dan di `Package.swift`
- Kapan ini pilihan yang benar (app UI) dan kapan tidak (library, package non-UI)

### 3. `nonisolated(nonsending)`
- Fungsi async nonisolated berjalan di execution context **pemanggil**
- Menghapus hop yang tidak diinginkan; membuat `await` tidak lagi berarti
  "pasti pindah thread"
- Upcoming feature flag terkait

### 4. `@concurrent`
- Menyatakan eksplisit bahwa fungsi ini memang harus di executor konkuren
- Pasangan yang benar: default main-actor + `@concurrent` untuk pekerjaan berat

### 5. Fitur pendamping Swift 6.2
- `InlineArray` — array ukuran tetap, storage inline, tanpa alokasi heap
- `Span` — akses aman ke memori kontigu tanpa overhead runtime
- Strict memory safety (opt-in) — menandai konstruksi `unsafe`
- Migration tooling resmi

### 6. Swift 6.4 (WWDC26) untuk concurrency
- `weak let` → class dengan weak reference immutable bisa `Sendable` tanpa `@unchecked`
- `~Sendable` untuk menolak conformance secara eksplisit
- `await` di dalam `defer`
- Peringatan untuk error yang tidak ditangani di closure `Task`

### 7. Rencana migrasi konkret untuk codebase yang ada
1. `.enableUpcomingFeature("StrictConcurrency")` → warning dulu
2. Model layer → service → view model → view
3. `@MainActor` pada layer UI
4. `@preconcurrency import` untuk SDK pihak ketiga
5. Audit setiap `@unchecked Sendable` yang tersisa

## Sumber untuk ditulis
- [Swift 6.2 released — Swift.org](https://www.swift.org/blog/swift-6.2-released/)
- [SE-0466 — Control default actor isolation inference](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md)
- `swift/userdocs/diagnostics/upcoming-language-features.md`
