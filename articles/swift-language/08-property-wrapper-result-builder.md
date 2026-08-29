---
title: "Property Wrapper & Result Builder: Sugar yang Di-desugar Compiler"
category: "Swift Language & Internals"
status: draft
tags:
  - swift-mastering
  - swift/language
  - article
---

# Property Wrapper & Result Builder: Sugar yang Di-desugar Compiler

> **Kategori:** Swift Language & Internals · **Level:** Lanjut · **Status:** 🚧 kerangka

## Kenapa topik ini penting
SwiftUI seluruhnya dibangun di atas dua fitur ini. Memahaminya mengubah SwiftUI
dari "sihir yang kadang tidak jalan" jadi mekanisme yang bisa kamu prediksi.

## Kerangka isi

### 1. Property wrapper: apa yang dihasilkan compiler
```swift
@Clamped(0...100) var progress = 50
// menjadi:
private var _progress = Clamped(wrappedValue: 50, 0...100)
var progress: Int { get { _progress.wrappedValue } set { _progress.wrappedValue = newValue } }
```
- `wrappedValue`, `projectedValue` (`$`), `init(wrappedValue:)`
- Kenapa `@State` butuh `projectedValue` (Binding)
- Batasan: tidak bisa di variabel lokal `let`, tidak bisa di protokol requirement

### 2. Menulis property wrapper sendiri
- Contoh: `@UserDefault`, `@Clamped`, `@Trimmed`
- Jebakan: wrapper yang menyimpan state di struct — kapan ia disalin?

### 3. Result builder
```swift
@resultBuilder struct ViewBuilder {
    static func buildBlock(_ parts: View...) -> View
    static func buildOptional / buildEither / buildArray / buildExpression / buildFinalResult
}
```
- Transformasi `if`/`switch`/`for` menjadi panggilan `buildOptional`/`buildEither`/`buildArray`
- Kenapa `if` di SwiftUI mengubah **identitas** view (→ hubungkan ke artikel SwiftUI state)
- Kenapa `ViewBuilder` dulu terbatas 10 subview (varian generic) dan bagaimana
  parameter pack (SE-0393) menghapus batas itu

### 4. Kapan TIDAK memakai keduanya
- Property wrapper yang menyembunyikan efek samping = kode yang tidak bisa dibaca
- Result builder untuk DSL yang jarang dipakai = biaya belajar > manfaat

## Cek pemahaman (draft)
1. Apa yang dihasilkan compiler untuk `@Published var x = 0`?
2. Kenapa `@State` tidak boleh dipakai di luar `View`?
3. Kenapa `if` di dalam `ViewBuilder` bisa membuat state view hilang?

## Sumber untuk ditulis
- [SE-0258 — Property Wrappers](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md)
- [SE-0289 — Result builders](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0289-result-builders.md)
