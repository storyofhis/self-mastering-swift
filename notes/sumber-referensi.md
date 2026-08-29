---
title: "Sumber & Referensi"
category: "Notes"
status: complete
tags:
  - swift-mastering
  - meta
---

# Sumber & Referensi

Prioritas sumber untuk catatan ini: **primer dulu, turunan belakangan.**

---

## Tingkat 1 — Sumber primer (paling dipercaya)

### Source code Swift (ada di mesin ini)
`~/Documents/Project/Learn Swift/swift/` — repo compiler & stdlib Swift resmi.

Yang paling sering dipakai di catatan ini:

| Path | Isi |
|---|---|
| `docs/ABI/TypeLayout.rst` | Layout existential container, enum, extra inhabitants |
| `docs/ABI/TypeMetadata.rst` | Struktur type metadata |
| `docs/ABI/KeyPaths.md` | Representasi runtime key path |
| `docs/ABI/GenericSignature.md` | Generic signature & witness table |
| `docs/OwnershipManifesto.md` | `borrowing`/`consuming`, noncopyable |
| `docs/ErrorHandling.md`, `ErrorHandlingRationale.md` | Desain `throws` |
| `docs/Arrays.md`, `ArrayImplementation.png` | Implementasi `Array` |
| `docs/Generics/` | Buku implementasi generics (LaTeX) |
| `stdlib/public/core/Array.swift` | `_makeMutableAndUnique`, COW |
| `stdlib/public/core/ManagedBuffer.swift` | `isKnownUniquelyReferenced` + doc |
| `stdlib/public/core/String*.swift` | String internals, small string |
| `stdlib/public/Concurrency/Actor.swift` | `protocol Actor`, `unownedExecutor` |
| `stdlib/public/Concurrency/MainActor.swift` | `@globalActor MainActor` |
| `stdlib/public/SwiftShims/swift/shims/RefCount.h` | Tiga refcount, side table, state machine |
| `userdocs/diagnostics/` | Penjelasan resmi tiap diagnostik compiler |

**Cara memakainya:**
```bash
cd ~/Documents/Project/Learn\ Swift/swift
grep -rn "isKnownUniquelyReferenced" stdlib/public/core/ | head
grep -rn -i "existential container" docs/
```

### Swift Evolution
https://github.com/swiftlang/swift-evolution/tree/main/proposals

Setiap fitur bahasa punya proposal. **Bagian "Motivation" adalah bagian terbaik** —
ia menjelaskan masalah apa yang mau diselesaikan, yang jauh lebih berguna daripada
sintaksnya.

Proposal yang paling sering dirujuk di catatan ini:
SE-0258 (property wrappers) · SE-0289 (result builders) · SE-0296 (async/await) ·
SE-0298 (AsyncSequence) · SE-0302 (Sendable) · SE-0304 (structured concurrency) ·
SE-0306 (actors) · SE-0309 (existential unlock) · SE-0316 (global actors) ·
SE-0341 (opaque parameters) · SE-0377 (ownership modifiers) · SE-0390 (noncopyable) ·
SE-0413 (typed throws) · SE-0414 (region-based isolation) · SE-0430 (`sending`) ·
SE-0466 (default actor isolation)

### Dokumentasi resmi
- [The Swift Programming Language](https://docs.swift.org/swift-book/)
- [Apple Developer Documentation](https://developer.apple.com/documentation/)
- [Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/)
- [Swift.org blog](https://www.swift.org/blog/) — pengumuman rilis dengan detail yang akurat

---

## Tingkat 2 — Sumber sekunder berkualitas tinggi

- **Hacking with Swift** (Paul Hudson) — https://www.hackingwithswift.com
  100 Days of Swift/SwiftUI untuk fondasi; "What's new in Swift X" untuk mengikuti rilis.
  Kekuatannya: cepat, praktis, contoh berjalan. Batasnya: jarang masuk ke level
  implementasi — untuk itu kembali ke Tingkat 1.
- **Swift by Sundell** (John Sundell) — artikel arsitektur & pola desain
- **SwiftLee** (Antoine van der Lee) — paling cepat membahas fitur baru
- **objc.io** — buku Advanced Swift & Thinking in SwiftUI; paling dekat ke level source
- **Point-Free** — teori tipe & fungsional, asal-usul TCA
- **WWDC session videos** — https://developer.apple.com/videos/
  Sesi yang paling sering dirujuk di catatan ini:
  - WWDC16 #416 — Understanding Swift Performance
  - WWDC19 #215, #220 — Compositional Layout, UI Data Sources
  - WWDC21 #10133, #10254 — Actors, Concurrency Behind the Scenes
  - WWDC22 #110352 — Embrace Swift Generics
  - WWDC21 #10252 — Blazing fast lists and collection views

---

## Tingkat 3 — Project di mesin ini sebagai bahan contoh

`~/Documents/Project/apple/movie/` — app UIKit + MVVM + async/await, klien iTunes Search.

| File | Dipakai di artikel |
|---|---|
| `DECISIONS.md` | Arsitektur, trade-off, project walkthrough |
| `REFLECTION.md` | Refleksi & utang teknis |
| `HomeViewModel.swift` | `async let`, `[weak self]`, batas lapisan |
| `SearchViewModel.swift` | Debounce, cancellation, state enum |
| `iTunesService.swift` | `URLComponents`, generic fetch, error mapping |
| `SearchResponses.swift` | Lossy decoding |
| `Track.swift` | `Codable` + `CodingKeys`, `Sendable` gratis |
| `TrackRowViewModel.swift` | Kapan display model layak dibuat |
| `TrackCell.swift` | Delegation, `weak`, `prepareForReuse` |
| `HomeViewController.swift` | Diffable data source, compositional layout |
| `LibraryStore.swift` | Singleton + DI, persistence sederhana |
| `UIKitBuilder/` | Local SPM package, builder pattern |

---

## Cara memverifikasi klaim teknis

Urutan yang dipakai saat menulis catatan ini:

1. Cari di source code Swift lokal (`grep -rn`)
2. Cari proposal Swift Evolution-nya, baca bagian Motivation & Detailed Design
3. Cek dokumentasi Apple untuk API-nya
4. **Buktikan di Playground** — `MemoryLayout`, `print`, `deinit`, address printing
5. Baru cari artikel/video kalau tiga langkah pertama tidak menjelaskan

Langkah 4 penting: banyak "fakta" yang beredar tentang Swift sudah kedaluwarsa
beberapa versi. `MemoryLayout` dan `deinit { print(...) }` menyelesaikan sebagian
besar perdebatan dalam 10 detik.
