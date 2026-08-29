---
title: "Instruments: Time Profiler, Allocations, Leaks"
category: "Performance"
status: draft
tags:
  - swift-mastering
  - ios/performance
  - article
---

# Instruments: Time Profiler, Allocations, Leaks

> **Kategori:** Performance · **Level:** Menengah–Lanjut · **Status:** 🚧 kerangka

## Prinsip yang lebih penting dari alatnya
**Ukur dulu.** Setiap optimasi tanpa pengukuran adalah tebakan yang membuat kode
lebih sulit dibaca dengan probabilitas 50% memperlambatnya.

## Kerangka isi

### 1. Time Profiler
- Sampling call stack, bukan instrumentasi
- Membaca call tree: "Separate by Thread", "Invert Call Tree", "Hide System Libraries"
- Yang dicari: main thread yang sibuk saat scroll/launch
- Tanda bahaya: `swift_retain`/`swift_release` di 10 besar → retain traffic
- **Selalu profil build Release** — Debug tidak punya spesialisasi generic

### 2. Allocations
- "Persistent" vs "Transient"
- **Mark Generation**: buka layar → tutup → mark → ulang 5×.
  Kalau Persistent naik terus, ada abandoned memory
- Ini satu-satunya cara menemukan leak yang bukan siklus

### 3. Leaks
- Hanya menemukan memori yang **tak terjangkau**
- Tidak akan menemukan cache yang tumbuh terus — itu tugas Allocations

### 4. Core Animation / Animation Hitches
- "Color Blended Layers" (merah = blending), "Color Offscreen-Rendered Yellow"
- Hitch = frame yang terlewat; budget 16,7 ms (60 Hz) atau 8,3 ms (120 Hz)

### 5. App Launch
- Pre-main vs post-main time
- Dynamic library loading, `+load`/static initializer, `didFinishLaunching` yang berat

### 6. os_signpost untuk mengukur kode sendiri
```swift
let log = OSLog(subsystem: "app", category: .pointsOfInterest)
os_signpost(.begin, log: log, name: "decode")
```

### 7. Alur diagnosis scroll lag (urutan tersangka)
1. Kerja di main thread saat `cellForRow`
2. Gambar tidak di-downsample
3. Offscreen rendering (shadow tanpa `shadowPath`)
4. Blended layers
5. Auto Layout yang terlalu dalam
6. `.estimated` yang jauh meleset

## Cek pemahaman (draft)
1. Kenapa profiling di Debug build menyesatkan?
2. Alat mana untuk leak, mana untuk abandoned memory?
3. Berapa budget waktu per frame di layar 120 Hz?
