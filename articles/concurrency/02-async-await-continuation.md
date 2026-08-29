---
title: "`async/await` Dibedah: Continuation, Suspension Point, dan Async Frame"
category: "Concurrency"
status: complete
tags:
  - swift-mastering
  - swift/concurrency
  - article
---

# `async/await` Dibedah: Continuation, Suspension Point, dan Async Frame

> **Kategori:** Concurrency · **Level:** Menengah–Lanjut ·
> **Prasyarat:** [Dari GCD ke Swift Concurrency](01-gcd-ke-swift-concurrency.md)
> **Baca ~20 menit**

---

## Yang sebenarnya dilakukan compiler

```swift
func loadProfile(id: String) async throws -> Profile {
    let user = try await fetchUser(id: id)
    let avatar = try await fetchAvatar(user.avatarURL)
    return Profile(user: user, avatar: avatar)
}
```

Compiler memotong fungsi ini di setiap `await` menjadi beberapa **partial function**,
dan membuat struktur di heap untuk menyimpan state di antara potongan-potongan itu:

```
loadProfile #1  : mulai, panggil fetchUser, SUSPEND
────── (thread dilepas; state disimpan di async frame) ──────
loadProfile #2  : resume dengan `user`, panggil fetchAvatar, SUSPEND
────── (thread dilepas lagi) ──────
loadProfile #3  : resume dengan `avatar`, bangun Profile, RETURN
```

Ini transformasi yang sama secara konsep dengan *continuation-passing style* —
hanya saja dilakukan compiler, bukan olehmu dengan callback bersarang.

---

## 1. Async frame: kenapa stack biasa tidak cukup

Fungsi sinkron memakai **stack frame**: dialokasi saat masuk, dibuang saat keluar,
dan hidup selama fungsi berjalan. Ini bekerja karena stack bersifat LIFO — fungsi
selalu keluar sebelum pemanggilnya.

Fungsi async melanggar itu. Ia bisa suspend, melepas thread, dan dilanjutkan
**di thread berbeda** beberapa detik kemudian. Stack frame-nya tidak boleh ikut hilang.

Karena itu Swift mengalokasikan **async frame** di heap (tepatnya di
*async task allocator*, alokator bump-pointer khusus per task yang jauh lebih murah
dari `malloc` umum). Isinya: variabel lokal yang masih hidup melewati suspension point,
plus pointer ke continuation berikutnya.

Konsekuensi praktis:

```swift
func f() async {
    let big = Array(repeating: 0, count: 1_000_000)   // hidup melewati await
    await something()
    print(big.count)                                   // → `big` disimpan di async frame
}

func g() async {
    let big = Array(repeating: 0, count: 1_000_000)
    print(big.count)                                   // selesai sebelum await
    await something()                                  // → `big` TIDAK disimpan
}
```

Variabel yang tidak dipakai setelah `await` tidak ikut ke async frame. Ini alasan nyata
untuk mempersempit scope variabel besar di fungsi async yang berumur panjang.

---

## 2. `await` bukan "tunggu di sini"

Ini kesalahpahaman paling mahal.

> `await` = **"fungsi ini BOLEH berhenti di sini, melepas thread, dan dilanjutkan nanti —
> mungkin di thread lain, mungkin setelah kode lain berjalan."**

Tiga hal yang tidak dijamin `await`:

1. **Tidak dijamin dilanjutkan di thread yang sama.**
2. **Tidak dijamin state-mu tidak berubah** selama suspensi (ini akar masalah
   [reentrancy actor](03-actor-isolation-reentrancy.md)).
3. **Tidak berarti ada penantian.** Kalau nilainya sudah tersedia, `await` bisa lanjut
   langsung tanpa suspensi sama sekali.

Karena itu setiap `await` harus dibaca sebagai **titik di mana dunia bisa berubah**.
Tandai secara mental setiap `await` di kode kamu; di situlah asumsi-asumsi kamu kedaluwarsa.

```swift
func update() async {
    guard items.count > 0 else { return }
    let fresh = await fetch()
    print(items[0])            // ⚠️ items bisa sudah kosong sekarang
}
```

---

## 3. Suspension point: di mana saja mereka?

Hanya di tempat yang ditandai `await` secara eksplisit — dan itu **fitur desain**,
bukan kebetulan. Kamu bisa membaca fungsi async dan langsung melihat semua titik
di mana ia bisa terputus. Bandingkan dengan preemptive threading di mana thread
bisa di-*preempt* di mana pun.

Bentuk-bentuk yang menghasilkan suspension point:

