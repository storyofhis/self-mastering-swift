---
title: "Interview Prep 06 — Architecture & iOS System Design"
category: "Interview Prep"
status: draft
tags:
  - swift-mastering
  - interview-prep
  - interview
---

# Interview Prep 06 — Architecture & iOS System Design

> **Status:** 🚧 kerangka — format & soal sudah ada, jawaban model belum ditulis penuh

## Format sesi system design iOS (biasanya 45–60 menit)

Berbeda dari system design backend. Yang diuji bukan sharding database, tapi:

| Dimensi | Pertanyaan yang harus kamu jawab sendiri |
|---|---|
| **Lapisan** | Bagaimana data mengalir dari jaringan ke pixel? |
| **State** | Siapa pemilik kebenaran? Bagaimana ia dibagikan? |
| **Offline** | Apa yang terjadi tanpa internet? Optimistic update? |
| **Concurrency** | Apa yang jalan di main thread, apa yang tidak? |
| **Skala data** | 10 item vs 100.000 item — apa yang berubah? |
| **Testability** | Di mana seam-nya? |
| **Error & edge case** | Kegagalan sebagian, token kedaluwarsa, race |
| **Trade-off** | Apa yang kamu tolak, dan kapan itu akan menyakitkan |

**Kerangka jawaban 5 langkah:**
1. Klarifikasi requirement & batasan (2 menit) — jangan langsung menggambar
2. Sketsa lapisan & aliran data (10 menit)
3. Perdalam satu bagian yang paling menarik (15 menit)
4. Bahas kegagalan & edge case (10 menit)
5. Sebutkan trade-off dan apa yang akan kamu ubah kalau skalanya 100× (5 menit)

## Soal desain yang harus dilatih

1. **Feed sosial dengan infinite scroll** — pagination, cache, prefetch, image loading,
   optimistic like
2. **Aplikasi chat** — real-time (WebSocket), pesan pending/gagal, urutan, offline queue
3. **Music/video player** — background audio, now playing, antrian, buffering
4. **Aplikasi peta dengan tracking lokasi** — background location, battery, geofencing,
   akurasi vs konsumsi daya
5. **Offline-first note app** — sinkronisasi, konflik, `NSPersistentCloudKitContainer` vs CRDT
6. **Image loader dari nol** — dua lapis cache, dedup, downsampling, prefetch
   *(sudah dibahas sebagian di [04-uikit-ios.md](04-uikit-ios.md) soal 10)*
7. **Feature flag / A-B testing system**
8. **Analytics pipeline di client** — batching, retry, persistence, privacy

## Pertanyaan arsitektur non-desain

9. Kapan kamu memilih MVVM dan kapan VIPER? Sebut titik di mana biayanya berbalik.
10. Bagaimana kamu memecah app monolitik jadi modul SPM? Mulai dari mana?
11. Bagaimana kamu memigrasikan codebase 200 ribu baris ke Swift 6?
12. Bagaimana kamu memutuskan menambah dependency pihak ketiga?
13. Bagaimana kamu menangani perubahan API yang breaking di tengah rilis?

## Yang membedakan jawaban senior
- Menyebut **apa yang ditolak** dan **kenapa**
- Menyebut **harga** yang dibayar, dan **kapan** harga itu mulai terasa
- Bertanya balik tentang batasan sebelum mendesain
- Bicara tentang tim dan proses, bukan hanya kode

Model jawaban terbaik untuk poin-poin ini ada di
`../../apple/movie/DECISIONS.md` — tiru **strukturnya**, bukan isinya.
