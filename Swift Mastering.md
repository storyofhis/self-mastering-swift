---
title: "Swift Mastering"
category: "Index"
status: index
tags:
  - swift-mastering
  - index
---

# Swift Mastering

Folder note / dashboard untuk vault ini. Catatan deep-dive Swift & iOS,
ditulis dengan sumber primer (source code compiler Swift, proposal Swift Evolution)
dan contoh dari project `movie`.

## Mulai dari sini

- [Index lengkap semua artikel](README.md)
- [Roadmap belajar bertahap](ROADMAP.md)
- [Interview prep](interview-prep/README.md)
- [Cheat sheet 30 menit sebelum interview](cheatsheet/swift-quick-reference.md)
- [Sumber & cara memverifikasi klaim teknis](notes/sumber-referensi.md)

## Cari lewat tag

Semua catatan di sini punya tag `#swift-mastering`. Tag kategorinya bersarang:

| Tag | Isi |
|---|---|
| `#swift/language` | Value semantics, COW, ARC, existential, enum layout, ownership |
| `#swift/concurrency` | async/await, actor, Sendable, structured concurrency |
| `#swift/testing` | Swift Testing, XCTest, test double |
| `#ios/uikit` | Lifecycle, layout, diffable, networking, cell reuse |
| `#ios/swiftui` | State & identity, Observation, interop |
| `#ios/performance` | Instruments, biaya tersembunyi, launch time |
| `#ios/architecture` | MVVM/VIPER/TCA, modularisasi, persistence |
| `#interview-prep` | Bank soal bertingkat + jawaban model |

Filter juga bisa lewat `status`:
- `status: complete` — artikel yang sudah ditulis penuh
- `status: draft` — kerangka terstruktur, siap diisi

Contoh query di search bar Obsidian:
```
tag:#swift/concurrency ["status": complete]
```

## Cara memakai vault ini

1. **Belajar terstruktur** → ikuti [ROADMAP.md](ROADMAP.md) dari Tahap 0.
   Jangan lompat Tahap 1 (memori & value semantics) — Tahap 3 bergantung padanya.
2. **Menjawab satu pertanyaan** → cari lewat tag, atau buka [README.md](README.md).
3. **Persiapan interview** → [interview-prep/README.md](interview-prep/README.md),
   tutup jawabannya, tulis dulu, baru bandingkan.
4. **Graph view** → semua artikel saling menaut. Cluster yang terbentuk menunjukkan
   topik mana yang saling bergantung; artikel yang berdiri sendiri biasanya tanda
   catatan itu belum terhubung ke apa pun dan layak ditinjau.

## Menulis lanjutan

File ber-`status: draft` sudah punya kerangka lengkap: poin kunci, tabel yang harus
diisi, draft soal cek pemahaman, dan daftar sumber yang harus dibaca. Isi dari sana,
lalu ubah `status` di frontmatter jadi `complete`.

## Konvensi

- Bahasa Indonesia, istilah teknis tetap Inggris (*retain cycle*, *witness table*)
- Kutipan source code selalu menyebut path filenya
- Setiap artikel diakhiri: Ringkasan → Selanjutnya → Sumber
- Setiap klaim teknis bisa dilacak ke sumber primer, bukan ke blog turunan
