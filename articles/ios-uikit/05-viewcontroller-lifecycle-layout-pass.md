---
title: "View Controller Lifecycle & Layout Pass"
category: "iOS & UIKit"
status: draft
tags:
  - swift-mastering
  - ios/uikit
  - article
---

# View Controller Lifecycle & Layout Pass

> **Kategori:** iOS & UIKit · **Level:** Menengah · **Status:** 🚧 kerangka

## Kerangka isi

### 1. Lifecycle lengkap dan apa yang benar di masing-masing
(tabel: `loadView` → `viewDidLoad` → `viewWillAppear` → `viewWillLayoutSubviews` →
`viewDidLayoutSubviews` → `viewDidAppear` → `viewWillDisappear` → `viewDidDisappear` → `deinit`)
- Aturan: setup sekali → `viewDidLoad`; refresh → `viewWillAppear`;
  apa pun yang bergantung `bounds` → `viewDidLayoutSubviews`
- Contoh nyata: `LibraryViewController` di project `movie` memuat ulang dari
  `LibraryStore` di `viewWillAppear` — dan `DECISIONS.md` menjelaskan kenapa itu
  cukup menggantikan Observer

### 2. Layout pass yang sebenarnya
```
update constraints  →  layout (layoutSubviews)  →  display (draw)
```
- `setNeedsUpdateConstraints` / `updateConstraintsIfNeeded`
- `setNeedsLayout` (asinkron) / `layoutIfNeeded` (sinkron)
- `setNeedsDisplay` / `draw(_:)`
- Kenapa `layoutIfNeeded()` wajib di dalam blok animasi constraint

### 3. Intrinsic content size & `invalidateIntrinsicContentSize`
- Kapan custom view perlu meng-override `intrinsicContentSize`
- Hubungannya dengan hugging & compression resistance

### 4. Container view controller
- `addChild` / `didMove(toParent:)` / `willMove(toParent:nil)` / `removeFromParent`
- Urutan yang benar dan apa yang bocor kalau salah

### 5. Trait collection & size class
- `traitCollectionDidChange` (dan penggantinya di iOS 17+: `registerForTraitChanges`)
- Kenapa mengecek `UIDevice.userInterfaceIdiom` adalah anti-pattern

## Cek pemahaman (draft)
1. Kenapa `layer.cornerRadius = bounds.width/2` di `viewDidLoad` menghasilkan lingkaran salah?
2. Berapa kali `viewDidLayoutSubviews` dipanggil dalam satu sesi layar?
3. Apa urutan benar melepas child view controller?
