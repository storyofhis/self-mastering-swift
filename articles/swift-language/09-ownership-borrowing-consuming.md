---
title: "Ownership: `borrowing`, `consuming`, dan Noncopyable Types"
category: "Swift Language & Internals"
status: draft
tags:
  - swift-mastering
  - swift/language
  - article
---

# Ownership: `borrowing`, `consuming`, dan Noncopyable Types

> **Kategori:** Swift Language & Internals · **Level:** Lanjut · **Status:** 🚧 kerangka

## Kenapa topik ini penting
Arah Swift ke depan untuk performa dan systems programming. Menguasainya menandakan
kamu mengikuti bahasa, bukan hanya framework.

## Kerangka isi

### 1. Masalah yang diselesaikan
- Retain/release traffic pada nilai yang sebenarnya cuma "dipinjam"
- Penyalinan yang tidak perlu pada nilai besar
- Resource yang tidak boleh disalin (file handle, lock token, buffer unik)

### 2. Convention parameter
| | Arti | Default untuk |
|---|---|---|
| `borrowing` | Pemanggil tetap memiliki; callee hanya membaca | parameter biasa |
| `consuming` | Kepemilikan berpindah; pemanggil tidak boleh memakai lagi | `__owned` initializer |
| `inout` | Akses eksklusif baca-tulis | `mutating self` |
- `consume x` operator dan `_ = consume x`

### 3. Noncopyable types: `~Copyable`
```swift
struct FileHandle: ~Copyable {
    let fd: Int32
    deinit { close(fd) }
}
```
- Kenapa `deinit` pada struct baru mungkin di sini
- Generic dengan `~Copyable` dan `borrowing`/`consuming`
- Hubungannya dengan `Span` dan `InlineArray` (Swift 6.2)

### 4. Exclusive access to memory
- Aturan Law of Exclusivity; static vs dynamic enforcement
- Kenapa `array[i] = f(&array)` bisa memicu "overlapping accesses"
- `-enforce-exclusivity=checked` di Release

### 5. Kapan ini relevan untuk app iOS biasa
Jujur: jarang. Tapi ia menjelaskan *kenapa* `isKnownUniquelyReferenced` butuh `inout`,
dan kenapa `Span` bisa aman tanpa overhead.

## Cek pemahaman (draft)
1. Kenapa `isKnownUniquelyReferenced` mengambil `inout`?
2. Apa bedanya `borrowing` dan `inout` dari sisi jaminan?
3. Kenapa tipe `~Copyable` bisa punya `deinit` sementara struct biasa tidak?

## Sumber untuk ditulis
- `swift/docs/OwnershipManifesto.md`, `swift/docs/ReferenceGuides/LifetimeAnnotation.md`
- [SE-0377 — borrowing and consuming parameters](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0377-parameter-ownership-modifiers.md)
- [SE-0390 — Noncopyable structs and enums](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md)
