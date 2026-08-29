---
title: "Roadmap: Dari Bisa Swift ke Dipercaya Pegang Codebase iOS"
category: "Index"
status: index
tags:
  - swift-mastering
  - index
---

# Roadmap: Dari Bisa Swift ke Dipercaya Pegang Codebase iOS

Roadmap ini bukan daftar topik. Ini urutan **kompetensi** — tiap tahap punya
"kamu dianggap lulus kalau bisa menjawab/melakukan ini".

Aturan mainnya: **jangan lompat tahap karena topiknya kedengaran keren.**
Actor dan `Sendable` akan terasa random kalau kamu belum benar-benar paham
value semantics dan ARC, karena keduanya dibangun persis di atas dua hal itu.

---

## Tahap 0 — Fondasi Bahasa (1–2 minggu)

Kamu kemungkinan besar sudah lewat ini. Cek cepat saja.

- Optional, `if let` / `guard let`, optional chaining, nil-coalescing
- `struct` vs `class` vs `enum` — bukan definisinya, tapi *kapan pilih yang mana*
- Closure & capture list dasar
- Protocol + extension, protocol-oriented style
- `Codable`
- Generic dasar (`func first<T>(...)`)

**Lulus kalau:** kamu bisa menjelaskan kenapa `struct` dengan property `var` di dalam
`let` konstanta tidak bisa dimutasi, tapi `class` bisa.

