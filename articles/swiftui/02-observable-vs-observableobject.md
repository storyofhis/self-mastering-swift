---
title: "`@Observable` vs `ObservableObject`: Apa yang Berubah di Observation"
category: "SwiftUI"
status: draft
tags:
  - swift-mastering
  - ios/swiftui
  - article
---

# `@Observable` vs `ObservableObject`: Apa yang Berubah di Observation

> **Kategori:** SwiftUI · **Level:** Menengah–Lanjut · **Status:** 🚧 kerangka

## Kerangka isi

### 1. Masalah `ObservableObject`
- `@Published` memberi tahu **seluruh objek berubah**, bukan property mana
- Setiap view yang mengamati objek itu di-render ulang, meski hanya memakai satu field
- `objectWillChange` dikirim **sebelum** perubahan → satu frame delay

### 2. `@Observable` (Observation framework, iOS 17+)
- Macro yang menghasilkan akses terlacak per-property
- SwiftUI hanya me-render ulang view yang benar-benar **membaca** property yang berubah
- Tidak butuh `@Published`; tidak butuh `@StateObject` (cukup `@State`)
- Implementasinya di `swift/stdlib/public/Observation/`

### 3. Perbandingan konkret
| | `ObservableObject` | `@Observable` |
|---|---|---|
| Granularitas | seluruh objek | per property yang dibaca |
| Wrapper di view | `@StateObject` / `@ObservedObject` | `@State` / biasa |
| Optional & array | canggung | natural |
| Minimum iOS | 13 | 17 |

### 4. Migrasi
- Langkah-langkah, dan apa yang rusak (mis. `Combine` publisher dari `@Published` hilang)
- `@Bindable` untuk binding ke objek `@Observable`

### 5. Cara kerja di balik layar
- `withObservationTracking(_:onChange:)`
- Bagaimana SwiftUI memakainya untuk menghitung dependency graph per view

## Cek pemahaman (draft)
1. Kenapa `@Observable` mengurangi re-render dibanding `ObservableObject`?
2. Kenapa `@Observable` tidak butuh `@StateObject`?
3. Apa yang kamu kehilangan saat bermigrasi dari `@Published`?
