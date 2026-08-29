---
title: "Protocol: Existential Container vs Generic Specialization"
category: "Swift Language & Internals"
status: complete
tags:
  - swift-mastering
  - swift/language
  - article
---

# Protocol: Existential Container vs Generic Specialization

> **Kategori:** Swift Language & Internals · **Level:** Menengah–Lanjut ·
> **Prasyarat:** [Value vs Reference Semantics](01-value-vs-reference-semantics.md)
> **Baca ~22 menit**

---

## Pertanyaan yang memisahkan kandidat

> "Apa bedanya `func draw(_ s: any Shape)` dan `func draw(_ s: some Shape)`?"

Jawaban level junior: "`some` itu opaque type, `any` itu existential."
Jawaban level mid: "`some` di-spesialisasi jadi tipe konkret saat compile,
`any` dibungkus existential container dan dipanggil lewat witness table."
Jawaban level senior: menambahkan **berapa word** container-nya, kapan terjadi
alokasi heap, dan **kapan `any` justru pilihan yang benar**.

Artikel ini membawa kamu ke yang ketiga.

---

## 1. Tiga cara menulis "sesuatu yang Shape"

```swift
protocol Shape {
    var area: Double { get }
    func scaled(by f: Double) -> Self
}

struct Circle: Shape { var r: Double; ... }
struct Square: Shape { var side: Double; ... }
```

```swift
func a(_ s: any Shape)  { }   // existential  — satu fungsi, tipe ditentukan runtime
func b<T: Shape>(_ s: T) { }  // generic      — di-spesialisasi per tipe konkret
func c(_ s: some Shape) { }   // opaque param — gula sintaks untuk (b)
```

`some` di posisi parameter **identik** dengan generic `<T: Shape>` sejak SE-0341.
Perbedaan nyatanya cuma satu: dengan `<T>` kamu bisa menyebut `T` di tempat lain
(mis. `func pair<T: Shape>(_ a: T, _ b: T)` memaksa keduanya bertipe sama);
dengan `some` kamu tidak bisa, jadi dua parameter `some Shape` boleh beda tipe.

---

## 2. Apa yang terjadi pada `any Shape`

Dari dokumen ABI resmi, `swift/docs/ABI/TypeLayout.rst`:

> *"Values of protocol type, protocol composition type, or `Any` type are laid out
> using **existential containers**... If there is no class constraint... the existential
> container has to accommodate a value of arbitrary size and alignment. It does this
> using a **fixed-size buffer**, which is three pointers in size and pointer-aligned.
> This either directly contains the value, if its size and alignment are both less than
> or equal to the fixed-size buffer's, or contains a pointer to a side allocation owned
> by the existential container."*

```c
struct OpaqueExistentialContainer {
  void *fixedSizeBuffer[3];        // 24 byte di platform 64-bit
  Metadata *type;                  // 8 byte
  WitnessTable *witnessTables[N];  // 8 byte per protokol
};
```

Jadi `any Shape` = **40 byte** (24 + 8 + 8), berapa pun kecilnya tipe di dalamnya.

```
any Shape yang memuat Circle (8 byte)          any Shape yang memuat BigShape (40 byte)

┌─────────────────────────┐                    ┌─────────────────────────┐
│ buffer[0] = r           │  ← inline          │ buffer[0] = ptr ────────┼──▶ heap!
│ buffer[1] = (kosong)    │                    │ buffer[1] = (kosong)    │    ┌────────┐
│ buffer[2] = (kosong)    │                    │ buffer[2] = (kosong)    │    │ a,b,c, │
├─────────────────────────┤                    ├─────────────────────────┤    │ d,e    │
│ type metadata → Circle  │                    │ type metadata→BigShape  │    └────────┘
│ witness table → Shape   │                    │ witness table → Shape   │
└─────────────────────────┘                    └─────────────────────────┘
        0 alokasi                                     1 malloc + 1 free
```

**Garis batasnya 3 word = 24 byte.** Di bawah itu gratis. Di atasnya, setiap
pembungkusan jadi `any Shape` memicu `malloc`, dan setiap penyalinan memicu
`malloc` lagi (existential container punya value witness untuk copy/destroy).

Ini alasan konkret dan bisa diukur kenapa `[any Shape]` bisa jauh lebih lambat dari
`[Circle]` — bukan sekadar "dynamic dispatch lebih lambat".

### Kalau protokolnya `AnyObject`-constrained, ceritanya lain

Masih dari dokumen yang sama:

```c
struct ClassExistentialContainer {
  HeapObject *value;
  WitnessTable *witnessTables[N];
};
```

> *"Note that if no witness tables are needed... then the only element of the layout is
> the heap object pointer. This is ABI-compatible with `id` and `id <Protocol>` types
> in Objective-C."*

Jadi `any AnyObject & Shape` cuma 16 byte, tanpa fixed-size buffer, tanpa side allocation.
Kalau kamu memang butuh existential dan tipenya toh class, tambahkan `: AnyObject`
pada protokolnya — kamu menghemat 24 byte dan potensi alokasi.

---

## 3. Witness table: bagaimana `s.area` dipanggil

