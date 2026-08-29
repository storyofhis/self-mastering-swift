---
title: "Optional & Enum: Kenapa `Optional` Tidak Punya Overhead"
category: "Swift Language & Internals"
status: complete
tags:
  - swift-mastering
  - swift/language
  - article
---

# Optional & Enum: Kenapa `Optional` Tidak Punya Overhead

> **Kategori:** Swift Language & Internals · **Level:** Menengah ·
> **Prasyarat:** [Value vs Reference Semantics](01-value-vs-reference-semantics.md)
> **Baca ~15 menit**

---

## Pertanyaan pembuka

```swift
MemoryLayout<Bool>.size            // 1
MemoryLayout<Bool?>.size           // ?
MemoryLayout<Bool??>.size          // ?
MemoryLayout<UnsafeRawPointer>.size   // 8
MemoryLayout<UnsafeRawPointer?>.size  // ?
```

Kalau `Optional` itu "enum dengan dua case", intuisi bilang butuh byte tambahan
untuk tag. Jawabannya: `1`, `1`, `8`. **Nol overhead.**

Mekanismenya bernama *extra inhabitants*, dan memahaminya membuat sejumlah perilaku
Swift yang tampak ajaib jadi masuk akal — termasuk kenapa `weak var x: Foo?` bisa
jadi `nil` tanpa flag, dan kenapa `Optional<Optional<Bool>>` masih 1 byte.

---

## 1. `Optional` bukan tipe istimewa

```swift
// bentuk konseptualnya:
enum Optional<Wrapped> {
    case none
    case some(Wrapped)
}

// deklarasi sebenarnya di stdlib/public/core/Optional.swift:
@frozen
public enum Optional<Wrapped: ~Copyable & ~Escapable>: ~Copyable, ~Escapable {
    case none
    case some(Wrapped)
}
```

(`~Copyable & ~Escapable` di sana artinya `Optional` **tidak mensyaratkan**
`Wrapped` bisa disalin atau di-escape — jadi ia juga bekerja untuk tipe noncopyable.
Untuk tipe biasa, perilakunya persis seperti bentuk konseptual di atas.)

Itu saja. `?` adalah gula sintaks untuk `Optional<T>`, `nil` adalah `.none`,
dan `if let` adalah gula untuk pattern matching `case .some(let x)`.

```swift
let a: Int? = 5
let b: Optional<Int> = .some(5)     // identik
let c: Int? = nil
let d: Optional<Int> = .none        // identik

switch a {
case .some(let value): print(value)
case .none: print("nil")
}
```

Yang istimewa bukan tipenya, tapi **cara compiler menyusun layout-nya**.

---

## 2. Extra inhabitants: bit pattern yang mustahil

Setiap tipe punya himpunan bit pattern yang **valid**. Bit pattern yang berada di
dalam ukurannya tapi tidak pernah bisa merepresentasikan nilai valid disebut
**extra inhabitant**, dan compiler memakainya sebagai tag gratis.

### `Bool`

`Bool` berukuran 1 byte = 256 bit pattern, tapi hanya `0x00` dan `0x01` yang valid.
Ada **254 extra inhabitant**.

```
Bool:            0x00 = false,  0x01 = true
Bool?:           0x00 = false,  0x01 = true,  0x02 = nil       ← gratis
Bool??:          0x00, 0x01, 0x02 = .some(nil), 0x03 = nil     ← masih gratis
```

Kamu bisa membungkus `Bool` dalam `Optional` sampai **254 lapis** sebelum ukurannya naik.

### Pointer

Pointer ke objek yang valid tidak pernah bernilai `0x0` (dan beberapa alamat rendah
lain tidak bisa dipetakan). Jadi:

```swift
MemoryLayout<UnsafeRawPointer>.size    // 8
MemoryLayout<UnsafeRawPointer?>.size   // 8  ← nil = alamat nol
MemoryLayout<AnyObject>.size           // 8
MemoryLayout<AnyObject?>.size          // 8
```

Inilah kenapa `weak var delegate: SomeDelegate?` tidak memakan byte lebih banyak dari
versi non-optional-nya, dan kenapa `class` reference selalu murah untuk dijadikan optional.

### `Int` — yang tidak punya extra inhabitant