```swift
await someAsyncFunc()                       // panggilan fungsi async
try await someThrowingAsyncFunc()
for await item in stream { }                // AsyncSequence
async let a = f(); let x = await a          // async let, saat di-await
await withTaskGroup(of: Int.self) { ... }
await actor.someMethod()                    // hop ke executor actor
try await Task.sleep(for: .seconds(1))
await MainActor.run { ... }
```

---

## 4. Menjembatani dunia callback: `withCheckedContinuation`

Ini keterampilan yang benar-benar dipakai — hampir semua codebase punya API lama
berbasis delegate atau completion handler yang harus disambungkan ke `async`.

```swift
func loadImage(url: URL) async throws -> UIImage {
    try await withCheckedThrowingContinuation { continuation in
        legacyLoader.load(url) { image, error in
            if let image {
                continuation.resume(returning: image)
            } else {
                continuation.resume(throwing: error ?? LoaderError.unknown)
            }
        }
    }
}
```

**Kontraknya keras: `resume` harus dipanggil TEPAT SATU KALI.**

| Pelanggaran | Akibat |
|---|---|
| Tidak pernah `resume` | Task tergantung selamanya (memory leak + spinner abadi). Versi `checked` mencetak peringatan saat continuation di-`deinit` tanpa resume |
| `resume` dua kali | **Crash** — `Fatal error: SWIFT TASK CONTINUATION MISUSE` |

Varian `Unsafe...` (`withUnsafeContinuation`) menghapus pengecekan itu demi performa.
Aturan praktis: tulis dengan `Checked`, jalankan tes, dan hanya ganti ke `Unsafe`
kalau profiling membuktikannya perlu. Biaya `Checked` sangat kecil dibanding
biaya men-debug continuation yang bocor.

### Jebakan: delegate yang dipanggil berkali-kali

```swift
// ❌ SALAH — delegate ini memanggil didUpdate berkali-kali
func nextLocation() async -> CLLocation {
    await withCheckedContinuation { continuation in
        self.continuation = continuation
        locationManager.startUpdatingLocation()
    }
}
func locationManager(_ m: CLLocationManager, didUpdateLocations locs: [CLLocation]) {
    continuation?.resume(returning: locs.last!)   // panggilan kedua → CRASH
}
```

Perbaikannya: jadikan continuation `nil` segera setelah resume, atau — lebih tepat —
pakai `AsyncStream` karena sumbernya memang menghasilkan **banyak** nilai, bukan satu.

```swift
continuation?.resume(returning: loc)
continuation = nil                    // ✅ minimal
```

---

## 5. `Task`: jembatan dari sinkron ke async

Kamu tidak bisa `await` dari fungsi sinkron. `Task` adalah pintu masuknya.

```swift
// HomeViewModel.load() — dari project movie
func load() {                     // sinkron: dipanggil dari viewDidLoad
    onStateChange?(.loading)
    Task { [weak self] in         // ← pintu masuk ke dunia async
        guard let self else { return }
        async let madeForYou = self.service.searchAlbums(term: "pop", limit: 10)
        async let newAlbums  = self.service.searchAlbums(term: "2026", limit: 10)
        do {
            let (a, b) = try await (madeForYou, newAlbums)
            ...
        } catch { ... }
    }
}
```

Tiga hal yang perlu diperhatikan pada `Task {}`:

1. **Ia tidak terstruktur.** Ia tidak punya induk; `load()` selesai seketika sementara
   task-nya masih jalan. Tidak ada yang otomatis membatalkannya kalau view controller-nya
   ditutup — kamu harus menyimpan `Task` dan memanggil `cancel()` sendiri.
2. **Ia mewarisi konteks.** Prioritas, actor isolation, dan task-local values dari
   tempat ia dibuat. `Task {}` di dalam method `@MainActor` mulai di main actor.
3. **`[weak self]` biasanya tetap perlu**, karena task yang belum selesai menahan
   apa pun yang di-*capture*-nya, termasuk view model dan (lewat closure UI) view controller.

### `async let` di contoh itu

```swift
async let madeForYou = service.searchAlbums(term: "pop", limit: 10)
async let newAlbums  = service.searchAlbums(term: "2026", limit: 10)
let (a, b) = try await (madeForYou, newAlbums)
```

`async let` memulai child task **segera**, bukan saat di-`await`. Jadi kedua request
berjalan paralel, dan `await` di baris ketiga menunggu keduanya.

Kalau kamu menulisnya begini, paralelismenya hilang:

```swift
let a = try await service.searchAlbums(term: "pop", limit: 10)      // ❌ berurutan
let b = try await service.searchAlbums(term: "2026", limit: 10)     //    total = t1 + t2
```