**Protocol Witness Table (PWT)** adalah tabel pointer fungsi: satu entri per requirement
protokol, berisi alamat implementasi milik tipe konkret.

```
WitnessTable untuk (Circle : Shape)
┌──────────────────────┬─────────────────────────┐
│ area (getter)        │ → Circle.area.getter    │
│ scaled(by:)          │ → Circle.scaled(by:)    │
└──────────────────────┴─────────────────────────┘
```

Saat kamu memanggil `s.area` di mana `s: any Shape`, runtime:

1. baca witness table pointer dari container,
2. ambil entri `area`,
3. panggil lewat pointer itu, dengan value witness table untuk tahu di mana nilainya
   (inline atau di side allocation).

Ini **tabel terpisah** dari vtable class. Class Swift punya vtable untuk method override;
conformance protokol punya witness table. Satu tipe bisa punya banyak witness table
(satu per protokol yang diadopsi), dan witness table dibuat untuk **pasangan**
(tipe, protokol) — bukan untuk tipe saja.

Untuk protokol dengan `associatedtype`, witness table juga memuat metadata untuk
associated type-nya, plus witness table untuk constraint-nya. Ini yang membuat
`Collection` mahal sebagai existential dan alasan historis kenapa protokol dengan
associated type dulu sama sekali tidak bisa dipakai sebagai `any` (sebelum SE-0309).

---

## 4. Apa yang terjadi pada `some Shape` / `<T: Shape>`

Compiler punya dua strategi:

**(a) Generic non-spesialisasi (default lintas modul).** Satu salinan kode, menerima
metadata + witness table sebagai **parameter tersembunyi**. Nilainya dioper lewat
alamat. Ini mirip existential dari sisi biaya — dan ini yang sering dilupakan orang
yang mengira `some` selalu gratis.

**(b) Spesialisasi.** Optimizer membuat salinan fungsi khusus untuk tipe konkret,
mengganti semua akses lewat witness table dengan panggilan langsung — lalu meng-*inline*-nya.

```swift
func totalArea<T: Shape>(_ shapes: [T]) -> Double {
    shapes.reduce(0) { $0 + $1.area }
}
totalArea([Circle(r: 1), Circle(r: 2)])
```

Setelah spesialisasi, `totalArea<Circle>` tidak lagi punya witness table sama sekali;
`$1.area` menjadi `$1.r * $1.r * .pi` yang di-inline. Nol dispatch, nol alokasi.

**Kapan spesialisasi terjadi:**

| Situasi | Bisa dispesialisasi? |
|---|---|
| Fungsi dan pemanggil di file yang sama | ✅ |
| Beda file, satu module, **Whole Module Optimization** aktif (default di Release) | ✅ |
| Beda module, fungsi bukan `@inlinable` | ❌ |
| Beda module, fungsi `@inlinable` (seperti stdlib) | ✅ |

Ini alasan stdlib Swift dipenuhi `@inlinable` — tanpa itu, `Array.map` dari module lain
tidak akan pernah bisa dispesialisasi untuk tipe kamu.

Ini juga alasan kenapa benchmark `some` vs `any` di Debug build **tidak berarti apa-apa**:
di `-Onone` tidak ada spesialisasi sama sekali.

---

## 5. Jadi kapan pakai yang mana?

Aturan yang bisa kamu pertahankan di depan interviewer:

### Pakai `some` / generic secara default

- Bisa dispesialisasi → nol overhead di Release.
- Menjaga **type identity**: `func pair<T: Shape>(_ a: T, _ b: T)` memaksa dua argumen
  bertipe sama. `any` tidak bisa menyatakan itu.
- Boleh punya `Self` requirement dan `associatedtype` tanpa batasan.

### Pakai `any` kalau kamu benar-benar butuh heterogenitas

```swift
var shapes: [any Shape] = [Circle(r: 1), Square(side: 2)]   // ✅ tidak ada pilihan lain
```

`[some Shape]` artinya "array berisi satu tipe konkret yang sama semua" — tidak bisa
menampung `Circle` dan `Square` sekaligus. Kalau kamu memang butuh koleksi campur,
`any` itu **jawaban yang benar**, bukan kompromi.

Kasus lain yang sah untuk `any`:
- Menyimpan tipe yang baru diketahui saat runtime (plugin, factory yang di-*register*).
- Memutus siklus dependensi tipe di API boundary.
- Type erasure yang disengaja untuk menjaga ABI stabil.

### Alternatif yang sering lebih baik dari keduanya: `enum`

Kalau himpunan tipenya **tertutup dan diketahui**, `enum` mengalahkan keduanya:

```swift
enum Shape {
    case circle(r: Double)
    case square(side: Double)

    var area: Double {
        switch self {
        case .circle(let r): return .pi * r * r
        case .square(let s): return s * s
        }
    }
}
```

Nol existential, nol witness table, exhaustiveness dicek compiler, dan layout-nya
sekecil case terbesar + tag. Banyak `[any Protocol]` di codebase iOS sebenarnya
adalah `enum` yang menyamar.

---

