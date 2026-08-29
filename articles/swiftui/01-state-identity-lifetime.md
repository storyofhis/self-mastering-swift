---
title: "State, Identity, dan Lifetime: Kenapa View Kamu Tidak Ter-update"
category: "SwiftUI"
status: draft
tags:
  - swift-mastering
  - ios/swiftui
  - article
---

# State, Identity, dan Lifetime: Kenapa View Kamu Tidak Ter-update

> **Kategori:** SwiftUI · **Level:** Menengah · **Status:** 🚧 kerangka

## Kerangka isi

### 1. View adalah deskripsi, bukan objek
- `struct View` dibuat ulang terus-menerus; `body` bisa dipanggil ratusan kali
- Yang persisten bukan view-nya, tapi **storage** yang dikelola SwiftUI
- Karena itu `@State` menyimpan nilai di luar struct

### 2. Identity: bagaimana SwiftUI tahu "view yang sama"
- **Structural identity** — posisi di view tree
- **Explicit identity** — `.id()`, `ForEach(id:)`
- `if`/`else` menghasilkan `_ConditionalContent` → dua cabang = dua identitas berbeda
  → state hilang saat berpindah cabang
- Solusi: satu view dengan modifier kondisional, bukan dua view berbeda

### 3. Property wrapper dan siapa yang memiliki apa
| | Kepemilikan | Pakai untuk |
|---|---|---|
| `@State` | View memiliki nilainya | State lokal, value type |
| `@Binding` | Referensi ke state milik orang lain | Kontrol anak |
| `@StateObject` | View **membuat** dan memiliki objek | Sekali per identity |
| `@ObservedObject` | View menerima objek dari luar | Objek yang dimiliki parent |
| `@Environment` | Dibaca dari environment | Dependency implisit |

- Bug klasik: `@ObservedObject` di tempat yang seharusnya `@StateObject` → objek
  dibuat ulang setiap kali body parent dievaluasi

### 4. Lifetime
- Kapan `@State` di-reset (identity berubah)
- `onAppear`/`onDisappear` bukan lifecycle yang bisa diandalkan seperti UIKit
- `task(id:)` untuk pekerjaan async yang terikat identity

### 5. Kenapa view tidak ter-update — daftar penyebab berurutan
1. Objeknya class tanpa `@Observable`/`ObservableObject`
2. Mutasi terjadi di luar jalur yang diamati (mis. property nested)
3. `@ObservedObject` yang seharusnya `@StateObject`
4. Identity berubah sehingga state di-reset (bukan "tidak update", tapi "reset")
5. `Equatable` view yang menolak re-render

## Cek pemahaman (draft)
1. Kenapa `if isEditing { TextField(...) } else { TextField(...) }` kehilangan fokus?
2. Kapan `@StateObject` dan kapan `@ObservedObject`?
3. Apa yang terjadi pada `@State` ketika `.id()` berubah?
