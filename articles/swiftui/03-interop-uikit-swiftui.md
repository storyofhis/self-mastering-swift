---
title: "Interop UIKit ↔ SwiftUI"
category: "SwiftUI"
status: draft
tags:
  - swift-mastering
  - ios/swiftui
  - article
---

# Interop UIKit ↔ SwiftUI

> **Kategori:** SwiftUI · **Level:** Menengah · **Status:** 🚧 kerangka

## Kerangka isi

### 1. SwiftUI di dalam UIKit: `UIHostingController`
- Sizing: `sizingOptions`, `intrinsicContentSize` (iOS 16+)
- Menaruhnya sebagai child view controller dengan urutan `addChild`/`didMove` yang benar
- Kasus nyata: mengganti satu layar di app UIKit tanpa menulis ulang navigasi

### 2. UIKit di dalam SwiftUI: `UIViewRepresentable` / `UIViewControllerRepresentable`
```swift
func makeUIView(context:) -> UIView
func updateUIView(_:context:)
func makeCoordinator() -> Coordinator     // untuk delegate & target-action
static func dismantleUIView(_:coordinator:)
```
- `Coordinator` sebagai jembatan delegate → binding
- Jebakan: `updateUIView` dipanggil sangat sering; harus idempoten dan murah

### 3. Menjembatani state
- `@Binding` ke `Coordinator`
- `ObservableObject`/`@Observable` yang dibagi dua dunia
- Kenapa mengirim closure ke `makeUIView` biasanya bug (closure lama tertahan)

### 4. Navigasi campuran
- `NavigationStack` yang mendorong `UIHostingController`
- `UINavigationController` yang mendorong SwiftUI view
- Masalah back-button, gesture, dan judul yang hilang

### 5. Kapan memilih mana untuk migrasi bertahap
- Layar-per-layar (hosting controller) vs komponen-per-komponen (representable)
- Rekomendasi: mulai dari layar daun yang sederhana

## Cek pemahaman (draft)
1. Kenapa `updateUIView` harus idempoten?
2. Kapan `Coordinator` diperlukan?
3. Apa masalah paling umum saat mencampur `NavigationStack` dan `UINavigationController`?
