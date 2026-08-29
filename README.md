---
title: "Swift Mastering"
category: "Index"
status: index
tags:
  - swift-mastering
  - index
---

# Swift Mastering

Catatan belajar Swift & iOS yang dalam, bukan ringkasan dokumentasi — artikel deep-dive per topik,
roadmap belajar, dan bank soal interview — tapi untuk Swift dan platform Apple.

**Prinsip catatan ini:**

1. **Jelaskan mekanismenya, bukan cuma API-nya.** Setiap artikel menjawab "kenapa begini",
   bukan cuma "cara pakainya begini".
2. **Sumbernya primer.** Banyak penjelasan diambil langsung dari source code compiler & stdlib Swift
   (`../swift/`), dokumen ABI resmi, dan Swift Evolution proposal — bukan dari blog turunan.
3. **Contohnya nyata.** Contoh kode iOS diambil dari project `movie` (UIKit + MVVM + async/await)
   yang ada di mesin ini, termasuk trade-off yang benar-benar diputuskan di `DECISIONS.md`.
4. **Bahasa Indonesia, istilah teknis Inggris.** *Retain cycle* tetap "retain cycle", bukan
   "siklus penahanan".

---

## Cara memakai catatan ini

| Kamu sedang... | Mulai dari |
|---|---|
| Baru mau menyusun jalur belajar | [`ROADMAP.md`](ROADMAP.md) |
| Ingin paham satu topik sampai dalam | [`articles/`](articles/) |
| Interview minggu depan | [`interview-prep/`](interview-prep/) |
| Perlu contekan cepat sebelum masuk ruangan | [`cheatsheet/`](cheatsheet/) |

Status tiap artikel: ✅ ditulis penuh · 🚧 kerangka (outline + poin kunci, siap diisi)

---

## Articles

### 🧬 Swift Language & Internals

Semantik nilai, memori, dan cara compiler benar-benar merepresentasikan tipe kamu.

| # | Artikel | Status |
|---|---|---|
| 01 | [Value vs Reference Semantics: Stack, Heap, dan Apa yang Sebenarnya Disalin](articles/swift-language/01-value-vs-reference-semantics.md) | ✅ |
| 02 | [Copy-on-Write: Membedah `Array` Langsung dari Source stdlib](articles/swift-language/02-copy-on-write-internals.md) | ✅ |
| 03 | [ARC, Retain Cycle, dan Kapan `weak` Bukan Jawabannya](articles/swift-language/03-arc-dan-retain-cycle.md) | ✅ |
| 04 | [Protocol: Existential Container vs Generic Specialization](articles/swift-language/04-protocol-existential-vs-generics.md) | ✅ |
| 05 | [Optional & Enum: Kenapa `Optional` Tidak Punya Overhead](articles/swift-language/05-optional-dan-enum-layout.md) | ✅ |
| 06 | [Error Handling: `throws`, `Result`, dan `typed throws`](articles/swift-language/06-error-handling.md) | 🚧 |
| 07 | [String Internals: Grapheme Cluster, UTF-8, dan Small String](articles/swift-language/07-string-internals.md) | 🚧 |
| 08 | [Property Wrapper & Result Builder: Sugar yang Di-desugar Compiler](articles/swift-language/08-property-wrapper-result-builder.md) | 🚧 |
| 09 | [Ownership: `borrowing`, `consuming`, dan Noncopyable Types](articles/swift-language/09-ownership-borrowing-consuming.md) | 🚧 |
| 10 | [Key Paths, Dynamic Member Lookup, dan Refleksi](articles/swift-language/10-keypath-dan-metaprogramming.md) | 🚧 |

### ⚡ Concurrency Modern

Model konkurensi Swift 5.5 → Swift 6.4: dari `async/await` sampai data-race safety yang dicek compiler.

| # | Artikel | Status |
|---|---|---|
| 01 | [Dari GCD ke Swift Concurrency: Thread Explosion dan Cooperative Pool](articles/concurrency/01-gcd-ke-swift-concurrency.md) | ✅ |
| 02 | [`async/await` Dibedah: Continuation, Suspension Point, dan Async Frame](articles/concurrency/02-async-await-continuation.md) | ✅ |
| 03 | [Actor: Isolation, Executor, dan Jebakan Reentrancy](articles/concurrency/03-actor-isolation-reentrancy.md) | ✅ |
| 04 | [`Sendable` dan Data-Race Safety: Cara Compiler Membuktikan Kode Kamu Aman](articles/concurrency/04-sendable-data-race-safety.md) | ✅ |
| 05 | [Structured Concurrency: `async let`, `TaskGroup`, dan Cancellation yang Benar](articles/concurrency/05-structured-concurrency-cancellation.md) | 🚧 |
| 06 | [Swift 6.2: Default Actor Isolation, `nonisolated(nonsending)`, `@concurrent`](articles/concurrency/06-swift62-default-isolation.md) | 🚧 |
| 07 | [`AsyncSequence` & `AsyncStream`: Bridging Dunia Callback](articles/concurrency/07-asyncsequence-asyncstream.md) | 🚧 |

### 📱 iOS & UIKit

