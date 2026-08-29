---
title: "Cell Reuse, Image Loading, dan Bug Gambar Ketuker"
category: "iOS & UIKit"
status: draft
tags:
  - swift-mastering
  - ios/uikit
  - article
---

# Cell Reuse, Image Loading, dan Bug Gambar Ketuker

> **Kategori:** iOS & UIKit · **Level:** Menengah · **Status:** 🚧 kerangka

## Kerangka isi

### 1. Mekanisme reuse
- Hanya cell yang terlihat (+buffer) yang ada; sisanya di reuse queue
- `dequeueReusableCell` mengembalikan cell dengan **isi lama masih menempel**
- `prepareForReuse` dan daftar lengkap yang harus dibersihkan

### 2. Bug gambar ketuker: dua perbaikan yang harus dipakai bersama
1. Batalkan request di `prepareForReuse`
2. Verifikasi identitas saat hasil datang (`guard self.trackID == requestedID`)
- Kenapa hanya nomor 1 saja tidak cukup (pembatalan bisa terlambat)
- Contoh dari `TrackCell` di project `movie` (`imageTask: URLSessionDataTask?`)

### 3. Downsampling — penyebab scroll lag nomor satu
```swift
CGImageSourceCreateThumbnailAtIndex(src, 0, [
    kCGImageSourceCreateThumbnailFromImageAlways: true,
    kCGImageSourceThumbnailMaxPixelSize: maxDimension,
    kCGImageSourceShouldCacheImmediately: true
] as CFDictionary)
```
- Kenapa `UIImage(data:)` menunda decoding sampai render (di main thread!)
- `kCGImageSourceShouldCacheImmediately` memindahkan decoding ke background

### 4. Caching dua lapis
- `NSCache<NSURL, UIImage>` untuk gambar yang sudah di-decode (otomatis dibuang
  saat memory pressure)
- `URLCache` untuk data mentah lewat header HTTP
- Kenapa `NSCache` bukan `Dictionary`

### 5. Deduplikasi request in-flight
- Dua cell meminta URL sama → satu unduhan
- Simpan `Task` di actor; tulis entri `.inProgress` **sebelum** `await` pertama
- Hubungkan ke [reentrancy actor](../concurrency/03-actor-isolation-reentrancy.md)

### 6. Prefetching
- `UICollectionViewDataSourcePrefetching`
- `cancelPrefetchingForItemsAt` — sering dilupakan

## Cek pemahaman (draft)
1. Kenapa membatalkan request di `prepareForReuse` saja tidak cukup?
2. Kenapa `UIImage(data:)` bisa menyebabkan hitch meski decoding di background?
3. Apa keunggulan `NSCache` dibanding `Dictionary` untuk cache gambar?
