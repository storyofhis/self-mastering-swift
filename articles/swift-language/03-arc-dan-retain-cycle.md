---
title: "ARC, Retain Cycle, dan Kapan `weak` Bukan Jawabannya"
category: "Swift Language & Internals"
status: complete
tags:
  - swift-mastering
  - swift/language
  - article
---

# ARC, Retain Cycle, dan Kapan `weak` Bukan Jawabannya

> **Kategori:** Swift Language & Internals · **Level:** Menengah ·
> **Prasyarat:** [Value vs Reference Semantics](01-value-vs-reference-semantics.md)
> **Baca ~22 menit**

---

## Satu kalimat yang harus kamu bisa ucapkan

> "ARC bukan garbage collector. Ia bukan proses yang jalan di belakang layar mencari
> objek tak terpakai. Ia adalah `retain`/`release` yang **disisipkan compiler** di
> waktu kompilasi, dan karena itu ia deterministik — dan karena itu pula ia tidak bisa
> mendeteksi siklus."

Kalimat itu memuat tiga fakta sekaligus, dan hampir semua pertanyaan ARC di interview
adalah turunan dari salah satunya.

---

## 1. Tiga refcount, bukan satu

Kebanyakan penjelasan bilang "objek punya satu refcount". Source-nya bilang lain.
Dari `swift/stdlib/public/SwiftShims/swift/shims/RefCount.h`:

> *"An object conceptually has three refcounts. These refcounts are stored either
> 'inline' in the field following the isa or in a 'side table entry' pointed to by
> the field following the isa."*

| Refcount | Menghitung | Saat mencapai nol |
|---|---|---|
| **strong** | strong reference | `deinit` dipanggil; baca `unowned` jadi error, baca `weak` jadi `nil` |
| **unowned** | unowned reference (+1 mewakili semua strong) | **alokasi memorinya dibebaskan** |
| **weak** | weak reference (+1 mewakili semua unowned) | side table entry dibebaskan |

Ini menjelaskan hal yang membingungkan banyak orang: **`deinit` dan pembebasan memori
adalah dua peristiwa berbeda.**

```
strong RC → 0   ⇒  deinit() jalan, objek "mati" secara logis  (state: DEINITED)
unowned RC → 0  ⇒  memorinya benar-benar di-free                (state: FREED)
weak RC → 0     ⇒  side table entry di-free
```

Selama masih ada `unowned` yang menunjuk objek yang sudah di-`deinit`, alokasinya
**belum** dibebaskan — supaya akses `unowned` bisa menghasilkan crash yang terdefinisi
(*"attempted to read an unowned reference but object was already deallocated"*),
bukan use-after-free acak.

---

## 2. Side table: kenapa `weak` lebih mahal dari `unowned`

Masih dari `RefCount.h`:

> *"Objects initially start with no side table. They can gain a side table when:
> a weak reference is formed... Gaining a side table entry is a one-way operation;
> an object with a side table entry never loses it. This prevents some thread races."*
>
> *"Strong and unowned variables point at the object. Weak variables point at the
> object's side table."*

Layout-nya, langsung dari source:

```c
HeapObject {
  isa
  InlineRefCounts {
    atomic<InlineRefCountBits> {
      strong RC + unowned RC + flags
      OR
      HeapObjectSideTableEntry*
    }
  }
}

HeapObjectSideTableEntry {
  SideTableRefCounts {
    object pointer
    atomic<SideTableRefCountBits> {
      strong RC + unowned RC + weak RC + flags
    }
  }
}
```

Konsekuensi praktis, dan ini jawaban yang bikin interviewer berhenti menggali:

1. Objek normal menyimpan refcount **inline** — nol alokasi tambahan.
2. Begitu **satu** `weak` dibuat, objek mendapat side table: satu alokasi heap tambahan,
   permanen seumur hidup objek.
3. `weak` reference menunjuk **side table**, bukan objek. Karena itu ia bisa jadi `nil`
   dengan aman: side table tetap hidup meski objeknya sudah bebas.
