---
title: "Launch Time & Scroll Performance"
category: "Performance"
status: draft
tags:
  - swift-mastering
  - ios/performance
  - article
---

# Launch Time & Scroll Performance

> **Kategori:** Performance · **Level:** Menengah–Lanjut · **Status:** 🚧 kerangka

## Kerangka isi

### 1. Anatomi launch
```
pre-main:  dyld → rebase/binding → ObjC runtime init → static initializer
post-main: didFinishLaunching → scene willConnectTo → first frame
```
- Target Apple: < 400 ms sampai frame pertama
- Mengukur: `DYLD_PRINT_STATISTICS`, Instruments App Launch, MetricKit di produksi

### 2. Memangkas pre-main
- Kurangi dynamic framework (setiap satu punya biaya tetap)
- Hindari `+load` dan static initializer yang berat
- Static linking kalau memungkinkan

### 3. Memangkas post-main
- Tunda apa pun yang tidak dibutuhkan untuk frame pertama (analytics, SDK, prefetch)
- Jangan membaca file besar/database secara sinkron di `didFinishLaunching`
- Layar pertama yang ringan; muat sisanya setelah `viewDidAppear`

### 4. Scroll performance
- Budget: 16,7 ms (60 Hz) / 8,3 ms (120 Hz) per frame
- Penyebab berurutan: kerja di main thread saat `cellForRow` → gambar tidak
  di-downsample → offscreen rendering → blending → Auto Layout dalam
- `shadowPath` wajib diset kalau memakai shadow
- `isOpaque = true` + background solid untuk menghilangkan blending

### 5. Memori
- Jetsam: app dibunuh saat memakai terlalu banyak memori
- `didReceiveMemoryWarning`, `NSCache` yang otomatis mengosongkan diri
- `autoreleasepool` di loop yang membuat banyak objek Obj-C

### 6. MetricKit
Mengukur di perangkat user, bukan di simulator: launch time, hang rate, hitch ratio,
memory. Ini yang membuat perbaikan bisa dibuktikan, bukan dirasakan.

## Cek pemahaman (draft)
1. Apa bedanya pre-main dan post-main time, dan mana yang bisa kamu kendalikan?
2. Kenapa shadow tanpa `shadowPath` mahal?
3. Bagaimana kamu membuktikan launch time membaik untuk user, bukan cuma di mesinmu?
