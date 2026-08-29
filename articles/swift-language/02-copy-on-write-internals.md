---
title: "Copy-on-Write: Membedah `Array` Langsung dari Source stdlib"
category: "Swift Language & Internals"
status: complete
tags:
  - swift-mastering
  - swift/language
  - article
---

# Copy-on-Write: Membedah `Array` Langsung dari Source stdlib

> **Kategori:** Swift Language & Internals · **Level:** Menengah ·
> **Prasyarat:** [Value vs Reference Semantics](01-value-vs-reference-semantics.md)
> **Baca ~20 menit**

---

## Paradoks yang mau dipecahkan

`Array` adalah `struct`. Value type. Menyalinnya berarti menyalin isinya.

Tapi:

```swift
let million = Array(repeating: 0, count: 1_000_000)   // 8 MB
let copy = million                                     // ...8 MB lagi?
```

Kalau benar-benar disalin, satu baris itu memakan 8 MB dan waktu `memcpy` yang nyata.
Padahal `let copy = million` di Swift praktis **gratis** — beberapa nanodetik.

Jawabannya: Swift **berbohong dengan jujur**. Secara semantik `copy` adalah array
terpisah. Secara implementasi, keduanya menunjuk buffer heap yang sama, sampai ada yang
mencoba menulis. Itulah **copy-on-write** (COW).

Kalau kamu bisa menjelaskan mekanismenya sampai ke `isKnownUniquelyReferenced`,
kamu sudah melewati pertanyaan memori level mid di kebanyakan interview iOS.

---

## 1. Struktur nyata `Array`

`Array` bukan berisi elemen. Ia berisi **satu pointer** ke buffer heap.

```swift
MemoryLayout<[Int]>.size   // 8 — satu word, apa pun jumlah elemennya
```

```
var a = [1, 2, 3]
var b = a

  a ┌──────────┐
    │ _buffer  │────┐
    └──────────┘    │
                    ▼   (heap)
  b ┌──────────┐  ┌─────────────┬────────────┬───────┬──────────┬───┬───┬───┐
    │ _buffer  │─▶│ metadata    │ refCount=2 │ count │ capacity │ 1 │ 2 │ 3 │
    └──────────┘  └─────────────┴────────────┴───────┴──────────┴───┴───┴───┘
                  └─── HeapObject header ───┘└─ ArrayBody ─┘└─ elemen ─┘
```

Menyalin `Array` = menyalin 8 byte pointer + satu `retain` (increment refcount).
Itu saja. Elemen tidak disentuh.

---

## 2. Titik keputusannya: `isKnownUniquelyReferenced`

Seluruh trik COW bertumpu pada satu fungsi di stdlib:

```swift
// swift/stdlib/public/core/ManagedBuffer.swift
@inlinable
public func isKnownUniquelyReferenced<T: AnyObject>(_ object: inout T) -> Bool {
  return _isUnique(&object)
}
```

Dokumentasi resminya di source itu menyebutkan pola pemakaiannya secara eksplisit:

> *"The `isKnownUniquelyReferenced(_:)` function is useful for implementing the
> copy-on-write optimization for the deep storage of value types:"*
>
> ```swift
> mutating func update(withValue value: T) {
>     if !isKnownUniquelyReferenced(&myStorage) {
>         myStorage = self.copiedStorage()
>     }
>     myStorage.update(withValue: value)
> }
> ```

Tiga catatan penting dari doc comment yang sama, yang sering jadi bahan follow-up:

1. **Hanya menghitung strong reference.** Weak dan unowned diabaikan — objek dengan
   1 strong + 3 weak tetap dilaporkan unik. (Masuk akal: weak reference tidak bisa
   memutasi apa pun tanpa dipromosikan dulu jadi strong.)
2. **Weak/unowned yang dioper sebagai argumen selalu menghasilkan `false`.**
3. **Tidak thread-safe sebagai alat sinkronisasi.** Source-nya menulis:
   *"If the instance passed as `object` is being accessed by multiple threads
   simultaneously, this function may still return `true`."* Makanya ia hanya boleh
   dipanggil dari `mutating` method yang aksesnya sudah dijamin eksklusif —
   dan di Swift itu dijamin oleh *exclusive access to memory*, bukan oleh lock.

