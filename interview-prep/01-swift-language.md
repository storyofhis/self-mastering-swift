---
title: "Interview Prep 01 — Swift Language"
category: "Interview Prep"
status: complete
tags:
  - swift-mastering
  - interview-prep
  - interview
---

# Interview Prep 01 — Swift Language

> 🟢 Junior · 🟡 Mid · 🔴 Senior — [cara pakai](README.md)

---

## 🟢 1. Kapan pakai `struct`, kapan `class`?

<details><summary>Jawaban model</summary>

**Default `struct`.** Naik ke `class` hanya kalau butuh salah satu dari:
identitas (dua objek berisi sama tetap berbeda), shared mutable state yang disengaja,
inheritance dari kelas Obj-C, atau `deinit`.

**Mekanismenya:** `struct` disimpan inline di dalam pemuatnya, tanpa header dan tanpa
reference counting. `class` selalu satu pointer 8 byte ke objek heap dengan header
16 byte (type metadata + refcount).

**Trade-off:** value type menghilangkan seluruh kelas bug aliasing, tapi kalau
struct-nya besar dan sering dioper, penyalinan jadi biaya — meski untuk koleksi
stdlib, copy-on-write sudah menyelesaikannya.
</details>

**Follow-up yang biasanya menyusul:**
- *"Kalau struct-nya besar, bukankah menyalinnya mahal?"* → jelaskan COW dan bahwa
  untuk struct sendiri kamu harus mengimplementasikannya.
- *"Kenapa `actor` tidak bisa `struct`?"* → isolation butuh identitas.

📖 [Value vs Reference Semantics](../articles/swift-language/01-value-vs-reference-semantics.md)

---

## 🟢 2. Apa itu Optional, dan apa bedanya `if let`, `guard let`, `??`, `!`?

<details><summary>Jawaban model</summary>

`Optional<Wrapped>` adalah enum dengan dua case: `.none` dan `.some(Wrapped)`.
`?` adalah gula sintaks untuknya, `nil` adalah `.none`.

- `if let` — unwrap ke scope di dalam blok.
- `guard let` — unwrap ke scope **setelahnya**, memaksa keluar lebih awal.
  Pilih ini untuk validasi di awal fungsi; ia menjaga jalur sukses tetap tidak ter-indent.
- `??` — nilai default.
- `!` — force unwrap; crash kalau `nil`.

Kalau nilainya wajib ada untuk seluruh sisa fungsi, `guard let`. Kalau hanya
dibutuhkan di satu cabang, `if let`.
</details>

**Follow-up:** *"Kapan `!` bisa dibenarkan?"* → `@IBOutlet`, `dequeueReusableCell as!`,
dan property yang dijamin terisi di `viewDidLoad`. Tidak pernah untuk data dari luar.

---

## 🟡 3. Berapa `MemoryLayout<Bool?>.size`? Kenapa?

<details><summary>Jawaban model</summary>

**1 byte.** Sama dengan `Bool`.

`Bool` memakai 1 byte tapi hanya 2 dari 256 bit pattern yang valid (`0x00`, `0x01`).
254 sisanya adalah **extra inhabitants** — bit pattern mustahil yang dipakai compiler
sebagai tag. `nil` mengambil `0x02`.

Bandingkan `Int?` = 9 byte (`stride` 16), karena `Int` memakai seluruh 2⁶⁴ bit
pattern-nya sehingga butuh tag byte terpisah.
</details>

**Follow-up:** *"Jadi `[Int?]` dengan 1000 elemen memakan berapa?"* → 16 KB, karena
array memakai `stride` (16), bukan `size` (9). Dua kali lipat `[Int]`.

📖 [Optional & Enum Layout](../articles/swift-language/05-optional-dan-enum-layout.md)

---

## 🟡 4. Apa bedanya `some P` dan `any P`?

<details><summary>Jawaban model</summary>

`some P` = **opaque type**: satu tipe konkret, ditentukan saat compile, disembunyikan
dari pemanggil. Di posisi parameter, ia identik dengan generic `<T: P>`.

