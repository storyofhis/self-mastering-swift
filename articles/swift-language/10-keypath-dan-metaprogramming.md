---
title: "Key Paths, Dynamic Member Lookup, dan Refleksi"
category: "Swift Language & Internals"
status: draft
tags:
  - swift-mastering
  - swift/language
  - article
---

# Key Paths, Dynamic Member Lookup, dan Refleksi

> **Kategori:** Swift Language & Internals · **Level:** Menengah–Lanjut · **Status:** 🚧 kerangka

## Kerangka isi

### 1. Key path sebagai nilai
- `KeyPath` / `WritableKeyPath` / `ReferenceWritableKeyPath` / `PartialKeyPath` / `AnyKeyPath`
- `\Track.title` — apa yang sebenarnya dibuat compiler
- Representasi runtime: lihat `swift/docs/ABI/KeyPaths.md`
- Key path sebagai fungsi: `tracks.map(\.title)`

### 2. Pemakaian praktis
- Sorting generic: `sorted(by: \.duration)`
- Type-safe observation (KVO Swift, `Observable`)
- Dependency injection berbasis key path (`@Environment` di SwiftUI)

### 3. `@dynamicMemberLookup`
- Kapan sah dipakai (proxy, wrapper, interop dengan bahasa dinamis)
- Kapan ia merusak type safety dan autocomplete

### 4. Refleksi: `Mirror`
- Apa yang bisa dan tidak bisa dilakukan
- Kenapa ia lambat dan tidak boleh ada di hot path
- Alternatif modern: macro (Swift 5.9+) yang menghasilkan kode saat compile

### 5. Macro sebagai pengganti metaprogramming runtime
- `@attached(member)`, `@freestanding(expression)`
- Contoh: `@Observable` adalah macro, bukan sihir compiler

## Cek pemahaman (draft)
1. Apa bedanya `KeyPath` dan `ReferenceWritableKeyPath`?
2. Kenapa `Mirror` tidak cocok untuk serialisasi produksi?
3. Bagaimana `@Observable` bekerja di balik layar?

## Sumber untuk ditulis
- `swift/docs/ABI/KeyPaths.md`
- [SE-0161 — Smart KeyPaths](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0161-key-paths.md)
