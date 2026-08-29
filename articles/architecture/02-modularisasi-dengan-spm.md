---
title: "Modularisasi dengan Swift Package Manager"
category: "Architecture"
status: draft
tags:
  - swift-mastering
  - ios/architecture
  - article
---

# Modularisasi dengan Swift Package Manager

> **Kategori:** Architecture · **Level:** Menengah–Lanjut · **Status:** 🚧 kerangka

## Kerangka isi

### 1. Kenapa memodulkan
- Waktu build (incremental build hanya menyentuh modul yang berubah)
- Batas yang ditegakkan **compiler**, bukan konvensi (`internal` jadi benar-benar berarti)
- Tim bisa bekerja paralel; preview/test per modul lebih cepat

### 2. Contoh nyata di mesin ini
`UIKitBuilder` di project `movie` adalah local Swift package dengan `Package.swift`,
`Sources/`, dan `Tests/` — contoh terkecil dari modularisasi yang benar:
sebuah utility (builder pattern untuk `UILabel`, `UIButton`,
`NSCollectionLayoutSection`) yang tidak punya alasan hidup di dalam target app.

### 3. Struktur yang bekerja
```
App                      (target aplikasi, tipis)
 ├─ Features/Home        (satu package per fitur)
 ├─ Features/Search
 ├─ Core/Networking      (tidak tahu apa-apa soal UI)
 ├─ Core/DesignSystem
 └─ Core/Models          (dependency paling bawah, tidak bergantung apa pun)
```
Aturan: dependency hanya boleh mengarah ke bawah.

### 4. `Package.swift` yang perlu diketahui
- `products` (library) vs `targets`
- `swiftSettings: [.enableUpcomingFeature("StrictConcurrency")]` per target →
  cara memigrasikan Swift 6 secara bertahap
- Resource, `.copy` vs `.process`
- Test target dan `@testable import`

### 5. Jebakan
- Circular dependency (compiler menolak — dan itu bagus)
- Modul yang terlalu kecil = overhead build lebih besar dari manfaatnya
- `public` di mana-mana → batasnya hilang lagi
- Preview dan resource yang tidak ketemu karena `Bundle.module`

### 6. Kapan TIDAK memodulkan
App di bawah ~20 layar dengan satu developer: biaya `public` boilerplate dan
konfigurasi lebih besar dari manfaatnya.

## Sumber untuk ditulis
- Project `movie`: `UIKitBuilder/Package.swift`, `Sources/`, `Tests/`