Praktik nyata, dibedah dari project `movie` di mesin ini.

| # | Artikel | Status |
|---|---|---|
| 01 | [MVVM & Dependency Injection: Studi Kasus Project `movie`](articles/ios-uikit/01-mvvm-dan-di-studi-kasus.md) | ✅ |
| 02 | [Diffable Data Source & Compositional Layout: Kenapa `reloadData()` Sudah Usang](articles/ios-uikit/02-diffable-data-source-compositional-layout.md) | ✅ |
| 03 | [Networking: `URLSession`, `Codable`, dan Lossy Decoding](articles/ios-uikit/03-networking-urlsession-codable.md) | ✅ |
| 04 | [App & Scene Lifecycle: `UISceneDelegate` dan State Restoration](articles/ios-uikit/04-app-dan-scene-lifecycle.md) | 🚧 |
| 05 | [View Controller Lifecycle & Layout Pass](articles/ios-uikit/05-viewcontroller-lifecycle-layout-pass.md) | 🚧 |
| 06 | [Cell Reuse, Image Loading, dan Bug Gambar Ketuker](articles/ios-uikit/06-cell-reuse-dan-image-loading.md) | 🚧 |
| 07 | [Auto Layout: Constraint Priority, Hugging, dan Compression Resistance](articles/ios-uikit/07-autolayout-mendalam.md) | 🚧 |
| 08 | [Navigation & Coordinator Pattern](articles/ios-uikit/08-navigation-dan-coordinator.md) | 🚧 |

### 🎨 SwiftUI

| # | Artikel | Status |
|---|---|---|
| 01 | [State, Identity, dan Lifetime: Kenapa View Kamu Tidak Ter-update](articles/swiftui/01-state-identity-lifetime.md) | 🚧 |
| 02 | [`@Observable` vs `ObservableObject`: Apa yang Berubah di Observation](articles/swiftui/02-observable-vs-observableobject.md) | 🚧 |
| 03 | [Interop UIKit ↔ SwiftUI](articles/swiftui/03-interop-uikit-swiftui.md) | 🚧 |

### 🧪 Testing

| # | Artikel | Status |
|---|---|---|
| 01 | [Swift Testing vs XCTest, dan Interop-nya di Swift 6.4](articles/testing/01-swift-testing-vs-xctest.md) | 🚧 |
| 02 | [Menguji Kode Async & Actor](articles/testing/02-testing-async-dan-actor.md) | 🚧 |
| 03 | [Test Double: Fake, Stub, Spy lewat Protocol](articles/testing/03-test-double-lewat-protocol.md) | 🚧 |

### 🚀 Performance

| # | Artikel | Status |
|---|---|---|
| 01 | [Instruments: Time Profiler, Allocations, Leaks](articles/performance/01-instruments-workflow.md) | 🚧 |
| 02 | [Biaya Tersembunyi: Retain/Release Traffic, Bridging, Dynamic Dispatch](articles/performance/02-biaya-tersembunyi.md) | 🚧 |
| 03 | [Launch Time & Scroll Performance](articles/performance/03-launch-time-dan-scrolling.md) | 🚧 |

### 🏛 Architecture

| # | Artikel | Status |
|---|---|---|
| 01 | [MVC → MVVM → VIPER → TCA: Memilih Berdasarkan Biaya, Bukan Tren](articles/architecture/01-memilih-arsitektur.md) | 🚧 |
| 02 | [Modularisasi dengan Swift Package Manager](articles/architecture/02-modularisasi-dengan-spm.md) | 🚧 |
| 03 | [Persistence: `UserDefaults`, Core Data, SwiftData, GRDB](articles/architecture/03-pilihan-persistence.md) | 🚧 |

---

## Interview Prep

Bank soal bertingkat (Junior → Mid → Senior) dengan **jawaban model** dan **jebakan follow-up**
yang biasanya dilempar interviewer sesudah kamu menjawab.

- [Cara pakai & format](interview-prep/README.md)
- [01 — Swift Language](interview-prep/01-swift-language.md) ✅
- [02 — Memory Management & ARC](interview-prep/02-memory-management.md) ✅
- [03 — Concurrency](interview-prep/03-concurrency.md) ✅
- [04 — UIKit & iOS Platform](interview-prep/04-uikit-ios.md) ✅
- [05 — SwiftUI](interview-prep/05-swiftui.md) 🚧
- [06 — Architecture & iOS System Design](interview-prep/06-architecture-system-design.md) 🚧
- [07 — Coding Challenge (live coding)](interview-prep/07-coding-challenge.md) 🚧
- [08 — Behavioral & Project Walkthrough](interview-prep/08-behavioral-project-walkthrough.md) 🚧

## Cheat Sheet

- [Swift Quick Reference](cheatsheet/swift-quick-reference.md) ✅
- [iOS Quick Reference](cheatsheet/ios-quick-reference.md) 🚧

## Notes

- [Sumber & referensi](notes/sumber-referensi.md)

---

*Dibuat sebagai catatan pribadi. Semua kutipan source code milik Swift project
(Apache License v2.0 with Runtime Library Exception).*