`any P` = **existential**: kotak yang bisa memuat tipe konkret apa pun yang
mengadopsi `P`, ditentukan saat runtime.

**Mekanismenya:** `any P` dibungkus existential container — fixed-size buffer 3 word
(24 byte) + type metadata + witness table, jadi minimal 40 byte. Nilai yang lebih
besar dari 24 byte dipindahkan ke alokasi heap terpisah. Pemanggilan method lewat
witness table (indirect).

`some P` bisa **dispesialisasi** optimizer jadi tipe konkret — witness table hilang,
panggilan jadi langsung dan bisa di-inline.
</details>

**Follow-up:**
- *"Jadi `any` selalu lebih lambat?"* → tidak selalu berarti. Kalau tipenya ≤ 24 byte
  tidak ada alokasi, dan kalau method-nya melakukan I/O, satu indirect call itu nol.
  Dan spesialisasi `some` **hanya terjadi** kalau optimizer melihat tipe konkretnya —
  lintas module tanpa `@inlinable`, tidak terjadi.
- *"Kapan `any` justru pilihan yang benar?"* → koleksi heterogen (`[any Shape]`),
  tipe yang baru diketahui runtime, dan boundary API untuk testability.

📖 [Existential vs Generics](../articles/swift-language/04-protocol-existential-vs-generics.md)

---

## 🟡 5. Apa itu protocol witness table?

<details><summary>Jawaban model</summary>

Tabel pointer fungsi yang dibuat compiler untuk setiap **pasangan** (tipe, protokol).
Satu entri per requirement protokol, berisi alamat implementasi milik tipe itu.

Ia terpisah dari **vtable** class (yang untuk method override). Satu tipe bisa punya
banyak witness table — satu per protokol yang diadopsi.

Untuk protokol dengan `associatedtype`, witness table juga memuat metadata associated
type dan witness table untuk constraint-nya.
</details>

---

## 🟡 6. Kenapa `mutating` hanya ada di value type?

<details><summary>Jawaban model</summary>

Karena method `mutating` sebenarnya menerima `self` sebagai `inout` — ia menulis balik
ke lokasi penyimpanan asli. Value type tidak punya identitas, jadi "mengubah nilainya"
berarti mengganti isi di tempat ia disimpan.

Class tidak butuh itu: method-nya memutasi lewat pointer, dan `self` adalah referensi
yang tidak berubah.

Ini juga menjelaskan kenapa `let s = SomeStruct()` melarang `s.mutate()` — `inout`
butuh akses tulis, dan `let` tidak menyediakannya.
</details>

---

## 🟡 7. Apa yang terjadi pada `self` di dalam closure?

<details><summary>Jawaban model</summary>

Closure adalah reference type dan meng-capture **variabel**, bukan salinan nilainya.
Untuk `self` di dalam class, itu berarti strong reference — closure menahan `self` hidup.

```swift
var counter = 0
let f = { counter += 1 }
f(); f()
print(counter)   // 2 — bukan 0
```

Compiler memindahkan `counter` ke heap box supaya closure dan scope asli menunjuk
storage yang sama. Untuk snapshot, pakai capture list: `{ [counter] in ... }`.
</details>

**Follow-up:** *"Kapan `[weak self]` TIDAK diperlukan?"* → kalau tidak ada jalur
`self → … → closure → self`. `array.forEach { self.f($0) }` (non-escaping) dan
`URLSession` completion (yang menahan closure adalah session, bukan self) tidak perlu.

📖 [ARC & Retain Cycle](../articles/swift-language/03-arc-dan-retain-cycle.md)

---

## 🔴 8. Apa itu copy-on-write, dan bagaimana kamu mengimplementasikannya?

<details><summary>Jawaban model</summary>

COW = value type menunda penyalinan sampai ada mutasi, dengan berbagi buffer heap
selama tidak ada yang menulis.

Mekanismenya bertumpu pada `isKnownUniquelyReferenced(&storage)` — kalau storage-nya
punya lebih dari satu strong reference, salin dulu sebelum menulis.