`Int` 64-bit memakai **semua** 2⁶⁴ bit pattern. Tidak ada yang tersisa.

```swift
MemoryLayout<Int>.size      // 8
MemoryLayout<Int?>.size     // 9   ← butuh 1 byte tag terpisah
MemoryLayout<Int?>.stride   // 16  ← dibulatkan ke alignment
MemoryLayout<Int??>.size    // 9   ← tag byte masih punya sisa nilai
```

Perhatikan `stride` = 16. `[Int?]` dengan 1000 elemen memakan **16 KB**, sementara
`[Int]` memakan 8 KB. **Dua kali lipat.** Ini bukan detail akademis — ini alasan
konkret untuk tidak menyimpan `[Double?]` di data pipeline yang besar kalau kamu bisa
memakai sentinel atau bitmap terpisah.

---

## 3. Layout enum secara umum

Swift memakai tiga strategi, dipilih otomatis:

### (a) No-payload enum → integer sekecil mungkin

```swift
enum Direction { case north, south, east, west }
MemoryLayout<Direction>.size   // 1  (butuh 2 bit, dibulatkan ke 1 byte)

enum HomeState { case loading, loaded, empty, error(String) }   // ← ada payload, lihat (c)
```

### (b) Single-payload enum → pakai extra inhabitant payload-nya

```swift
enum MaybeBool { case none; case some(Bool) }
MemoryLayout<MaybeBool>.size   // 1 — persis seperti Bool?
```

Ini strategi yang sama dengan `Optional`, dan berlaku untuk enum apa pun yang punya
tepat satu case ber-payload.

### (c) Multi-payload enum → tag terpisah (atau spare bits)

```swift
enum SearchState { case idle, loading, loaded, empty, error(String) }
```

`String` di stdlib Swift berukuran 16 byte. Case tanpa payload butuh dibedakan dari
`.error`. Compiler mencoba memakai *spare bits* di dalam `String` dulu; kalau tidak
cukup, ia menambahkan tag byte.

Yang penting untuk dipahami: **ukuran enum = ukuran payload terbesar + tag (kalau perlu)**,
bukan jumlah semua payload. `SearchState` di project `movie` berukuran 16 byte —
sama seperti `String` di dalamnya.

```swift
// movie/ViewModel/SearchViewModel.swift
enum SearchState: Equatable {
    case idle
    case loading
    case loaded
    case empty
    case error(String)
}
```

Ini contoh pemakaian enum yang tepat: lima state yang **saling eksklusif**, dan compiler
memaksa `switch`-nya exhaustive. Bandingkan dengan alternatif yang buruk:

```swift
// ❌ struct dengan flag — bisa dalam state mustahil
struct BadState {
    var isLoading = false
    var isEmpty = false
    var error: String?
    // isLoading == true && isEmpty == true ⟶ apa artinya?
}
```

Enum menghilangkan seluruh kelas bug ini secara struktural. Itu argumen desainnya —
ukuran memori cuma bonus.

---

## 4. `indirect`: kapan enum butuh heap

Enum rekursif tidak punya ukuran terhingga tanpa indireksi:

```swift
indirect enum Expr {
    case number(Int)
    case add(Expr, Expr)     // tanpa `indirect`: ukuran tak hingga → compile error
}
```

`indirect` menaruh payload di heap box, jadi case-nya cuma menyimpan pointer.
Kamu bisa menaruhnya per-case untuk membatasi biayanya:

```swift
enum Expr {
    case number(Int)                    // inline
    indirect case add(Expr, Expr)       // hanya case ini yang di-box
}
```

---

## 5. `Optional` yang perlu diperhatikan di kode nyata

### 5a. `Optional` bersarang tidak otomatis rata

```swift
let dict: [String: Int?] = ["a": nil]
let value = dict["a"]        // Int?? — bukan Int?
```

`dict["a"]` mengembalikan `Optional` dari value type-nya. Karena value-nya sendiri
`Int?`, hasilnya `Int??`: "tidak ada key" vs "ada key tapi nilainya nil".
Dua hal berbeda, dan Swift jujur membedakannya.

```swift
if let inner = dict["a"], let actual = inner { ... }
// atau
if case .some(.some(let actual)) = dict["a"] { ... }
```

