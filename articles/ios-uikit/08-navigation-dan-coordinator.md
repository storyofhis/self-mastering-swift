---
title: "Navigation & Coordinator Pattern"
category: "iOS & UIKit"
status: draft
tags:
  - swift-mastering
  - ios/uikit
  - article
---

# Navigation & Coordinator Pattern

> **Kategori:** iOS & UIKit · **Level:** Menengah · **Status:** 🚧 kerangka

## Kerangka isi

### 1. Masalahnya
```swift
// HomeViewController — project movie
navigationController?.pushViewController(AlbumDetailViewController(album: album), animated: true)
```
View controller membangun view controller berikutnya **dan** memutuskan cara
menampilkannya. Untuk satu edge navigasi, itu keputusan yang benar —
`DECISIONS.md` mencatatnya sebagai gap yang disadari, bukan kelalaian.

### 2. Kapan biayanya mulai terasa
- Layar yang bisa dicapai dari beberapa tempat
- Deep link / push notification yang harus membuka layar di tengah stack
- A/B test alur
- Test yang ingin memverifikasi navigasi tanpa UI

### 3. Coordinator
```swift
protocol Coordinator: AnyObject {
    var childCoordinators: [Coordinator] { get set }
    func start()
}
```
- Kepemilikan: parent memiliki child; child **harus** dilepas saat selesai
  (sumber leak nomor satu di implementasi Coordinator)
- `UINavigationControllerDelegate` untuk mendeteksi pop dan membersihkan child

### 4. Alternatif
- Router (VIPER) — satu per modul
- Enum-based routing + `NavigationStack` (SwiftUI)
- Deep-link-first: setiap layar punya URL, navigasi = membuka URL

### 5. Cara memutuskan
Tulis biayanya, bukan preferensinya: berapa file tambahan, berapa layar,
berapa edge navigasi. Coordinator dibayar mulai sekitar 5+ layar dengan
percabangan, bukan sebelumnya.

## Cek pemahaman (draft)
1. Apa sumber memory leak paling umum di implementasi Coordinator?
2. Bagaimana deep link ke layar detail bekerja tanpa Coordinator?
3. Kapan kamu akan menolak menambahkan Coordinator?
