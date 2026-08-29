---
title: "`Sendable` dan Data-Race Safety: Cara Compiler Membuktikan Kode Kamu Aman"
category: "Concurrency"
status: complete
tags:
  - swift-mastering
  - swift/concurrency
  - article
---

# `Sendable` dan Data-Race Safety: Cara Compiler Membuktikan Kode Kamu Aman

> **Kategori:** Concurrency · **Level:** Lanjut ·
> **Prasyarat:** [Actor: Isolation & Reentrancy](03-actor-isolation-reentrancy.md)
> **Baca ~20 menit**

---

## Klaim besar Swift 6

> Data race adalah **compile-time error**, bukan bug runtime.

Ini klaim yang berani. Data race secara historis adalah kelas bug paling sulit:
tidak deterministik, sering hanya muncul di perangkat lambat/cepat tertentu,
dan tidak bisa direproduksi di debugger.

Yang membuat klaim itu bisa ditegakkan adalah satu protokol dengan nol requirement.

---

## 1. `Sendable` adalah marker protocol

```swift
public protocol Sendable {}
```

Tidak ada method, tidak ada property. Ia tidak punya witness table dan tidak ada
di runtime — ia murni **kontrak untuk type checker**:

> "Nilai bertipe ini aman dioper melintasi batas concurrency domain."

Batas concurrency domain = masuk/keluar `Task`, masuk/keluar actor, dioper ke
`TaskGroup`, disimpan di property `@Sendable` closure.

---

## 2. Siapa yang otomatis `Sendable`

| Tipe | Sendable? |
|---|---|
| `Int`, `String`, `Bool`, `Double`, semua tipe value stdlib | ✅ otomatis |
| `struct`/`enum` yang **semua** stored property-nya `Sendable` | ✅ otomatis (kalau tidak `public`) |
| `struct` `public` di module dengan library evolution | ⚠️ harus deklarasi eksplisit |
| `actor` | ✅ otomatis |
| `final class` dengan semua property `let` bertipe `Sendable` | ✅ bisa dideklarasikan |
| `class` non-final, atau punya `var` | ❌ |
| Closure | ✅ hanya kalau ditandai `@Sendable` |
| Tipe `@MainActor` | ✅ (isolasinya yang menjamin) |

```swift
// movie/Model/Track.swift — Sendable otomatis
struct Track: Codable, Hashable, Identifiable {
    let id: Int
    let title: String
    let artistName: String?
    let albumName: String?
    let artworkPath: String?
    let durationMillis: Int?
}
```

Semua property `let` bertipe `Sendable` → `Track` otomatis `Sendable`.
Ini bukan kebetulan; ini konsekuensi langsung dari memilih `struct` dengan `let`
untuk model data. Keputusan desain di [artikel value semantics](../swift-language/01-value-vs-reference-semantics.md)
membayar dirinya di sini.

Sebaliknya:

```swift
final class LibraryStore {
    static let shared = LibraryStore()
    private(set) var savedTracks: [Track] = []      // ❌ var → tidak Sendable
}
```

Di Swift 6 language mode, `static let shared` pada tipe non-`Sendable` adalah error:
*"static property 'shared' is not concurrency-safe because non-'Sendable' type
may have shared mutable state"*.

---

## 3. Tiga cara memperbaiki tipe yang tidak `Sendable`

### (a) Jadikan immutable

```swift
final class Config: Sendable {
    let apiKey: String
    let baseURL: URL
    init(apiKey: String, baseURL: URL) { ... }
}
```

`final` + semua `let` + semua tipe property `Sendable` = compiler menerima
conformance-nya tanpa argumen. Ini opsi terbaik kalau memungkinkan.

### (b) Jadikan actor / `@MainActor`

```swift
@MainActor
final class LibraryStore {
    static let shared = LibraryStore()
    private(set) var savedTracks: [Track] = []
}
```

Isolasi menggantikan immutability sebagai bukti keamanan. Untuk store yang memang
hanya disentuh dari UI, ini pilihan yang paling jujur.

### (c) `@unchecked Sendable` — janji manual

