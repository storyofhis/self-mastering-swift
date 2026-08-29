---
title: "Value vs Reference Semantics: Stack, Heap, dan Apa yang Sebenarnya Disalin"
category: "Swift Language & Internals"
status: complete
tags:
  - swift-mastering
  - swift/language
  - article
---

# Value vs Reference Semantics: Stack, Heap, dan Apa yang Sebenarnya Disalin

> **Kategori:** Swift Language & Internals · **Level:** Fondasi ·
> **Prasyarat:** paham `struct`, `class`, dan closure dasar
> **Baca ~18 menit**

---

## Kenapa ini artikel pertama

Hampir semua "keanehan" Swift yang bikin frustrasi bermuara ke satu pertanyaan:
**siapa yang memegang nilai ini, dan apa yang terjadi kalau saya menyalinnya?**

- Kenapa `struct` di dalam `let` tidak bisa diubah tapi `class` bisa?
- Kenapa mutasi array di dalam closure kadang "hilang"?
- Kenapa `actor` hanya bisa dipakai untuk `class`, bukan `struct`?
- Kenapa `Sendable` gampang untuk `struct` dan susah untuk `class`?

Semuanya jawaban dari artikel ini. Kalau bagian ini kabur, Tahap 3 (concurrency)
akan terasa seperti hafalan aturan compiler.

---

## 1. Dua model, satu perbedaan

```swift
struct PointS { var x: Int; var y: Int }
final class PointC { var x: Int; var y: Int; init(x: Int, y: Int) { self.x = x; self.y = y } }

var a = PointS(x: 1, y: 2)
var b = a          // menyalin NILAI
b.x = 99
print(a.x)         // 1  ← a tidak tersentuh

let c = PointC(x: 1, y: 2)
let d = c          // menyalin REFERENSI (alamat)
d.x = 99
print(c.x)         // 99 ← c ikut berubah, karena c dan d adalah objek yang sama
```

Kalimat kuncinya: **value type menyalin isinya, reference type menyalin alamatnya.**

Tapi ini penjelasan level permukaan. Yang bikin kamu bisa menjawab follow-up
interviewer adalah level di bawahnya.

---

## 2. Apa yang sebenarnya ada di memori

### Value type: isi langsung, inline

`PointS` di atas **adalah** dua `Int` yang bersebelahan. Tidak ada header, tidak ada
pointer, tidak ada alokasi terpisah.

```
var a = PointS(x: 1, y: 2)

  a ┌───────────────┬───────────────┐
    │   x = 1 (8B)  │   y = 2 (8B)  │      total: 16 byte
    └───────────────┴───────────────┘
```

Kamu bisa membuktikannya:

```swift
MemoryLayout<PointS>.size        // 16
MemoryLayout<PointS>.stride      // 16
MemoryLayout<PointS>.alignment   // 8
```

### Reference type: satu pointer, isinya di heap

```swift
MemoryLayout<PointC>.size        // 8  ← hanya pointer!
```

```
let c = PointC(x: 1, y: 2)

  c ┌──────────┐
    │ pointer  │────────┐
    └──────────┘        │
                        ▼   (di heap)
      ┌─────────────┬──────────────┬─────────┬─────────┐
      │ metadata ptr│ refCounts    │  x = 1  │  y = 2  │
      └─────────────┴──────────────┴─────────┴─────────┘
      └────── HeapObject header (16 byte) ──────┘
```

Setiap instance `class` di Swift punya header 16 byte (di platform 64-bit):
satu pointer ke **type metadata** (dipakai untuk dynamic dispatch, casting, refleksi)
dan satu word untuk **reference counts** (strong + unowned, dengan side table
untuk weak — lihat [artikel ARC](03-arc-dan-retain-cycle.md)).

Konsekuensi langsung: `class` dengan dua `Int` memakan **32 byte di heap + 8 byte pointer**,
sementara `struct` yang sama memakan **16 byte** tanpa alokasi heap sama sekali.

### `size` vs `stride` — pertanyaan interview favorit

```swift
struct Mixed { var flag: Bool; var value: Int }
MemoryLayout<Mixed>.size      // 9   ← 1 byte Bool + 8 byte Int
MemoryLayout<Mixed>.stride    // 16  ← dibulatkan ke kelipatan alignment
MemoryLayout<Mixed>.alignment // 8
```

`size` = byte yang benar-benar dipakai. `stride` = jarak antar elemen di dalam array.
`Array<Mixed>` dengan 100 elemen memakan `100 × 16 = 1600` byte, bukan 900.

> **Optimasi gratis:** urutkan property dari yang alignment-nya besar ke kecil.
> `struct { Int; Int; Bool }` → stride 24. `struct { Bool; Int; Int }` → stride 24 juga
> (padding di tengah). Tapi `struct { Bool; Bool; Int }` → stride 16, sementara
> `struct { Bool; Int; Bool }` → stride 24. Padding itu nyata.

---

## 3. "Value type ada di stack" — mitos yang setengah benar