📖 Sumber: [Hacking with Swift — 100 Days of Swift](https://www.hackingwithswift.com/100),
[The Swift Programming Language](https://docs.swift.org/swift-book/)

---

## Tahap 1 — Memori & Semantik Nilai (2–3 minggu) ⬅ *tahap yang paling sering dilewati*

Ini pembeda terbesar antara kandidat junior dan mid di interview iOS.

- [Value vs reference semantics](articles/swift-language/01-value-vs-reference-semantics.md) — stack, heap, `HeapObject`
- [Copy-on-write](articles/swift-language/02-copy-on-write-internals.md) — `isKnownUniquelyReferenced`, kenapa `Array` murah disalin
- [ARC](articles/swift-language/03-arc-dan-retain-cycle.md) — retain/release, `weak` vs `unowned` vs `unowned(unsafe)`, side table
- [Layout `Optional` & enum](articles/swift-language/05-optional-dan-enum-layout.md) — spare bit, kenapa `Optional<Bool>` tetap 1 byte
- Exclusive access to memory (`inout`, simultaneous access)

**Lulus kalau:** diberi sepotong kode dengan closure di dalam `class`, kamu bisa
menunjuk retain cycle-nya, menjelaskan siapa yang menahan siapa, dan memilih antara
`[weak self]` dan `[unowned self]` **dengan alasan**, bukan kebiasaan.

📖 Sumber: `../swift/stdlib/public/core/`, `../swift/docs/ABI/TypeLayout.rst`

---

## Tahap 2 — Abstraksi & Compiler (2 minggu)

- [Existential (`any P`) vs generic (`some P`)](articles/swift-language/04-protocol-existential-vs-generics.md) — witness table, boxing, spesialisasi
- Static vs dynamic dispatch: kapan `final`, `private`, dan whole-module-optimization mengubah dispatch
- `@inlinable`, `@frozen`, dan library evolution (kenapa stdlib dipenuhi atribut ini)
- Associated type, `some`/`any`, primary associated type (`Collection<Int>`)
- [Property wrapper & result builder](articles/swift-language/08-property-wrapper-result-builder.md) — apa yang di-desugar compiler

**Lulus kalau:** kamu bisa menjelaskan kenapa `[any Shape]` lebih lambat dari
`[Circle]` di level representasi memori, bukan cuma bilang "karena dynamic dispatch".

---

## Tahap 3 — Concurrency Modern (3–4 minggu)

- [Kenapa GCD ditinggalkan](articles/concurrency/01-gcd-ke-swift-concurrency.md) — thread explosion, priority inversion, cooperative thread pool
- [`async/await` & continuation](articles/concurrency/02-async-await-continuation.md) — suspension point, async frame, `withCheckedContinuation`
- [Structured concurrency](articles/concurrency/05-structured-concurrency-cancellation.md) — `async let`, `TaskGroup`, cancellation kooperatif
- [Actor](articles/concurrency/03-actor-isolation-reentrancy.md) — isolation, executor, **reentrancy**
- [`Sendable`](articles/concurrency/04-sendable-data-race-safety.md) — region-based isolation, `sending`, `@unchecked`
- [Swift 6.2 default isolation](articles/concurrency/06-swift62-default-isolation.md) — `nonisolated(nonsending)`, `@concurrent`
- Migrasi ke Swift 6 language mode

**Lulus kalau:** kamu bisa menjelaskan kenapa kode ini bisa *tetap* punya bug logika
meski compiler tidak protes:

```swift
actor Counter {
    private var value = 0
    func incrementAfterFetch() async {
        let remote = await fetchRemote()   // ← suspension point
        value = remote + 1                 // ← state bisa sudah berubah di sini
    }
}
```

---

## Tahap 4 — iOS Platform (4 minggu)

- [App & scene lifecycle](articles/ios-uikit/04-app-dan-scene-lifecycle.md)
- [View controller lifecycle + layout pass](articles/ios-uikit/05-viewcontroller-lifecycle-layout-pass.md) — `setNeedsLayout` vs `layoutIfNeeded`
- [Auto Layout mendalam](articles/ios-uikit/07-autolayout-mendalam.md) — priority, hugging, compression resistance
- [Diffable data source & compositional layout](articles/ios-uikit/02-diffable-data-source-compositional-layout.md)
- [Networking](articles/ios-uikit/03-networking-urlsession-codable.md) — `URLSession`, `Codable`, error mapping
- [Cell reuse & image loading](articles/ios-uikit/06-cell-reuse-dan-image-loading.md)
- Persistence: `UserDefaults` → Core Data / SwiftData / GRDB
- Background execution, push notification, deep link

**Lulus kalau:** kamu bisa membuka Instruments, menemukan leak yang kamu buat sendiri
dengan sengaja, dan menutupnya.

---

## Tahap 5 — Arsitektur & Rekayasa (berkelanjutan)

- [Memilih arsitektur berdasarkan biaya](articles/architecture/01-memilih-arsitektur.md) — MVC, MVVM, VIPER, TCA
- [Modularisasi SPM](articles/architecture/02-modularisasi-dengan-spm.md)
- [Testing](articles/testing/01-swift-testing-vs-xctest.md) — Swift Testing, XCTest, test double lewat protocol
- [Performance](articles/performance/01-instruments-workflow.md) — profiling sebelum menebak
- CI/CD, code signing, fastlane, App Store review

**Lulus kalau:** kamu bisa menulis dokumen seperti `../../apple/movie/DECISIONS.md` —
menyebut apa yang kamu **tolak** dan **harga** yang kamu bayar untuk pilihan kamu.
Ini yang dicari interviewer senior. Bukan "saya pakai MVVM", tapi
"saya menolak VIPER karena X, dan konsekuensinya Y akan mulai sakit kalau Z".

---

## Tahap 6 — Interview Mode (2 minggu sebelum hari-H)

1. Baca ulang [cheat sheet](cheatsheet/swift-quick-reference.md) — 30 menit
2. Kerjakan [bank soal](interview-prep/) per kategori, **tulis jawabanmu dulu** sebelum
   melihat jawaban model
3. Latih [project walkthrough](interview-prep/08-behavioral-project-walkthrough.md)
   pakai project `movie` — 5 menit versi ringkas, 15 menit versi dalam
4. Latih [system design iOS](interview-prep/06-architecture-system-design.md) — 2 sesi

---

## Peta ketergantungan

```
Tahap 0  Fondasi Bahasa
    │
    ▼
Tahap 1  Memori & Value Semantics ──────┐
    │                                    │
    ▼                                    ▼
Tahap 2  Abstraksi & Compiler      Tahap 4  iOS Platform
    │                                    │
    ▼                                    │
Tahap 3  Concurrency ◄───────────────────┘
    │
    ▼
Tahap 5  Arsitektur & Rekayasa
    │
    ▼
Tahap 6  Interview Mode
```

Tahap 4 bisa jalan paralel dengan Tahap 2–3 — kamu tidak perlu menguasai witness table
untuk memasang Auto Layout. Tapi Tahap 3 **wajib** setelah Tahap 1.
