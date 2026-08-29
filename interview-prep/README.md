---
title: "Interview Prep"
category: "Interview Prep"
status: complete
tags:
  - swift-mastering
  - interview-prep
  - interview
---

# Interview Prep

Bank soal untuk posisi iOS Engineer, disusun per kategori dan per tingkat.

## Cara pakai — bukan dibaca, tapi dikerjakan

1. **Tutup jawabannya.** Setiap soal punya `<details>` — jangan dibuka dulu.
2. **Tulis jawabanmu**, atau ucapkan keras-keras. Menulis mengungkap lubang yang
   tidak terlihat saat "merasa paham".
3. **Baru buka jawaban model.** Bandingkan strukturnya, bukan kata-katanya.
4. **Baca bagian _Follow-up_.** Ini pertanyaan lanjutan yang biasanya dilempar
   interviewer setelah jawaban benar. Di sinilah kandidat mid dan senior berpisah.
5. **Tandai yang meleset**, ulang 3 hari kemudian.

## Tingkatan

| Tanda | Level | Ekspektasi |
|---|---|---|
| 🟢 | Junior (0–2 th) | Tahu API-nya, bisa memakainya dengan benar |
| 🟡 | Mid (2–5 th) | Tahu mekanismenya, bisa menjelaskan trade-off |
| 🔴 | Senior (5+ th) | Tahu kapan aturannya tidak berlaku, dan bisa mendesain di sekitarnya |

Jangan lewati 🟢. Interview sering gagal bukan di soal sulit, tapi di soal mudah
yang dijawab dengan ragu-ragu.

## Kategori

| File | Isi | Status |
|---|---|---|
| [01 — Swift Language](01-swift-language.md) | Value/reference, optional, enum, generics, protocol | ✅ |
| [02 — Memory Management](02-memory-management.md) | ARC, retain cycle, weak/unowned, COW, leak hunting | ✅ |
| [03 — Concurrency](03-concurrency.md) | GCD, async/await, actor, Sendable, cancellation | ✅ |
| [04 — UIKit & iOS Platform](04-uikit-ios.md) | Lifecycle, layout, cell reuse, networking, persistence | ✅ |
| [05 — SwiftUI](05-swiftui.md) | State, identity, Observation, performance | 🚧 |
| [06 — Architecture & System Design](06-architecture-system-design.md) | MVVM/VIPER/TCA, desain fitur end-to-end | 🚧 |
| [07 — Coding Challenge](07-coding-challenge.md) | Live coding: algoritma + Swift idiomatik | 🚧 |
| [08 — Behavioral & Project Walkthrough](08-behavioral-project-walkthrough.md) | STAR, cerita project, konflik, kegagalan | 🚧 |

## Aturan menjawab yang berlaku untuk semua kategori

**Jawab dalam tiga lapis, berhenti kalau interviewer sudah puas:**

1. **Definisi singkat** (1 kalimat)
2. **Mekanismenya** (kenapa begitu, di level implementasi)
3. **Trade-off / kapan tidak berlaku**

Contoh, untuk "apa itu `weak`?":

> 1. "Referensi yang tidak menambah strong reference count, jadi ia tidak menahan
>    objeknya hidup, dan otomatis jadi `nil` saat objeknya di-deallocate."
> 2. "Di implementasinya, objek yang punya weak reference mendapat *side table* —
>    alokasi terpisah yang menyimpan refcount, dan weak reference menunjuk ke sana,
>    bukan ke objeknya. Itu sebabnya ia bisa jadi `nil` dengan aman."
> 3. "Harganya: satu alokasi permanen begitu weak pertama dibuat, dan pembacaan yang
>    lebih mahal dari `unowned`. Jadi untuk hot path saya pilih `unowned` kalau
>    lifetime-nya benar-benar terjamin."

Lapis 2 dan 3 yang membedakan. Sebagian besar kandidat berhenti di lapis 1.

**Kalau tidak tahu:** katakan tidak tahu, lalu tunjukkan cara kamu mencari tahu.
*"Saya belum pernah pakai itu. Kalau ketemu di kerjaan, saya akan mulai dari
dokumentasi Apple dan proposal Swift Evolution-nya — biasanya bagian Motivation-nya
menjelaskan masalah apa yang mau diselesaikan."* Ini jawaban yang jauh lebih baik
daripada menebak dengan percaya diri.
