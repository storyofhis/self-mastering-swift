---
title: "Swift Testing vs XCTest, dan Interop-nya di Swift 6.4"
category: "Testing"
status: draft
tags:
  - swift-mastering
  - swift/testing
  - article
---

# Swift Testing vs XCTest, dan Interop-nya di Swift 6.4

> **Kategori:** Testing · **Level:** Menengah · **Status:** 🚧 kerangka

## Kerangka isi

### 1. Swift Testing sekilas
```swift
import Testing

@Test func trackRowFormatsDuration() {
    let vm = TrackRowViewModel(track: .fixture(durationMillis: 225_000), isSaved: false)
    #expect(vm.duration == "3:45")
}

@Test(arguments: [(0, "0:00"), (59_000, "0:59"), (225_000, "3:45")])
func durationFormatting(millis: Int, expected: String) { ... }
```
- `#expect` vs `#require` (yang kedua menghentikan test)
- Parameterized test menggantikan loop manual
- `@Suite`, tag, dan paralelisme default

### 2. Perbandingan dengan XCTest
| | XCTest | Swift Testing |
|---|---|---|
| Penemuan test | prefix `test` | atribut `@Test` |
| Assertion | `XCTAssertEqual` dkk | `#expect` / `#require` |
| Parameterized | manual | built-in |
| Paralel | opt-in per class | default |
| Async | `async` didukung | native |
| UI testing | ✅ | ❌ (masih XCTest) |

### 3. Interop di Swift 6.4
- Kegagalan XCTest dilaporkan sebagai issue di dalam Swift Testing
- API Swift Testing bisa dipakai di dalam XCTest
- Isu lintas-framework jadi warning secara default, bisa dinaikkan jadi failure
- Artinya: migrasi bertahap tanpa kehilangan coverage

### 4. Apa yang tetap di XCTest
- UI test (`XCUIApplication`)
- Performance test (`measure`)

### 5. Strategi migrasi
Unit test dulu, per target, mulai dari yang tidak punya `setUp`/`tearDown` kompleks.

## Sumber untuk ditulis
- [Swift Testing — swift.org](https://www.swift.org/documentation/testing/)
- InfoQ: Swift 6.4 Swift Testing/XCTest interop