4. Membaca `weak` bukan operasi gratis — ia harus meng-*load* lewat side table dan
   melakukan operasi atomik untuk mempromosikan sementara jadi strong. `unowned` cuma
   dereference biasa.

> **Karena itu:** `weak` di dalam `cellForRowAt` yang dipanggil ribuan kali saat scroll
> bukan ide bagus. `weak` di property delegate — sempurna.

---

## 3. Retain cycle: mekanismenya, bukan definisinya

```swift
final class Person {
    let name: String
    var apartment: Apartment?
    init(name: String) { self.name = name }
    deinit { print("\(name) deinit") }
}

final class Apartment {
    let unit: String
    var tenant: Person?           // ⚠️ strong
    init(unit: String) { self.unit = unit }
    deinit { print("\(unit) deinit") }
}

var john: Person? = Person(name: "John")
var unit4A: Apartment? = Apartment(unit: "4A")

john?.apartment = unit4A
unit4A?.tenant = john

john = nil
unit4A = nil
// ❌ tidak ada deinit yang tercetak
```

Kenapa: setelah `john = nil`, strong RC `Person` masih 1 — karena `Apartment.tenant`
masih memegangnya. Begitu pula sebaliknya. Keduanya saling menahan, dan tidak ada
satu pun referensi dari luar yang bisa memutusnya. Memorinya bocor. Selamanya.

Perbaikannya: putuskan satu sisi.

```swift
final class Apartment {
    weak var tenant: Person?      // ✅
}
```

**Siapa yang jadi `weak`?** Aturannya bukan selera — ikuti arah kepemilikan:
sisi yang **tidak memiliki** yang jadi `weak`. Apartemen tidak memiliki penghuninya.
Parent memiliki child (strong); child menunjuk balik ke parent (`weak`).

---

## 4. `weak` vs `unowned` vs `unowned(unsafe)`

| | Bisa `nil`? | Biaya | Crash kalau objek sudah mati? | Pakai kalau |
|---|---|---|---|---|
| `weak` | Ya (jadi `Optional`) | Side table + atomic load | Tidak, jadi `nil` | Lifetime target **lebih pendek atau tidak pasti** |
| `unowned` | Tidak | Murah, cuma dereference | **Ya**, crash terdefinisi | Target dijamin hidup **selama** pemegangnya |
| `unowned(unsafe)` | Tidak | Nol (seperti pointer C) | Undefined behavior | Hampir tidak pernah. Hanya untuk hot path yang sudah diprofil |

Contoh `unowned` yang benar — parent dijamin hidup lebih lama:

```swift
final class Country {
    let name: String
    var capital: City!
    init(name: String, capitalName: String) {
        self.name = name
        self.capital = City(name: capitalName, country: self)
    }
}

final class City {
    let name: String
    unowned let country: Country     // ✅ City tidak mungkin hidup tanpa Country
    init(name: String, country: Country) { self.name = name; self.country = country }
}
```

Pilih `unowned` di sini bukan karena "lebih cepat", tapi karena `unowned` **mendokumentasikan
invariant**: kalau suatu saat `City` bisa hidup lebih lama dari `Country`, kamu ingin
crash-nya keras dan segera, bukan `nil` diam-diam yang bikin bug lain 3 layar kemudian.

---

## 5. Closure: sumber retain cycle nomor satu di app nyata

Closure adalah reference type. Ia menahan apa pun yang di-capture-nya.

```swift
final class SearchViewModel {
    var onStateChange: ((SearchState) -> Void)?

    func setup() {
        onStateChange = { state in
            self.handle(state)       // ⚠️ closure → self, dan self → closure
        }
    }
}
```

`self` menahan `onStateChange`; `onStateChange` menahan `self`. Siklus.

```swift
onStateChange = { [weak self] state in
    guard let self else { return }
    self.handle(state)
}
```

### `guard let self else { return }` — kenapa polanya begitu

Tanpa `guard`, setiap pemakaian `self?` di dalam closure adalah cek terpisah:
`self` bisa hidup di baris pertama dan mati di baris ketiga. `guard let self` mengambil
**satu** strong reference untuk seluruh durasi closure — konsisten, dan `self` dijamin
hidup sampai closure selesai. Bukan menghidupkan kembali siklus, karena strong-nya
temporer, bukan tersimpan.