`inout` di signature-nya **bukan** karena fungsi ini memutasi apa pun. Doc-nya menyebut
itu *"an implementation artifact"* — `inout` memaksa akses eksklusif, sehingga compiler
tidak boleh menyimpan salinan sementara yang akan menaikkan refcount dan merusak hasilnya.

---

## 3. Bagaimana `Array` benar-benar memakainya

Di `swift/stdlib/public/core/Array.swift`, setiap operasi mutasi dimulai dengan
memanggil `_makeMutableAndUnique()`:

```swift
@inlinable
@_semantics("array.make_mutable")
@_effects(notEscaping self.**)
internal mutating func _makeMutableAndUnique() {
  if _slowPath(!_buffer.beginCOWMutation()) {
    _buffer = _buffer._consumeAndCreateNew()
  }
}
```

Bacanya:

- `_buffer.beginCOWMutation()` → "apakah buffer ini milikku sendiri?"
  (di baliknya: `isUniquelyReferenced` + penandaan mulai-mutasi untuk optimizer).
- Kalau **tidak** unik (`_slowPath` menandai ini jalur yang jarang, supaya branch
  predictor & optimizer mengoptimalkan jalur unik), buffer diganti buffer baru
  hasil salinan: `_consumeAndCreateNew()`.
- Setelah itu, mutasi berjalan pada buffer yang dijamin milik sendiri.

Kamu bisa melihat pemanggilannya bertebaran di operasi-operasi mutasi:

```
Array.swift:767   _makeMutableAndUnique()   // makes the array native, too
Array.swift:1317  _makeMutableAndUnique()
Array.swift:1348  _makeMutableAndUnique()
Array.swift:1754  _makeMutableAndUnique()
Array.swift:1822  _makeMutableAndUnique()
```

Ada pasangannya juga:

```swift
/// Marks the end of an Array mutation.
///
/// After a call to `_endMutation` the buffer must not be mutated until a call
/// to `_makeMutableAndUnique`.
internal mutating func _endMutation() {
  _buffer.endCOWMutation()
}
```

Pasangan `begin_cow_mutation` / `end_cow_mutation` ini adalah instruksi SIL nyata —
mereka memberi tahu optimizer batas "wilayah di mana buffer dijamin unik", supaya
pengecekan keunikan yang berulang di dalam loop bisa di-*hoist* keluar loop.
Itu sebabnya loop `for i in 0..<n { arr[i] += 1 }` tidak melakukan `n` kali
pengecekan refcount.

Dan `append` punya varian sendiri yang menggabungkan cek keunikan dengan pertumbuhan
kapasitas, supaya tidak menyalin dua kali:

```swift
internal mutating func _makeUniqueAndReserveCapacityIfNotUnique() {
  if _slowPath(!_buffer.beginCOWMutation()) {
    _createNewBuffer(bufferIsUnique: false,
                     minimumCapacity: count &+ 1,
                     growForAppend: true)
  }
}
```

---

## 4. Melihat COW terjadi dengan mata sendiri

```swift
func addressOfBuffer<T>(_ array: [T]) -> String {
    array.withUnsafeBufferPointer { String(describing: $0.baseAddress) }
}

var a = [1, 2, 3]
var b = a
print(addressOfBuffer(a))   // 0x0000600000...
print(addressOfBuffer(b))   // 0x0000600000...  ← SAMA

b.append(4)
print(addressOfBuffer(a))   // 0x0000600000...  ← a tetap
print(addressOfBuffer(b))   // 0x0000600001...  ← b pindah buffer
```

Salinan baru terjadi tepat di `append`, bukan di `let b = a`. Itulah "on write".

---

## 5. Menulis COW sendiri