Dan ini child task **terstruktur**: kalau salah satu melempar, yang lain otomatis
dibatalkan, dan tidak ada yang bocor keluar dari scope-nya.

---

## 6. Cancellation itu kooperatif

Membatalkan task **tidak menghentikan** kodenya. Ia hanya menyalakan sebuah flag.
Kode kamu yang harus memeriksanya.

```swift
// SearchViewModel.performSearch — dari project movie
searchTask = Task { [weak self] in
    guard let self else { return }
    do {
        let results = try await self.service.searchTracks(term: query, limit: 10)
        guard !Task.isCancelled else { return }        // ✅ cek setelah await
        self.rows = results.map { ... }
        self.onStateChange?(results.isEmpty ? .empty : .loaded)
    } catch {
        guard !Task.isCancelled else { return }        // ✅ jangan tampilkan error
        self.onStateChange?(.error("Couldn't load results. Check your connection."))
    }
}
```

Dua pengecekan itu **keduanya** penting, dan yang di `catch` sering dilupakan:
saat task dibatalkan, `URLSession` melempar `URLError.cancelled`. Tanpa guard itu,
setiap kali user mengetik huruf baru, layar akan berkedip menampilkan
"Couldn't load results" dari request yang sengaja kamu batalkan.

Dua cara memeriksa:

```swift
if Task.isCancelled { return }              // cek diam-diam, kamu yang menangani
try Task.checkCancellation()                // melempar CancellationError
```

Tempat memeriksa: **setelah setiap `await`**, dan di dalam loop panjang.

---

## 7. Cek pemahaman

**Q1.** Apa yang salah?
```swift
func fetchAll(urls: [URL]) async throws -> [Data] {
    var results: [Data] = []
    for url in urls {
        results.append(try await fetch(url))
    }
    return results
}
```
<details><summary>Jawaban</summary>

Secara benar tidak ada yang salah — tapi ini **serial**. 10 URL × 200 ms = 2 detik.
Untuk paralel, pakai `withThrowingTaskGroup`. Tidak ada `Task.isCancelled` juga, jadi
loop-nya akan terus mengunduh meski task sudah dibatalkan.
</details>

**Q2.** Kenapa `withCheckedContinuation` yang tidak pernah di-`resume` tidak crash,
tapi yang di-`resume` dua kali crash?
<details><summary>Jawaban</summary>

Resume dua kali berarti menulis hasil ke slot yang sudah terisi dan menjadwalkan
continuation yang sudah berjalan — itu korupsi state yang tidak bisa dipulihkan, jadi
`fatalError`. Tidak pernah resume "hanya" membuat task tergantung selamanya; versi
`Checked` mendeteksinya saat continuation di-`deinit` dan mencetak peringatan
alih-alih crash.
</details>

**Q3.** Fungsi ini `async` tapi tidak punya satu pun `await`. Legal?
<details><summary>Jawaban</summary>

Legal (compiler hanya memberi peringatan "no 'await' expressions occur within 'await' expression"
pada pemanggilnya kalau relevan). Tapi biasanya tanda bahwa `async`-nya tidak perlu —
kecuali kamu sengaja menyediakan titik suspensi untuk pemanggil, atau menyiapkan API
untuk kebutuhan async di masa depan.
</details>

---

## Ringkasan

- Compiler memotong fungsi async di setiap `await` menjadi partial function; state
  disimpan di **async frame** di heap, bukan stack.
- Variabel yang hidup melewati `await` ikut ke async frame — persempit scope nilai besar.
- `await` = titik di mana thread bisa dilepas **dan dunia bisa berubah**. Bukan "tunggu".
- `withCheckedContinuation` menjembatani callback; `resume` **tepat sekali** —
  nol kali = leak, dua kali = crash.
- `Task {}` = pintu dari sinkron ke async; tidak terstruktur, mewarisi konteks,
  harus disimpan kalau mau bisa dibatalkan.
- `async let` memulai pekerjaan **saat dideklarasikan**, bukan saat di-`await`.
- Cancellation kooperatif: cek `Task.isCancelled` setelah setiap `await` — **termasuk
  di blok `catch`**.

## Selanjutnya

→ [03 — Actor: Isolation, Executor, dan Jebakan Reentrancy](03-actor-isolation-reentrancy.md)

## Sumber

- `swift/stdlib/public/Concurrency/Continuation.swift`, `CheckedContinuation.swift`, `Task.swift`
- [SE-0296 — async/await](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0296-async-await.md)
- [WWDC21 — Swift concurrency: Behind the scenes](https://developer.apple.com/videos/play/wwdc2021/10254/)
- Project `movie`: `HomeViewModel.load`, `SearchViewModel.performSearch`