Di project `movie`, pola ini dipakai dengan benar di dua tempat sekaligus:

```swift
// HomeViewModel.load()
Task { [weak self] in
    guard let self else { return }
    async let madeForYou = self.service.searchAlbums(term: "pop", limit: 10)
    ...
}

// HomeViewController.bindViewModel()
viewModel.onStateChange = { [weak self] state in
    guard let self else { return }
    switch state { ... }
}
```

Yang kedua itu **wajib**: `HomeViewController` memegang `viewModel` (strong),
dan `viewModel.onStateChange` memegang closure. Tanpa `[weak self]`,
view controller-nya tidak akan pernah di-`deinit` setelah di-pop dari navigation stack.

### Kapan `[weak self]` TIDAK diperlukan

Ini yang sering salah ke arah sebaliknya — orang menaruh `[weak self]` di mana-mana
sampai jadi noise.

```swift
// ❌ tidak perlu: closure non-escaping, selesai sebelum fungsi return
array.forEach { self.process($0) }
UIView.animate(withDuration: 0.3) { self.view.alpha = 0 }   // escaping, tapi lifetime pendek & terbatas

// ❌ tidak perlu: self tidak menahan closure-nya
URLSession.shared.dataTask(with: url) { data, _, _ in
    self.handle(data)     // tidak ada siklus — URLSession yang menahan closure, bukan self
}
// (tapi: `self` akan ditahan hidup sampai request selesai. Kadang itu justru yang kamu mau.)

// ✅ perlu: closure disimpan di property milik self, atau di objek yang dimiliki self
self.completionHandler = { [weak self] in ... }
NotificationCenter.default.addObserver(forName: ..., queue: nil) { [weak self] _ in ... }
Timer.scheduledTimer(withTimeInterval: 1, repeats: true) { [weak self] _ in ... }
```

**Uji cepatnya:** gambar panahnya. Apakah ada jalur `self → ... → closure → self`?
Kalau tidak ada jalur balik ke `self`, `[weak self]` cuma menambah `guard let` tanpa guna.

---

## 6. Sumber leak lain yang bukan closure

### `Timer` dengan target-action

```swift
timer = Timer.scheduledTimer(timeInterval: 1, target: self,
                             selector: #selector(tick), userInfo: nil, repeats: true)
```

`Timer` menahan `target` dengan **strong**, dan run loop menahan `Timer`.
`[weak self]` tidak berlaku di API target-action. Solusinya: `invalidate()` di
`viewDidDisappear`/`deinit`, atau pakai varian closure dengan `[weak self]`.

### `NotificationCenter.addObserver(forName:...)`

Varian **closure**-nya menahan closure kamu sampai kamu menghapus observernya.
Varian target-selector (yang lama) sejak iOS 9 tidak lagi butuh `removeObserver`
manual, tapi varian closure tetap butuh — simpan token-nya dan hapus.

### `DispatchSourceTimer`, `CADisplayLink`, KVO, delegate yang lupa `weak`

```swift
protocol TrackCellDelegate: AnyObject { ... }   // ✅ `: AnyObject` supaya bisa `weak`
final class TrackCell: UITableViewCell {
    weak var delegate: TrackCellDelegate?        // ✅ cell tidak memiliki delegate-nya
}
```

Ini pola dari `movie/UI/Search/TrackCell.swift`, dan `: AnyObject` di protokolnya bukan
kosmetik — tanpa itu, `weak var delegate` tidak akan compile, karena `weak` hanya berlaku
untuk reference type.

---

## 7. Membuktikan, bukan menebak

Urutan yang benar saat mencurigai leak:

1. **`deinit` + `print`.** Paling murah, 30 detik. Taruh di setiap view controller
   dan view model. Kalau tidak tercetak saat kamu pop layar → ada yang menahan.
2. **Xcode Memory Graph Debugger** (ikon tiga node di debug bar). Jalankan app, pop layar,
   tekan tombolnya. Objek yang seharusnya mati akan muncul; klik untuk melihat
   **siapa yang menahannya**. Purple runtime issue badge menandai siklus yang terdeteksi.
