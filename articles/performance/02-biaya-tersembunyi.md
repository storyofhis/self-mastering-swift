---
title: "Biaya Tersembunyi: Retain/Release Traffic, Bridging, Dynamic Dispatch"
category: "Performance"
status: draft
tags:
  - swift-mastering
  - ios/performance
  - article
---

# Biaya Tersembunyi: Retain/Release Traffic, Bridging, Dynamic Dispatch

> **Kategori:** Performance · **Level:** Lanjut · **Status:** 🚧 kerangka

## Kerangka isi

### 1. Retain/release traffic
- Setiap operasi adalah **atomik** → cache coherency antar core
- Muncul di: loop yang mengoper objek class, existential besar, bridging
- Cara mengurangi: value type di hot path, `final`, Whole Module Optimization

### 2. Empat level dispatch di Swift
| | Kapan | Biaya |
|---|---|---|
| Static (inline) | `final`, `private`, `struct`, WMO | ~0 |
| Vtable | method class yang bisa di-override | 1 indirect |
| Witness table | protokol | 1 indirect + kemungkinan existential |
| `objc_msgSend` | `@objc dynamic`, subclass NSObject | paling mahal |
- Kenapa `final` dan `private` benar-benar mengubah kode yang dihasilkan

### 3. Existential & boxing
- Batas 24 byte; di atasnya alokasi heap per pembungkusan
- `[any P]` vs `[ConcreteType]` — ukur bedanya

### 4. Bridging Swift ↔ Objective-C
- `String` ↔ `NSString`, `Array` ↔ `NSArray`: kapan gratis, kapan menyalin
- `as` bridging di dalam loop = biaya tersembunyi terbesar di kode campuran

### 5. ARC yang tidak perlu
- `withExtendedLifetime`, `Unmanaged` (hanya untuk kasus ekstrem yang sudah diprofil)
- Kenapa `unowned(unsafe)` hampir tidak pernah layak

### 6. Alokasi
- Heap allocation ~100× lebih mahal dari stack
- `reserveCapacity`, `ContiguousArray`, `InlineArray` (Swift 6.2)

### 7. Urutan optimasi yang benar
Algoritma → alokasi → dispatch → micro. Kebanyakan orang mulai dari yang terakhir.

## Cek pemahaman (draft)
1. Kenapa `final` bisa mengubah performa, bukan cuma gaya?
2. Kapan `String` ↔ `NSString` bridging gratis?
3. Kenapa `[any Shape]` bisa jauh lebih lambat dari `[Circle]`?