Ini kalimat yang paling sering diucapkan dan paling sering salah di interview.

Yang benar: **value type tidak punya identitas dan tidak dialokasi sendiri di heap.
Di mana ia berada tergantung siapa yang memuatnya.**

```swift
struct Config { var retries: Int }

func f() {
    var local = Config(retries: 3)   // ✅ stack
}

final class Holder {
    var config = Config(retries: 3)  // ❌ bukan stack — ikut di heap, di dalam Holder
}

let boxed: [Config] = [Config(retries: 3)]  // ❌ di buffer heap milik Array

func g() {
    var local = Config(retries: 3)
    DispatchQueue.global().async { print(local) }  // ❌ ter-capture → dipindahkan ke heap box
}
```

Jawaban yang membuat interviewer mengangguk:

> "Struct disimpan *inline* di dalam apa pun yang memuatnya. Kalau pemuatnya adalah
> stack frame, ia di stack. Kalau pemuatnya instance class, array buffer, atau
> closure context, ia ikut di heap. Yang dijamin Swift bukan lokasinya — yang dijamin
> adalah ia tidak punya identitas terpisah dan tidak di-reference-count sendiri."

---

## 4. Existential: kapan struct kamu diam-diam masuk heap

```swift
protocol Drawable { func draw() }
struct BigShape: Drawable {
    var a, b, c, d, e: Int      // 40 byte
    func draw() {}
}

let items: [any Drawable] = [BigShape(a:1,b:2,c:3,d:4,e:5)]
```

`any Drawable` disimpan dalam **existential container**. Dari dokumen ABI resmi Swift
(`swift/docs/ABI/TypeLayout.rst`):

```c
struct OpaqueExistentialContainer {
  void *fixedSizeBuffer[3];      // 3 word = 24 byte
  Metadata *type;
  WitnessTable *witnessTables[NUM_WITNESS_TABLES];
};
```

> *"This either directly contains the value, if its size and alignment are both less
> than or equal to the fixed-size buffer's, or contains a pointer to a side allocation
> owned by the existential container."*

Artinya: **struct ≤ 24 byte muat inline (gratis). Struct > 24 byte memicu alokasi heap
setiap kali dibungkus jadi `any P`.** `BigShape` 40 byte → boxed → `malloc`.

Ini alasan konkret kenapa `[any Drawable]` bisa jauh lebih lambat dari `[BigShape]`,
dan jawaban yang membedakan kamu dari kandidat yang cuma bilang "karena dynamic dispatch".
Detailnya di [artikel existential vs generics](04-protocol-existential-vs-generics.md).

---

## 5. `let` bekerja berbeda pada keduanya

```swift
let s = PointS(x: 1, y: 2)
s.x = 5        // ❌ error: cannot assign to property: 's' is a 'let' constant

let c = PointC(x: 1, y: 2)
c.x = 5        // ✅ boleh
```

Kenapa? `let` membekukan **nilai yang disimpan variabel itu**.

- Untuk `PointS`, nilainya adalah `(x, y)` itu sendiri → membeku seluruhnya.
- Untuk `PointC`, nilainya adalah **pointer**-nya → yang membeku pointernya. Isi objek
  di ujung pointer itu bukan urusan `let`.

Ini juga kenapa `mutating` hanya ada di value type: method `mutating` sebenarnya
menerima `self` sebagai `inout` — ia menulis balik ke lokasi penyimpanan asli.
Method `class` tidak butuh itu karena mutasi terjadi lewat pointer.

```swift
extension PointS {
    mutating func moveRight() { x += 1 }
    // secara efektif: static func moveRight(_ self: inout PointS)
}
```

---

## 6. Jebakan praktis (yang benar-benar terjadi di app)

### 6a. Struct di dalam array: mutasi lewat variabel salinan

```swift
struct Item { var count = 0 }
var items = [Item(), Item()]

for var item in items { item.count += 1 }   // ❌ sia-sia
print(items[0].count)                        // 0 — `item` adalah SALINAN

for i in items.indices { items[i].count += 1 }  // ✅ menulis ke storage asli
```

Interviewer sering menaruh versi `map` dari bug ini:

```swift
items.map { var copy = $0; copy.count += 1; return copy }  // menghasilkan array baru,
                                                            // `items` tidak berubah
```

### 6b. Closure meng-capture value type: by reference, bukan snapshot

```swift
var counter = 0
let increment = { counter += 1 }
increment(); increment()
print(counter)   // 2 — bukan 0!
```

Closure meng-capture **variabel**, bukan nilainya. Compiler memindahkan `counter`
ke heap box supaya closure dan scope asli menunjuk storage yang sama.
Kalau kamu mau snapshot, minta eksplisit:

```swift
let snapshot = { [counter] in print(counter) }   // capture list = salin sekarang
```

### 6c. Reference type di dalam struct: value semantics-mu bocor

