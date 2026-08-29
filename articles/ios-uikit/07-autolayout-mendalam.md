---
title: "Auto Layout: Constraint Priority, Hugging, dan Compression Resistance"
category: "iOS & UIKit"
status: draft
tags:
  - swift-mastering
  - ios/uikit
  - article
---

# Auto Layout: Constraint Priority, Hugging, dan Compression Resistance

> **Kategori:** iOS & UIKit · **Level:** Menengah · **Status:** 🚧 kerangka

## Kerangka isi

### 1. Auto Layout adalah solver, bukan aturan
- Cassowary linear constraint solver
- Constraint = persamaan/pertidaksamaan dengan prioritas
- Implikasi performa: biayanya superlinear terhadap jumlah constraint

### 2. Prioritas
- `.required` (1000), `.defaultHigh` (750), `.defaultLow` (250)
- Kenapa 999 dipakai untuk constraint yang harus mengalah saat konflik sementara
  (mis. cell yang sedang berubah tinggi)
- Membaca log "Unable to simultaneously satisfy constraints"

### 3. Content hugging vs compression resistance
- Hugging = menolak membesar; compression resistance = menolak mengecil
- Kasus dua label bersebelahan: siapa yang di-truncate
- Contoh dari `TrackCell`: `saveButton.setContentHuggingPriority(.required, for: .horizontal)`

### 4. Constraint yang benar untuk cell yang self-sizing
- Rantai vertikal lengkap dari `contentView.top` ke `contentView.bottom`
- `greaterThanOrEqualTo` / `lessThanOrEqualTo` untuk padding fleksibel
- Kenapa `.estimated` yang jauh meleset memicu perhitungan ulang

### 5. Layout guide
- `safeAreaLayoutGuide`, `readableContentGuide`, `layoutMarginsGuide`
- Contoh helper dari project `movie`: `pinToEdges(of:)`

### 6. Kapan Auto Layout bukan jawabannya
- Cell yang sangat kompleks di scroll view berperforma tinggi → layout manual
- Compositional layout dengan `.absolute` menghindari solver untuk kasus grid

## Cek pemahaman (draft)
1. Apa bedanya priority 999 dan 1000 secara praktis?
2. Dua label bersebelahan, yang mana yang di-truncate dan bagaimana mengendalikannya?
3. Kenapa self-sizing cell butuh rantai constraint vertikal yang lengkap?
