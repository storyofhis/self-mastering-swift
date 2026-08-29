---
title: "String Internals: Grapheme Cluster, UTF-8, dan Small String"
category: "Swift Language & Internals"
status: draft
tags:
  - swift-mastering
  - swift/language
  - article
---

# String Internals: Grapheme Cluster, UTF-8, dan Small String

> **Kategori:** Swift Language & Internals · **Level:** Lanjut · **Status:** 🚧 kerangka

## Kenapa topik ini penting
`String` adalah tipe yang paling sering dipakai dan paling sering disalahpahami.
Pertanyaan "kenapa `String` tidak bisa di-index dengan `Int`" adalah pintu masuk
ke pemahaman Unicode yang membedakan kandidat.

## Kerangka isi

### 1. `String` adalah koleksi `Character`, bukan koleksi byte
- `Character` = **extended grapheme cluster**, bukan code point
- `"é"` bisa 1 code point (U+00E9) atau 2 (U+0065 U+0301) — keduanya satu `Character`
- `"👨‍👩‍👧‍👦".count == 1` tapi `.unicodeScalars.count == 7`

### 2. Kenapa index bukan `Int`
- Grapheme cluster panjangnya variabel → `s[5]` tidak bisa O(1)
- `String.Index` menyimpan offset byte + informasi cache
- Konsekuensi: `s.index(s.startIndex, offsetBy: 5)` adalah O(n)
- Kalau butuh index numerik berulang, konversi ke `Array(s)` sekali

### 3. Representasi internal
- UTF-8 sebagai encoding native sejak Swift 5 (`_StringGuts`)
- **Small string optimization**: string ≤15 byte disimpan inline, nol alokasi
- Bridging lazy dengan `NSString` di platform Apple
- `String` 16 byte: dua word yang bisa berupa (small form) atau (pointer + flags)

### 4. Empat view
`characters` (default) · `unicodeScalars` · `utf8` · `utf16`
- Kapan pakai masing-masing; kenapa `utf16` ada (kompatibilitas NSString/Obj-C)

### 5. Normalisasi & perbandingan
- `==` membandingkan secara **canonical equivalence**, bukan byte
- Implikasi performa; kapan pakai `.compare(options:)`
- `localizedCaseInsensitiveCompare` untuk perbandingan yang ditampilkan ke user

### 6. Performa praktis
- `+=` dalam loop vs `reserveCapacity`
- `String.Index` yang di-cache vs dihitung ulang
- Kapan `StaticString` berguna

## Cek pemahaman (draft)
1. Kenapa `"👍🏽".count == 1`?
2. Berapa `MemoryLayout<String>.size`, dan apa isi dua word itu?
3. Kenapa `s[s.index(s.startIndex, offsetBy: n)]` dalam loop menghasilkan O(n²)?

## Sumber untuk ditulis
- `swift/stdlib/public/core/String*.swift`, `StringGuts.swift`, `SmallString.swift`
- [Swift.org — UTF-8 String](https://www.swift.org/blog/utf8-string/)