```swift
final class Buffer { var data: [Int] = [] }
struct Wrapper { var buffer = Buffer() }        // ⚠️

var w1 = Wrapper()
var w2 = w1              // "menyalin" Wrapper
w2.buffer.data.append(1)
print(w1.buffer.data)    // [1] — value semantics-nya PALSU
```

`Wrapper` terlihat seperti value type, tapi karena memuat referensi, salinan dangkalnya
berbagi state. Ini penyebab bug yang sangat sulit dilacak, dan alasan kenapa
`Sendable` **tidak** otomatis diberikan ke struct yang memuat class non-`Sendable`.

Kalau kamu memang butuh pola ini, kamu harus mengimplementasikan copy-on-write
sendiri — persis seperti yang dilakukan `Array` di stdlib.
Lihat [artikel COW](02-copy-on-write-internals.md).

---

## 7. Cara memilih: `struct` atau `class`?

Aturan praktis yang bisa kamu ucapkan di interview:

**Pakai `struct` secara default.** Naik ke `class` hanya kalau salah satu ini benar:

| Alasan | Contoh nyata |
|---|---|
| Kamu butuh **identitas** — dua objek dengan isi sama tetap objek berbeda | `URLSession`, `UIViewController`, `LibraryStore` |
| Kamu butuh **shared mutable state** yang disengaja | store, cache, koneksi |
| Kamu perlu **inheritance** dari kelas Objective-C | `UIView`, `UIViewController` |
| Kamu perlu `deinit` | menutup file handle, membatalkan observer |
| Objeknya besar & sering dioper, dan profiling membuktikan penyalinan itu mahal | *ukur dulu — COW biasanya sudah menyelesaikannya* |

Dalam project `movie` di mesin ini, pembagiannya persis mengikuti aturan ini:

```swift
struct Track: Codable, Hashable, Identifiable { ... }   // data murni → struct
struct iTunesService: MusicSearching { ... }            // stateless → struct
final class LibraryStore { static let shared = ... }    // shared mutable state → class
final class SearchViewModel { ... }                     // identitas + lifetime → class
```

`Track` sebagai `struct` bukan kebetulan: karena ia value type dan `Hashable`,
ia langsung bisa dipakai sebagai item `NSDiffableDataSourceSnapshot` tanpa risiko
dua cell diam-diam berbagi objek yang sama.

---

## 8. Cek pemahaman

Jawab tanpa menjalankan kodenya.

**Q1.**
```swift
struct S { var arr = [1, 2, 3] }
var s1 = S()
var s2 = s1
s2.arr.append(4)
print(s1.arr.count)
```
<details><summary>Jawaban</summary>

`3`. `Array` sendiri value type dengan copy-on-write. Saat `s2.arr.append(4)`,
buffer-nya tidak lagi unik → disalin dulu. `s1.arr` tetap `[1,2,3]`.
</details>

**Q2.** Berapa `MemoryLayout<Optional<PointC>>.size`? (`PointC` adalah class)
<details><summary>Jawaban</summary>

`8`. Sama dengan `PointC`. Pointer tidak pernah `0x0` untuk objek valid, jadi Swift
memakai nilai nol itu sebagai representasi `nil` — tanpa byte tambahan.
Ini disebut *spare bit / extra inhabitant*, dibahas di
[artikel layout Optional](05-optional-dan-enum-layout.md).
</details>

**Q3.** Kenapa `actor` harus reference type?
<details><summary>Jawaban</summary>

Karena isolation butuh **identitas**: runtime perlu tahu "state ini milik actor yang
mana" untuk menyerialkan akses ke executor yang sama. Value type disalin — dua salinan
akan punya state independen, sehingga tidak ada yang bisa diserialkan.
Karena itu `actor` selalu class-like dan otomatis `Sendable`.
</details>

---

## Ringkasan

- Value type = isi disimpan inline; reference type = pointer 8 byte ke objek heap
  dengan header 16 byte (metadata + refcount).
- "Struct di stack" itu setengah benar: struct ikut ke mana pun pemuatnya berada.
- `size` ≠ `stride`; `stride` yang menentukan konsumsi memori array.
- Existential (`any P`) memuat nilai inline hanya kalau ≤ 3 word (24 byte); lebih dari
  itu ada alokasi heap.
- `let` membekukan nilai variabel — untuk class, nilainya cuma pointer.
- Value semantics palsu kalau struct memuat class. Ini akar banyak bug dan alasan
  aturan `Sendable`.

## Selanjutnya

→ [02 — Copy-on-Write: Membedah `Array` dari Source stdlib](02-copy-on-write-internals.md)
→ [03 — ARC, Retain Cycle, dan Kapan `weak` Bukan Jawabannya](03-arc-dan-retain-cycle.md)

## Sumber

- `swift/docs/ABI/TypeLayout.rst` — Existential Container Layout
- `swift/stdlib/public/core/` — implementasi `Array`, `Optional`
- [The Swift Programming Language — Structures and Classes](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/classesandstructures/)
- [WWDC16 — Understanding Swift Performance](https://developer.apple.com/videos/play/wwdc2016/416/)