Kamu butuh ini setiap kali membuat value type yang menyimpan data besar atau
membungkus resource — dan ini soal live-coding yang cukup sering keluar.

```swift
struct Matrix {
    // Storage sengaja `final class` — kita butuh refcount untuk cek keunikan.
    private final class Storage {
        var values: [Double]
        let rows: Int, cols: Int

        init(rows: Int, cols: Int, values: [Double]) {
            self.rows = rows; self.cols = cols; self.values = values
        }

        func copy() -> Storage {
            Storage(rows: rows, cols: cols, values: values)
        }
    }

    private var storage: Storage

    init(rows: Int, cols: Int) {
        storage = Storage(rows: rows, cols: cols,
                          values: Array(repeating: 0, count: rows * cols))
    }

    // Satu-satunya gerbang menuju mutasi.
    private mutating func ensureUnique() {
        if !isKnownUniquelyReferenced(&storage) {
            storage = storage.copy()
        }
    }

    subscript(row: Int, col: Int) -> Double {
        get { storage.values[row * storage.cols + col] }   // baca: TIDAK menyalin
        set {
            ensureUnique()                                  // tulis: salin kalau perlu
            storage.values[row * storage.cols + col] = newValue
        }
    }
}
```

Tiga hal yang sering salah saat menulis ini:

| Kesalahan | Akibat |
|---|---|
| Memanggil `ensureUnique()` di `get` juga | Setiap pembacaan menyalin. COW-nya sia-sia. |
| Lupa `mutating` di `ensureUnique()` | Tidak compile — dan itu bagus, compiler menolongmu. |
| `Storage` dibuat `struct` | `isKnownUniquelyReferenced` butuh `AnyObject`. Tidak compile. |
| Menyimpan `storage` di variabel lokal sebelum cek | Refcount naik → selalu dianggap tidak unik → selalu menyalin. |

Yang terakhir itu jebakan paling halus:

```swift
private mutating func ensureUniqueWRONG() {
    let s = storage                                  // ⚠️ refcount jadi 2
    if !isKnownUniquelyReferenced(&storage) { ... }  // selalu false
}
```

---

## 6. Tipe apa saja yang COW?

Semua tipe koleksi stdlib: `Array`, `ContiguousArray`, `ArraySlice`, `Dictionary`,
`Set`, `String`, dan `Data` (Foundation).

Yang **bukan** COW: struct kamu sendiri, kecuali kamu mengimplementasikannya.
Struct dengan lima `Int` benar-benar disalin byte per byte setiap kali dioper —
dan itu tidak apa-apa, karena 40 byte `memcpy` lebih murah daripada satu alokasi heap.

> **Aturan praktis:** COW baru menguntungkan kalau biaya penyalinan > biaya
> satu alokasi heap + refcount traffic. Untuk struct kecil, COW justru memperlambat.

---

## 7. Jebakan performa yang nyata di app

### 7a. `append` di dalam loop tanpa `reserveCapacity`

```swift
var result: [Track] = []
for t in tracks { result.append(transform(t)) }
```

`Array` tumbuh dengan menggandakan kapasitas — jadi ini `O(n)` amortized, aman.
Tapi kalau kamu sudah tahu jumlahnya, satu baris ini menghapus semua realokasi:

```swift
var result: [Track] = []
result.reserveCapacity(tracks.count)
```

### 7b. Mutasi array yang tersimpan di property class

```swift
final class ViewModel {
    var rows: [TrackRowViewModel] = []

    func markSaved(at index: Int) {
        rows[index] = ...      // ✅ satu akses, buffer unik, in-place
    }

    func markSavedSlow(at index: Int) {
        var copy = rows        // ⚠️ refcount = 2
        copy[index] = ...      // → menyalin seluruh buffer
        rows = copy
    }
}
```

Di project `movie`, `SearchViewModel` menulis langsung ke elemen — ini bentuk yang benar:

```swift
func toggleSave(at index: Int) {
    guard tracks.indices.contains(index) else { return }
    let isNowSaved = library.toggleSave(tracks[index])
    rows[index] = TrackRowViewModel(track: tracks[index], isSaved: isNowSaved)
}
```