## 6. Dampak nyata di project `movie`

```swift
// movie/Services/MusicSearching.swift
protocol MusicSearching {
    func searchAlbums(term: String, limit: Int) async throws -> [Album]
    func searchTracks(term: String, limit: Int) async throws -> [Track]
    func albumTracks(albumID: Int) async throws -> [Track]
}

// movie/ViewModel/HomeViewModel.swift
final class HomeViewModel {
    private let service: MusicSearching                       // ← existential
    init(service: MusicSearching = iTunesService()) { ... }
}
```

`MusicSearching` di sini adalah existential (`any MusicSearching`, `any`-nya implisit
karena project ini belum pakai Swift 6 language mode yang mewajibkannya ditulis).

**Apakah ini masalah performa?** Tidak sama sekali, dan penting untuk bisa menjelaskan
kenapa:

1. `iTunesService` adalah `struct` kosong (hanya `let baseURL: URL` = 8 byte) →
   muat inline di fixed-size buffer, nol alokasi.
2. Yang dipanggil lewat witness table adalah fungsi yang melakukan **network request**.
   Overhead satu indirect call di sebelah latensi jaringan 200 ms itu nol koma nol.
3. Yang dibeli dengan existential ini adalah **kemampuan testing**: fake bisa disuntikkan
   lewat `init(service:)` tanpa membuat `HomeViewModel` jadi generic.

Kalau dibuat generic, biayanya nyata di sisi lain:

```swift
final class HomeViewModel<Service: MusicSearching> { ... }   // ⚠️
// sekarang setiap yang menyentuh HomeViewModel harus tahu tipe Service-nya,
// dan HomeViewController ikut jadi generic, dan seterusnya menular ke atas.
```

Itulah trade-off sebenarnya: **existential membeli boundary; generic membeli performa.**
Di boundary I/O, pilih boundary. Di inner loop, pilih performa.

Bandingkan dengan `LibraryStore` di project yang sama — `DECISIONS.md` mencatat jujur
bahwa ia *tidak* di balik protokol, jadi tidak bisa di-fake di test. Itu contoh nyata
harga yang dibayar saat boundary-nya tidak dibuat.

---

## 7. Cek pemahaman

**Q1.** Berapa `MemoryLayout<any Shape>.size` kalau `Shape` protokol tanpa constraint?
<details><summary>Jawaban</summary>

`40` byte: 3 word fixed-size buffer (24) + type metadata pointer (8) + 1 witness table (8).
Tambahkan 8 byte per protokol tambahan dalam komposisi.
</details>

**Q2.** Kenapa ini tidak compile?
```swift
protocol Container { associatedtype Item; func first() -> Item? }
let c: any Container = ...
let x = c.first()
```
<details><summary>Jawaban</summary>

Sampai Swift 5.6 ini ditolak mentah-mentah. Sejak SE-0309 + SE-0352 kamu bisa memegang
`any Container`, tapi memanggil requirement yang mengembalikan `Item` menghasilkan
tipe opaque yang tidak bisa dipakai kecuali di-*open* lewat generic function.
Solusi idiomatiknya: beri primary associated type (`Container<Int>`) atau operasikan
lewat fungsi generic yang membuka existential-nya.
</details>

**Q3.** Kamu punya protokol yang hanya diadopsi class. Perubahan satu kata apa yang
menghemat memori paling banyak?
<details><summary>Jawaban</summary>

Tambahkan `: AnyObject`. Existential-nya berubah dari `OpaqueExistentialContainer`
(40 byte + potensi side allocation) jadi `ClassExistentialContainer` (16 byte, cuma
pointer + witness table).
</details>

---

## Ringkasan

- `any P` = existential container: fixed-size buffer 3 word + metadata + witness table
  (≥ 40 byte). Nilai > 24 byte dipindahkan ke heap.
- Protokol `: AnyObject` memakai container yang jauh lebih ramping (pointer + witness table).
- Witness table = tabel pointer fungsi per pasangan (tipe, protokol), terpisah dari vtable.
- `some P` / `<T: P>` bisa dispesialisasi jadi nol-overhead — **tapi hanya kalau optimizer
  bisa melihat tipe konkretnya** (WMO atau `@inlinable`).
- Default ke `some`/generic. Pakai `any` saat butuh heterogenitas atau boundary.
  Pertimbangkan `enum` saat himpunan tipenya tertutup.
- Di boundary I/O (seperti `MusicSearching`), existential itu pilihan yang benar —
  yang dibeli adalah testability, dan biayanya nol dibanding latensi jaringan.

## Selanjutnya

→ [05 — Optional & Enum: Kenapa `Optional` Tidak Punya Overhead](05-optional-dan-enum-layout.md)

## Sumber

- `swift/docs/ABI/TypeLayout.rst` — Existential Container Layout
- `swift/docs/ABI/GenericSignature.md`
- [SE-0309 — Unlock existentials for all protocols](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md)
- [SE-0341 — Opaque parameter declarations](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0341-opaque-parameters.md)
- [WWDC22 — Embrace Swift generics](https://developer.apple.com/videos/play/wwdc2022/110352/)