3. **Instruments → Leaks**. Untuk leak yang tidak terlihat di graph (mis. C buffer,
   `CoreFoundation`).
4. **Instruments → Allocations, "Persistent" + Mark Generation.** Cara terbaik untuk
   *abandoned memory*: memori yang tidak "leak" secara teknis (masih ada yang menunjuk)
   tapi tidak akan pernah dipakai lagi. Buka layar → tutup → Mark Generation → ulangi 5×.
   Kalau grafiknya naik terus, ada yang menumpuk.

Poin 4 penting karena Leaks **tidak** akan menangkap kasus paling umum di app iOS:
array yang terus bertambah, cache tanpa batas, atau view controller yang tersimpan
di dictionary dan tak pernah dihapus. Itu bukan siklus — itu kelalaian kepemilikan.

---

## 8. Cek pemahaman

**Q1.** Apa yang dicetak?
```swift
final class A { deinit { print("A") } }
var a1: A? = A()
unowned let a2 = a1!
a1 = nil
print("done")
```
<details><summary>Jawaban</summary>

`A` lalu `done`. `unowned` tidak menahan strong RC, jadi `a1 = nil` langsung memicu
`deinit`. Memorinya sendiri belum dibebaskan (unowned RC masih 1) — dan kalau kamu
menyentuh `a2` setelah ini, kamu dapat crash terdefinisi, bukan use-after-free.
</details>

**Q2.** Apakah ini retain cycle?
```swift
final class VM {
    var items: [Item] = []
    func load() {
        service.fetch { result in self.items = result }
    }
}
```
<details><summary>Jawaban</summary>

**Belum tentu.** Kalau `service` tidak dimiliki `self` dan closure-nya dilepas setelah
selesai (seperti `URLSession` completion), tidak ada siklus — `self` hanya ditahan hidup
sampai request selesai. Kalau `service` adalah property `self` **dan** ia menyimpan
closure itu di property-nya sendiri, barulah siklus: `self → service → closure → self`.

Jawaban terbaik di interview: "tergantung siapa yang menyimpan closure itu, dan berapa lama."
</details>

**Q3.** Kenapa `weak var delegate: SomeProtocol?` tidak compile kalau protokolnya
tidak `: AnyObject`?
<details><summary>Jawaban</summary>

Karena protokol tanpa constraint kelas bisa diadopsi `struct`/`enum`, dan value type
tidak punya refcount untuk di-`weak`-kan. `: AnyObject` menjamin adopternya reference type.
</details>

---

## Ringkasan

- ARC = `retain`/`release` yang disisipkan compiler; deterministik, tapi buta terhadap siklus.
- Objek punya **tiga** refcount (strong/unowned/weak). `deinit` (strong→0) dan pembebasan
  memori (unowned→0) adalah peristiwa terpisah.
- `weak` memaksa objek mendapat **side table** — satu alokasi permanen + baca yang lebih mahal.
  `unowned` cuma dereference.
- Pilih `weak` kalau lifetime target tidak pasti; `unowned` kalau target dijamin hidup
  lebih lama, karena ia mendokumentasikan invariant-nya.
- `[weak self]` diperlukan **hanya** kalau ada jalur `self → … → closure → self`.
  Menaburnya di mana-mana adalah noise.
- Leak paling umum di app bukan siklus, tapi *abandoned memory* — dan itu ditemukan
  dengan Allocations + Mark Generation, bukan Leaks.

## Selanjutnya

→ [04 — Protocol: Existential Container vs Generic Specialization](04-protocol-existential-vs-generics.md)
→ [Interview Prep: Memory Management](../../interview-prep/02-memory-management.md)

## Sumber

- `swift/stdlib/public/SwiftShims/swift/shims/RefCount.h` — tiga refcount, side table,
  object lifecycle state machine
- `swift/docs/RefcountingStates.graffle`
- [TSPL — Automatic Reference Counting](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/automaticreferencecounting/)
- Project `movie`: `HomeViewController.bindViewModel()`, `TrackCell.delegate`
