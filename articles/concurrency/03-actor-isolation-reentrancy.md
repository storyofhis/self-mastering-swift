---
title: "Actor: Isolation, Executor, dan Jebakan Reentrancy"
category: "Concurrency"
status: complete
tags:
  - swift-mastering
  - swift/concurrency
  - article
---

# Actor: Isolation, Executor, dan Jebakan Reentrancy

> **Kategori:** Concurrency · **Level:** Lanjut ·
> **Prasyarat:** [async/await & continuation](02-async-await-continuation.md)
> **Baca ~22 menit**

---

## Apa yang sebenarnya dijamin `actor`

Satu kalimat, dan perhatikan apa yang **tidak** ada di dalamnya:

> Actor menjamin **hanya satu task yang mengeksekusi kode terisolasi milik actor itu
> pada satu waktu**.

Yang **tidak** dijamin:

- ❌ Tidak menjamin urutan pemanggilan.
- ❌ Tidak menjamin sebuah method berjalan sampai selesai tanpa disela.
- ❌ Tidak menjamin state kamu konsisten setelah `await`.

Tiga hal yang tidak dijamin itu adalah isi separuh artikel ini, dan sumber bug
konkurensi paling halus di app Swift modern.

---

## 1. Actor adalah reference type dengan executor

```swift
actor Counter {
    private var value = 0            // terisolasi otomatis
    func increment() { value += 1 }  // method terisolasi
    nonisolated let id = UUID()      // tidak terisolasi (immutable)
}
```

Dari `swift/stdlib/public/Concurrency/Actor.swift`:

```swift
public protocol Actor: AnyObject, Sendable {
  nonisolated var unownedExecutor: UnownedSerialExecutor { get }
}
```

Dan doc comment-nya menjelaskan tempat kerjanya:

> *"By default, actors execute tasks on a shared global concurrency thread pool.
> This pool is shared by all default actors and tasks, unless an actor or task
> specified a more specific executor requirement."*

Dua hal penting dari signature itu:

1. **`AnyObject`** — actor selalu reference type. Isolation butuh identitas;
   kalau actor bisa disalin, dua salinan punya state independen dan tidak ada
   yang bisa diserialkan. (Lihat
   [Value vs Reference Semantics §8](../swift-language/01-value-vs-reference-semantics.md).)
2. **`Sendable`** — actor otomatis aman dioper lintas concurrency domain, karena
   satu-satunya cara menyentuh state-nya adalah lewat executor-nya.

`unownedExecutor` inilah mekanismenya: setiap panggilan ke method terisolasi
dari luar actor dijadwalkan sebagai **job** di serial executor itu. Bukan lock —
tidak ada thread yang memblokir menunggu giliran. Task yang menunggu di-suspend,
thread-nya dipakai pekerjaan lain.

Bedanya dengan serial `DispatchQueue` sebagai lock:

| | Serial queue | Actor |
|---|---|---|
| Menunggu = | thread blocking (`sync`) atau callback (`async`) | suspensi task, thread bebas |
| Yang menjamin tidak ada akses langsung | konvensi + review | **compiler** |
| Deadlock kalau re-entrant | ya (`sync` ke queue sendiri) | tidak (tapi ada masalah lain — lihat §4) |

---

## 2. Isolation: aturan siapa boleh menyentuh apa

```swift
actor BankAccount {
    private var balance: Decimal = 0

    func deposit(_ amount: Decimal) { balance += amount }     // isolated
    func getBalance() -> Decimal { balance }                  // isolated

    nonisolated let accountNumber: String = "123"             // boleh diakses langsung
    nonisolated func format(_ d: Decimal) -> String {         // tidak boleh sentuh balance
        d.formatted(.currency(code: "IDR"))
    }
}

let account = BankAccount()

await account.deposit(100)          // ✅ dari luar: WAJIB await
let b = await account.getBalance()  // ✅
print(account.accountNumber)        // ✅ nonisolated, tanpa await
print(account.balance)              // ❌ error: actor-isolated property
```

Aturannya:

- **Dari dalam actor** ke anggotanya sendiri: sinkron, tanpa `await`.
- **Dari luar**: selalu `await` (kamu mungkin harus menunggu giliran executor).
- **`nonisolated`**: keluar dari isolasi — hanya boleh menyentuh state immutable
  atau `Sendable`. Compiler menegakkannya.

### Kenapa property `var` tidak bisa dibaca dari luar bahkan dengan `await`?

```swift
let b = await account.balance     // ✅ ini sebenarnya legal (getter implisit)
account.balance = 100             // ❌ ini tidak, dan tidak akan pernah
```

Menulis dari luar akan membuat pola `read-modify-write` mustahil diamankan:
`account.balance = await account.balance + 100` punya celah di antara baca dan tulis.
Swift menutup celah itu dengan melarang penulisan lintas actor sepenuhnya —
kamu harus menyediakan method terisolasi yang melakukan keduanya secara atomik.

---

## 3. Global actor & `@MainActor`

`@MainActor` adalah **global actor**: satu instance actor yang dipakai lintas seluruh
program, executor-nya adalah main thread.