```swift
final class Cache: @unchecked Sendable {
    private let lock = NSLock()
    private var storage: [String: Data] = [:]

    func value(for key: String) -> Data? {
        lock.lock(); defer { lock.unlock() }
        return storage[key]
    }
}
```

`@unchecked` berarti: **"aku menjamin ini aman dengan mekanisme yang tidak bisa
dilihat compiler."** Di sini mekanismenya `NSLock`.

Ini bukan escape hatch untuk membungkam error. Setiap `@unchecked Sendable`
adalah utang yang harus dibayar dengan komentar yang menjelaskan **apa** yang
menjamin keamanannya. Kalau kamu tidak bisa menuliskan alasannya dalam satu kalimat,
kamu belum aman.

> **Swift 6.4** menambahkan `~Sendable` untuk kasus sebaliknya: menandai tipe secara
> eksplisit **tidak boleh** dianggap `Sendable`, mencegah inferensi yang tidak diinginkan.
> Ia juga membuat `weak let` bisa dipakai, sehingga class dengan referensi weak
> yang immutable bisa jadi `Sendable` tanpa `@unchecked`.

---

## 4. `@Sendable` closure

```swift
func run(_ work: @Sendable () -> Void) { }
```

`@Sendable` pada closure berarti closure itu boleh menyeberang concurrency domain,
dan compiler memaksa **semua yang di-capture-nya** juga `Sendable`.

```swift
var counter = 0
Task {                              // closure Task selalu @Sendable
    counter += 1                    // ❌ error: mutation of captured var
}
```

Ini bukan pedantry — kode itu benar-benar adalah data race: `counter` ada di stack
frame pemanggil, dan task bisa menulisnya dari thread lain kapan saja.

Perbaikannya tergantung apa yang kamu maksud:

```swift
let snapshot = counter
Task { print(snapshot) }            // ✅ capture nilai, bukan variabel

let counter = Counter()             // actor
Task { await counter.increment() }  // ✅ mutasi lewat isolasi
```

---

## 5. Region-based isolation: kenapa Swift 6 lebih pintar dari yang kamu kira

Aturan naif "semua yang menyeberang harus `Sendable`" terlalu ketat. Contohnya:

```swift
func process() async {
    let client = HTTPClient()          // class biasa, TIDAK Sendable
    await handler.use(client)          // ❌ menurut aturan naif: error
}
```

Padahal ini aman: `client` baru dibuat, tidak ada referensi lain yang memegangnya.
Setelah dioper, pemanggil tidak menyentuhnya lagi.

Swift 6 memahami ini lewat **region-based isolation** (SE-0414). Compiler melacak
"region" nilai-nilai yang mungkin saling terhubung, dan mengizinkan pemindahan
nilai non-`Sendable` selama regionnya **terputus** — tidak ada jalur akses tersisa
dari sisi pengirim.

```swift
func process() async {
    let client = HTTPClient()
    await handler.use(client)          // ✅ region terputus — diperbolehkan
    // client.doSomething()            // ❌ kalau kamu memakainya lagi di sini, barulah error
}
```

Dan `sending` (SE-0430) membuat kontrak itu eksplisit di API:

```swift
func handOff(_ value: sending NonSendableThing) { }
// "aku mengambil alih nilai ini; pemanggil tidak boleh menyentuhnya lagi"
```

Ini penting untuk diketahui karena banyak tutorial lama mengajarkan bahwa kamu
harus membuat semuanya `Sendable`. Di Swift 6, sering jawabannya adalah
**mendesain alur kepemilikan yang jelas**, bukan menambal dengan `@unchecked`.

---

## 6. Strategi migrasi ke Swift 6 language mode

Menyalakan Swift 6 di codebase besar sekaligus akan menghasilkan ratusan error dan
membuat tim menyerah. Urutan yang berhasil:

**Langkah 1 — nyalakan warning dulu, per target.**

```
// Build Settings → Other Swift Flags, atau di Package.swift
swiftSettings: [ .enableUpcomingFeature("StrictConcurrency") ]
```
Di Swift 5 mode, ini memberi **warning**, bukan error. Codebase tetap bisa dibangun.

**Langkah 2 — mulai dari daun, bukan akar.**
Model layer dulu (`Track`, `Album` — biasanya sudah `Sendable` gratis), lalu service,
lalu view model, terakhir view layer.