```swift
struct Matrix {
    private final class Storage { var values: [Double]; func copy() -> Storage { ... } }
    private var storage: Storage

    private mutating func ensureUnique() {
        if !isKnownUniquelyReferenced(&storage) { storage = storage.copy() }
    }
    subscript(i: Int) -> Double {
        get { storage.values[i] }            // baca: tidak menyalin
        set { ensureUnique(); storage.values[i] = newValue }
    }
}
```

Di stdlib, `Array` melakukannya lewat `_makeMutableAndUnique()` yang dipanggil
di awal setiap operasi mutasi.
</details>

**Follow-up:** *"Kenapa `Storage` harus `class`?"* → `isKnownUniquelyReferenced`
butuh `AnyObject`; refcount hanya ada pada reference type.
*"Kenapa `inout` di signature-nya?"* → untuk memaksa akses eksklusif; tanpa itu
compiler bisa membuat salinan sementara yang menaikkan refcount.

📖 [Copy-on-Write Internals](../articles/swift-language/02-copy-on-write-internals.md)

---

## 🔴 9. Kenapa stdlib Swift penuh `@inlinable` dan `@frozen`?

<details><summary>Jawaban model</summary>

Karena **library evolution**. Tanpa keduanya, module yang dikompilasi terpisah harus
mengakses tipe lain lewat indireksi supaya ABI-nya tetap stabil saat library diperbarui.

- `@inlinable` mengekspos body fungsi ke module pemanggil → generic bisa dispesialisasi
  dan fungsinya bisa di-inline lintas module.
- `@frozen` menjanjikan layout tipe tidak akan berubah → pemanggil boleh mengasumsikan
  ukuran dan offset field-nya saat compile, bukan menanyakannya ke runtime.

Harganya: keduanya adalah **janji ABI**. Setelah dirilis, body `@inlinable` dan layout
`@frozen` tidak bisa diubah tanpa merusak binary yang sudah ada.
</details>

---

## 🔴 10. `Self` (kapital) vs `self` — dan kenapa protokol dengan `Self` requirement dulu tidak bisa jadi existential?

<details><summary>Jawaban model</summary>

`self` = instance. `Self` = tipe dinamis dari instance itu.

```swift
protocol Copyable { func copy() -> Self }
```

Untuk existential `any Copyable`, `Self` tidak diketahui saat compile — compiler tidak
tahu tipe apa yang dikembalikan `copy()`, jadi hasilnya tidak bisa dipakai untuk apa pun.
Itu sebabnya protokol dengan `Self` requirement dulu ditolak sebagai tipe existential.

Sejak SE-0309, kamu bisa memegang `any Copyable`, tapi memanggil requirement yang
mengembalikan `Self` menghasilkan nilai bertipe opaque yang harus "dibuka" lewat
fungsi generic sebelum bisa dipakai.
</details>

---

## Latihan cepat (5 menit, jawab lisan)

1. Apa yang dicetak? `struct S { var a = [1,2] }; var x = S(); var y = x; y.a.append(3); print(x.a.count)`
2. `MemoryLayout<AnyObject?>.size`?
3. Kenapa `weak var d: SomeProtocol?` tidak compile kalau protokolnya bukan `: AnyObject`?
4. Sebutkan tiga kasus di mana `enum` lebih baik dari `[any Protocol]`.
5. Apa bedanya `Equatable` sintesis dan `Hashable` sintesis untuk diffable data source?

<details><summary>Kunci</summary>

1. `2` — `Array` COW, `x` tidak tersentuh.
2. `8` — pointer punya extra inhabitant (alamat nol), jadi `nil` gratis.
3. `weak` butuh reference counting; protokol tanpa `: AnyObject` bisa diadopsi struct.
4. Himpunan tipe tertutup; butuh exhaustive `switch`; ingin menghindari alokasi/existential container.
5. Keduanya disintesis dari semua stored property. Untuk diffable, itu berarti perubahan
   satu field membuat item dianggap **baru** (delete+insert), bukan diperbarui — kalau
   mau animasi yang benar, implementasikan `Hashable` berdasarkan `id` saja lalu pakai
   `reconfigureItems`.
</details>