Catatan halus: `rows` di sana punya `didSet { onRowsChange?() }`. Property observer
pada value type **tidak** mematikan COW in-place untuk subscript assignment
(compiler memakai `modify` coroutine accessor), tapi ia memang memicu `didSet`
setiap penulisan elemen — jadi kalau kamu memutasi 50 baris dalam loop, UI-nya
akan diberi tahu 50 kali. Kumpulkan perubahan dulu, tulis sekali.

### 7c. `ArraySlice` menahan seluruh buffer

```swift
let huge = Array(0..<1_000_000)
let firstTen = huge[0..<10]      // ArraySlice — masih memegang buffer 8 MB!
```

`ArraySlice` berbagi buffer dengan induknya. Selama slice hidup, 8 MB itu tidak
dibebaskan. Kalau slice-nya mau disimpan lama:

```swift
let firstTen = Array(huge[0..<10])   // ✅ buffer baru, kecil
```

Slice juga tidak me-reindex: `firstTen.startIndex` bisa bukan `0`. Selalu pakai
`indices`, `first`, atau konversi ke `Array` — jangan pakai literal `0`.

---

## 8. Cek pemahaman

**Q1.** Berapa kali buffer disalin?
```swift
var a = [1, 2, 3]
var b = a
a.append(4)
b.append(5)
```
<details><summary>Jawaban</summary>

**Satu kali.** Saat `a.append(4)`, refcount = 2 → `a` mendapat buffer baru.
Sekarang `b` memegang buffer lama sendirian (refcount = 1), jadi `b.append(5)`
memutasi in-place tanpa menyalin.
</details>

**Q2.** Kenapa `isKnownUniquelyReferenced` mengambil `inout` padahal tidak memutasi?
<details><summary>Jawaban</summary>

Untuk memaksa **exclusive access**. Kalau parameternya biasa, compiler bisa membuat
salinan sementara yang menaikkan refcount, dan hasilnya selalu `false`.
Source stdlib menyebutnya *"an implementation artifact"*.
</details>

**Q3.** Struct ini COW atau tidak?
```swift
struct Wrapper { var items: [Int] }
```
<details><summary>Jawaban</summary>

`Wrapper` sendiri tidak mengimplementasikan COW — tapi ia mewarisi perilakunya dari
`items`. Menyalin `Wrapper` menyalin pointer `Array`-nya + retain. Mutasi `items`
memicu COW milik `Array`. Efeknya sama, biayanya sama.
</details>

---

## Ringkasan

- `Array` adalah struct berisi satu pointer; menyalinnya = pointer + retain.
- COW dipicu oleh `isKnownUniquelyReferenced` — hanya strong reference yang dihitung,
  bukan alat sinkronisasi.
- `Array._makeMutableAndUnique()` adalah gerbang tunggal sebelum setiap mutasi;
  `begin/end_cow_mutation` membiarkan optimizer mengangkat cek keunikan keluar loop.
- Untuk COW buatan sendiri: `final class Storage`, `ensureUnique()` hanya di jalur tulis,
  dan jangan pernah menyimpan `storage` ke variabel lokal sebelum mengeceknya.
- `ArraySlice` menahan buffer induknya — bungkus dengan `Array(...)` kalau disimpan lama.

## Selanjutnya

→ [03 — ARC, Retain Cycle, dan Kapan `weak` Bukan Jawabannya](03-arc-dan-retain-cycle.md)

## Sumber

- `swift/stdlib/public/core/Array.swift` — `_makeMutableAndUnique`, `_endMutation`,
  `_makeUniqueAndReserveCapacityIfNotUnique`
- `swift/stdlib/public/core/ManagedBuffer.swift` — `isKnownUniquelyReferenced` + doc comment
- `swift/docs/Arrays.md`, `swift/docs/ArrayImplementation.png`
- [WWDC18 — Embracing Algorithms](https://developer.apple.com/videos/play/wwdc2018/223/)