```swift
// swift/stdlib/public/Concurrency/MainActor.swift
@globalActor public final actor MainActor: GlobalActor {
  public static let shared = MainActor()

  public nonisolated var unownedExecutor: UnownedSerialExecutor {
    return unsafe UnownedSerialExecutor(Builtin.buildMainActorExecutorRef())
  }

  public nonisolated func enqueue(_ job: UnownedJob) {
    _enqueueOnMain(job)
  }
}
```

`enqueue` yang meneruskan ke `swift_task_enqueueMainExecutor` — itulah yang
menggantikan `DispatchQueue.main.async`. Bedanya: sekarang **compiler tahu**
kode mana yang harus di main thread.

```swift
@MainActor
final class HomeViewController: UIViewController {
    func applySnapshot() { ... }        // dijamin di main thread oleh compiler
}
```

Ini peningkatan besar dibanding `DispatchQueue.main.async`: bug "UI update dari
background thread" berubah dari crash saat runtime jadi **error saat compile**.

Kamu bisa menandai di tiga level:

```swift
@MainActor class VM { }               // seluruh tipe
@MainActor func update() { }          // satu fungsi
@MainActor var items: [Item] = []     // satu property
```

Sejak Swift 6.2, ada opsi build untuk membuat **seluruh module** default `@MainActor`
(`-default-isolation MainActor`) — untuk app UI, ini sering justru default yang benar.
Lihat [artikel Swift 6.2](06-swift62-default-isolation.md).

---

## 4. Reentrancy — bagian yang membuat orang tersandung

**Actor Swift bersifat reentrant.** Saat sebuah method terisolasi mencapai `await`,
ia melepas actor-nya. Task lain boleh masuk dan menjalankan method actor yang sama.

```swift
actor ImageCache {
    private var cache: [URL: UIImage] = [:]

    func image(for url: URL) async throws -> UIImage {
        if let cached = cache[url] { return cached }

        let image = try await download(url)     // ⚠️ actor DILEPAS di sini
        cache[url] = image
        return image
    }
}
```

Kode ini **tidak punya data race** — compiler benar, `cache` selalu diakses secara
terserialkan. Tapi ia punya bug logika:

```
t=0   Task A: image(for: X) → cache kosong → mulai download → SUSPEND
t=1   Task B: image(for: X) → cache MASIH kosong → mulai download LAGI → SUSPEND
t=2   Task A: resume, cache[X] = img
t=3   Task B: resume, cache[X] = img (menimpa)
```

Dua unduhan untuk satu URL. Kalau ini bukan gambar tapi request pembuatan order,
kamu baru saja membuat dua order.

### Solusi: simpan task-nya, bukan hasilnya

```swift
actor ImageCache {
    private enum Entry {
        case inProgress(Task<UIImage, Error>)
        case ready(UIImage)
    }
    private var cache: [URL: Entry] = [:]

    func image(for url: URL) async throws -> UIImage {
        if let entry = cache[url] {
            switch entry {
            case .ready(let image):  return image
            case .inProgress(let t): return try await t.value   // ikut menunggu yang sudah jalan
            }
        }

        let task = Task { try await download(url) }
        cache[url] = .inProgress(task)          // ✅ ditulis SEBELUM await pertama

        do {
            let image = try await task.value
            cache[url] = .ready(image)
            return image
        } catch {
            cache[url] = nil                    // jangan cache kegagalan
            throw error
        }
    }
}
```

Kuncinya: **lakukan semua mutasi state yang menjaga invariant SEBELUM `await` pertama.**
Setelah `await`, anggap semua asumsimu kedaluwarsa.

### Pola pemeriksaan yang harus jadi refleks

```swift
func doWork() async {
    guard !isRunning else { return }
    isRunning = true                 // ✅ tandai sebelum await
    defer { isRunning = false }

    let result = await fetch()

    guard !isCancelled else { return }   // ✅ periksa ulang setelah await
    apply(result)
}
```

---

## 5. Actor bukan solusi untuk semua state bersama

Ini pertanyaan yang membedakan kandidat yang paham dari yang ikut tren.

**Jangan pakai actor kalau:**

| Situasi | Pakai apa |
|---|---|
| Data-nya immutable | `let` + `Sendable`. Tidak butuh isolasi sama sekali |
| State-nya hanya dipakai dari main thread (kebanyakan UI state) | `@MainActor` |
| Satu nilai atomik sederhana (counter, flag) | `Atomic<Int>` dari modul `Synchronization` (Swift 6+) |
| Objeknya berumur sangat pendek dan tidak dibagi | biarkan value type |
| Kamu butuh urutan FIFO yang dijamin | actor **tidak** menjamin urutan; pakai `AsyncStream` atau antrian eksplisit |

Baris terakhir itu penting dan sering luput: **actor tidak menjamin urutan**.
Task yang di-`await` lebih dulu tidak dijamin dilayani lebih dulu. Kalau urutan
adalah bagian dari kebenaran program kamu (mis. memproses event log), actor saja
tidak cukup.