### 5b. Optional chaining selalu menaikkan satu lapis

```swift
struct A { var b: B? }
struct B { func f() -> Int { 1 } }

let a: A? = A(b: nil)
let x = a?.b?.f()      // Int? — bukan Int??; chaining meratakan otomatis
```

### 5c. `try?` dan `as?` menghasilkan Optional yang bisa menelan informasi

```swift
let result = try? decoder.decode(Track.self, from: data)   // Track?
// error-nya HILANG. Kamu tidak tahu kenapa gagal.
```

Di project `movie`, `iTunesService.fetch` sengaja **tidak** memakai `try?` —
error di-*map* jadi tipe domain supaya UI bisa menampilkan pesan yang berbeda:

```swift
do {
    return try JSONDecoder().decode(Response.self, from: data)
} catch {
    throw MusicServiceError.decodingFailed
}
```

### 5d. `!` — tiga jenis, tidak semuanya sama buruk

```swift
let x = optional!                     // force unwrap: crash kalau nil
var v: UIView!                        // implicitly unwrapped optional (IUO)
let cell = ... as! AlbumPosterCell    // force cast
```

IUO punya tempat yang sah: property yang dijamin terisi sebelum dipakai tapi tidak bisa
diisi di `init` — seperti `@IBOutlet`, atau `collectionView` di
`HomeViewController` yang dibuat di `viewDidLoad`. Force cast pada `dequeueReusableCell`
juga umum diterima: kalau tipenya salah, itu bug programmer yang harus meledak keras
saat development, bukan gagal diam-diam.

Yang tidak bisa dibela: `!` pada data yang datang dari luar (JSON, user input, jaringan).

---

## 6. Cek pemahaman

**Q1.** Kenapa `MemoryLayout<Int8?>.size == 2` tapi `MemoryLayout<Bool?>.size == 1`?
<details><summary>Jawaban</summary>

`Int8` memakai seluruh 256 bit pattern-nya (−128…127) → tidak ada extra inhabitant →
butuh tag byte terpisah. `Bool` hanya memakai 2 dari 256 → 254 extra inhabitant tersedia.
</details>

**Q2.** Berapa `MemoryLayout<Direction?>.size` untuk `enum Direction { case n, s, e, w }`?
<details><summary>Jawaban</summary>

`1`. `Direction` butuh 2 bit dari 1 byte → masih ada 252 extra inhabitant.
`nil` mengambil salah satunya.
</details>

**Q3.** Kenapa `[Int?]` bisa dua kali lebih besar dari `[Int]`, padahal `Int?` cuma
"satu byte lebih besar"?
<details><summary>Jawaban</summary>

Karena array memakai `stride`, bukan `size`. `Int?` punya `size` 9 tapi `alignment` 8,
sehingga `stride`-nya dibulatkan jadi 16. Setiap elemen memakan 16 byte, 7 di antaranya
padding.
</details>

---

## Ringkasan

- `Optional<T>` adalah enum biasa; keajaibannya ada di layout, bukan di tipe.
- **Extra inhabitants**: bit pattern yang tidak valid dipakai sebagai tag gratis.
  `Bool?`, pointer?, dan enum kecil? = nol overhead.
- Tipe yang memakai seluruh ruang bit-nya (`Int`, `Double`) butuh tag byte terpisah →
  `size` +1, tapi `stride` bisa +8 karena padding.
- Ukuran enum = payload terbesar + tag, bukan jumlah semua payload.
- `indirect` memindahkan payload ke heap; bisa diterapkan per-case.
- Enum dengan associated value adalah cara struktural menghapus state mustahil —
  argumen utamanya desain, bukan memori.

## Selanjutnya

→ [06 — Error Handling: `throws`, `Result`, dan `typed throws`](06-error-handling.md)
→ [Concurrency: Dari GCD ke Swift Concurrency](../concurrency/01-gcd-ke-swift-concurrency.md)

## Sumber

- `swift/stdlib/public/core/Optional.swift`
- `swift/docs/ABI/TypeLayout.rst` — bagian "Fragile Enum Layout" & extra inhabitants
- Project `movie`: `SearchState`, `HomeState`, `MusicServiceError`