**Langkah 3 — tandai UI dengan `@MainActor`.**
Sebagian besar error di layer UI hilang begitu tipe-tipenya diberi `@MainActor`,
karena isolasi menggantikan kebutuhan `Sendable`.

**Langkah 4 — untuk dependency pihak ketiga yang belum siap:**

```swift
@preconcurrency import SomeOldSDK
```
Ini menekan diagnostik dari module itu sampai mereka memperbaruinya.

**Langkah 5 — audit setiap `@unchecked Sendable` yang tersisa.**
Masing-masing butuh komentar yang menjelaskan mekanisme penjaminnya.

Swift 6.2 juga membawa **migration tooling** resmi yang bisa menerapkan sebagian
perubahan ini otomatis, dan opsi `-default-isolation MainActor` yang membuat
seluruh module default berjalan di main actor — untuk app UI, itu sering menghapus
mayoritas anotasi sekaligus.

---

## 7. Cek pemahaman

**Q1.** Kenapa ini error di Swift 6?
```swift
final class Logger {
    static let shared = Logger()
    var level: Level = .info
}
```
<details><summary>Jawaban</summary>

`static let shared` bisa diakses dari thread mana pun, tapi `Logger` punya `var level`
yang tidak dilindungi apa pun → shared mutable state tanpa isolasi.
Perbaikan: jadikan `level` immutable, jadikan `Logger` actor/`@MainActor`, atau lindungi
dengan lock + `@unchecked Sendable` beserta komentar penjelasnya.
</details>

**Q2.** `struct Wrapper { let cache: NSCache<NSString, UIImage> }` — Sendable?
<details><summary>Jawaban</summary>

Tidak otomatis, karena `NSCache` adalah class yang tidak dideklarasikan `Sendable`
(meski dokumentasinya menyebut ia thread-safe). Ini kasus sah untuk `@unchecked Sendable`
dengan komentar: *"NSCache is documented as thread-safe."* Perhatikan bahwa `Wrapper`
punya `let`, tapi `Sendable` menular dari **tipe** property, bukan dari mutabilitasnya.
</details>

**Q3.** Apa beda `Sendable` dan `@Sendable`?
<details><summary>Jawaban</summary>

`Sendable` = protokol untuk **tipe**. `@Sendable` = atribut untuk **closure/fungsi**,
yang mensyaratkan semua capture-nya `Sendable`. Keduanya menyatakan hal yang sama
untuk dua kategori berbeda.
</details>

---

## Ringkasan

- `Sendable` adalah marker protocol tanpa requirement; ia kontrak untuk type checker,
  bukan kode runtime.
- `struct`/`enum` dengan property `Sendable` mendapatkannya gratis — inilah imbalan
  memilih value type untuk model data.
- Tiga jalan memperbaiki: immutability (`final` + `let`), isolasi (`actor` / `@MainActor`),
  atau `@unchecked Sendable` dengan mekanisme penjamin yang bisa kamu tuliskan.
- `@Sendable` closure memaksa semua capture-nya `Sendable`.
- **Region-based isolation** membuat Swift 6 jauh lebih permisif dari aturan naif:
  nilai non-`Sendable` boleh berpindah selama regionnya terputus; `sending` membuatnya eksplisit.
- Migrasi: warning dulu, dari daun ke akar, `@MainActor` untuk UI, `@preconcurrency`
  untuk SDK pihak ketiga.

## Selanjutnya

→ [05 — Structured Concurrency & Cancellation](05-structured-concurrency-cancellation.md)
→ [06 — Swift 6.2: Default Actor Isolation](06-swift62-default-isolation.md)

## Sumber

- [SE-0302 — Sendable and @Sendable closures](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md)
- [SE-0414 — Region based isolation](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0414-region-based-isolation.md)
- [SE-0430 — `sending` parameter and result values](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0430-transferring-parameters-and-results.md)
- [Swift 6.2 released — Swift.org](https://www.swift.org/blog/swift-6.2-released/)
- `swift/userdocs/diagnostics/` — `sendable-closure-captures.md`, `sending-closure-risks-data-race.md`,
  `mutable-global-variable.md`, `non-sendable-superclass.md`