Dan biaya actor bukan nol: setiap panggilan lintas actor adalah *executor hop* —
penjadwalan job, kemungkinan context switch. Untuk method yang dipanggil ribuan kali
per detik, itu terasa.

---

## 6. Menerapkannya ke project `movie`

`LibraryStore` di project itu adalah singleton yang menyimpan state mutable bersama —
kandidat paling jelas untuk actor:

```swift
// versi sekarang (implisit: aman karena hanya disentuh dari main thread)
final class LibraryStore {
    static let shared = LibraryStore()
    private(set) var savedTracks: [Track] = []
    func toggleSave(_ track: Track) -> Bool { ... }
    func isSaved(_ track: Track) -> Bool { ... }
}
```

Ada dua migrasi yang benar, dan memilih di antaranya adalah keputusan desain nyata:

**Opsi A — `@MainActor`.** Store ini hanya disentuh dari view model yang meng-update UI.
`@MainActor final class LibraryStore` membuat compiler menegakkan apa yang sudah
menjadi kenyataan, dan `isSaved` tetap sinkron dari kode `@MainActor` lain.
Ini pilihan yang tepat untuk app ini.

**Opsi B — `actor LibraryStore`.** Benar juga, tapi setiap `isSaved` jadi `await`:

```swift
self.rows = results.map { TrackRowViewModel(track: $0, isSaved: self.library.isSaved($0)) }
// ❌ tidak bisa: `map` sinkron, `isSaved` sekarang async
```

Kamu harus mengubahnya jadi loop async atau mengambil seluruh set sekali:

```swift
let savedIDs = await library.savedIDs()          // satu hop, bukan N hop
self.rows = results.map { TrackRowViewModel(track: $0, isSaved: savedIDs.contains($0.id)) }
```

Perhatikan bahwa versi kedua itu **lebih baik apa pun pilihanmu** — ia mengganti N
pemanggilan dengan satu. Itu pola umum saat memigrasikan kode ke actor: batasi
jumlah lintasan boundary, jangan memindahkan boundary ke dalam loop.

---

## 7. Cek pemahaman

**Q1.** Apakah ini punya data race?
```swift
actor Counter {
    var value = 0
    func increment() async {
        let current = value
        await Task.yield()
        value = current + 1
    }
}
// 100 task memanggil increment() bersamaan
```
<details><summary>Jawaban</summary>

**Tidak ada data race** — compiler benar, akses ke `value` selalu terserialkan.
Tapi hasil akhirnya **bukan 100**. `await Task.yield()` melepas actor; banyak task
membaca `current` yang sama lalu menulis nilai yang sama. Ini *lost update*,
bug logika klasik akibat reentrancy. Perbaikannya: hilangkan `await` di antara
baca dan tulis (`value += 1` saja).
</details>

**Q2.** Kenapa `nonisolated func` tidak boleh menyentuh property `var` actor?
<details><summary>Jawaban</summary>

Karena `nonisolated` berarti ia bisa dipanggil dari concurrency domain mana pun tanpa
melewati executor actor — jadi tidak ada yang menyerialkan aksesnya. Ia hanya boleh
menyentuh `let` immutable atau nilai `Sendable`.
</details>

**Q3.** Kapan `@MainActor` lebih tepat daripada `actor` khusus?
<details><summary>Jawaban</summary>

Kapan pun state-nya memang milik UI dan hanya dibaca/ditulis dari main thread.
Membuat actor terpisah untuk itu justru menambah executor hop dan memaksa `await`
di tempat yang sebelumnya sinkron — biaya tanpa manfaat, plus permukaan reentrancy baru.
</details>

---

## Ringkasan

- Actor menjamin **eksekusi terserialkan**, bukan urutan, bukan atomisitas melintasi `await`.
- `protocol Actor: AnyObject, Sendable` dengan `unownedExecutor` — actor selalu class-like
  dan berjalan di cooperative pool kecuali diberi executor khusus.
- `@MainActor` adalah global actor yang executor-nya main thread; ia mengubah bug
  "UI dari background thread" jadi error compile.
- **Reentrancy**: setiap `await` di dalam method actor melepas actor. Lakukan mutasi
  penjaga-invariant sebelum `await` pertama, dan periksa ulang asumsi setelahnya.
- Cache actor yang naif menghasilkan pekerjaan duplikat — simpan `Task`, bukan hasil.
- Actor bukan default untuk semua state bersama: pertimbangkan `let`, `@MainActor`,
  atau `Atomic` lebih dulu. Setiap lintasan actor adalah executor hop.

## Selanjutnya

→ [04 — `Sendable` dan Data-Race Safety](04-sendable-data-race-safety.md)

## Sumber

- `swift/stdlib/public/Concurrency/Actor.swift` — `protocol Actor`, `unownedExecutor`
- `swift/stdlib/public/Concurrency/MainActor.swift` — `@globalActor MainActor`, `enqueue`
- [SE-0306 — Actors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md)
- [SE-0316 — Global actors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0316-global-actors.md)
- [WWDC21 — Protect mutable state with Swift actors](https://developer.apple.com/videos/play/wwdc2021/10133/)
