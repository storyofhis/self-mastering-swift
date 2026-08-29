---
title: "iOS Quick Reference"
category: "Cheat Sheet"
status: draft
tags:
  - swift-mastering
  - cheatsheet
---

# iOS Quick Reference

> **Status:** 🚧 kerangka — daftar isi sudah ada, tabel detail belum diisi penuh.
> Sebagian besar materi UIKit sudah ada di [swift-quick-reference.md](swift-quick-reference.md#uikit).

## Yang akan diisi

### Lifecycle
- [ ] Tabel `UIApplicationDelegate` vs `UISceneDelegate` lengkap
- [ ] Urutan lifecycle view controller + apa yang benar di masing-masing
- [ ] Background execution: batas waktu, jenis task, kapan sistem membunuh app

### Layout
- [x] `frame` vs `bounds` — ada di swift-quick-reference
- [x] `setNeedsLayout` vs `layoutIfNeeded` — ada di swift-quick-reference
- [ ] Nilai prioritas constraint dan artinya
- [ ] Layout guide: `safeArea`, `readableContent`, `layoutMargins`

### Collection & table
- [x] Aturan diffable data source — ada di swift-quick-reference
- [ ] Checklist `prepareForReuse` lengkap
- [ ] Compositional layout: cheat sheet dimensi & `orthogonalScrollingBehavior`

### Networking
- [x] `URLSession` tidak melempar untuk 4xx/5xx
- [ ] Konfigurasi default yang layak diubah (timeout, cache policy, connections per host)
- [ ] Kode error `URLError` yang sering ditemui

### Persistence
- [ ] Matriks pilihan storage
- [ ] Core Data: context rules, `perform` vs `performAndWait`

### Performance
- [ ] Budget frame, target launch time
- [ ] Checklist offscreen rendering & blending
- [ ] Alat Instruments mana untuk gejala apa

### Debugging
- [ ] Memory Graph Debugger — cara membaca
- [ ] Breakpoint simbolik yang berguna (`UIViewAlertForUnsatisfiableConstraints`,
      `-[UIView setNeedsLayout]`)
- [ ] Environment variable: `DYLD_PRINT_STATISTICS`, `UIViewShowAlignmentRects`
