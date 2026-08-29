---
title: "Persistence: `UserDefaults`, Core Data, SwiftData, GRDB"
category: "Architecture"
status: draft
tags:
  - swift-mastering
  - ios/architecture
  - article
---

# Persistence: `UserDefaults`, Core Data, SwiftData, GRDB

> **Kategori:** Architecture · **Level:** Menengah · **Status:** 🚧 kerangka

## Kerangka isi

### 1. Matriks keputusan
| Pilihan | Untuk | Hindari kalau |
|---|---|---|
| `UserDefaults` | Preferensi, flag, data < ~100 KB | Data terstruktur atau perlu query |
| File + `Codable` | Data kecil-menengah, skema sederhana | Perlu query parsial, data besar |
| Keychain | Token, kredensial | Data non-rahasia (lambat) |
| GRDB (SQLite) | Kontrol penuh, SQL, migrasi eksplisit | Tim tidak nyaman SQL |
| Core Data | Graf objek + relasi, `NSFetchedResultsController`, CloudKit sync | Skema sederhana |
| SwiftData | Project SwiftUI baru | Migrasi rumit, target iOS lama |

### 2. Kriteria yang paling menentukan: **migrasi**
- Core Data lightweight migration: apa yang didukung dan apa yang gagal diam-diam
- GRDB memaksa migrasi eksplisit — lebih banyak kerja awal, jauh lebih sedikit kejutan
- `Codable` + file: kamu harus menangani versioning sendiri

### 3. Studi kasus: `LibraryStore` di project `movie`
```swift
private func persist() {
    guard let data = try? JSONEncoder().encode(savedTracks) else { return }
    UserDefaults.standard.set(data, forKey: defaultsKey)
}
```
- Kenapa ini **pilihan yang benar** untuk app ini (puluhan item, tanpa query)
- Batas yang harus disebutkan: seluruh array ditulis ulang setiap perubahan,
  tanpa query, tanpa migrasi, dan `try?` menelan error encoding
- Kapan ia mulai patah: ribuan item, atau saat butuh filter/sort di storage

### 4. Core Data yang perlu diketahui untuk interview
- `NSManagedObjectContext` per thread; `perform`/`performAndWait`
- Parent-child context; background context untuk import
- Faulting, dan kenapa `count` bisa memicu fetch
- `NSFetchedResultsController` + diffable data source

### 5. Concurrency & persistence
- Store sebagai `actor` atau `@MainActor` — kapan masing-masing
  ([lihat artikel actor](../concurrency/03-actor-isolation-reentrancy.md))
- Menghindari N kali lintasan boundary di dalam loop

## Cek pemahaman (draft)
1. Kapan `UserDefaults` + `Codable` cukup, dan apa tanda ia mulai patah?
2. Apa yang gagal diam-diam di Core Data lightweight migration?
3. Kenapa store yang dijadikan `actor` bisa memaksa refactor besar di ViewModel?
