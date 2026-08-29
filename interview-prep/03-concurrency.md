---
title: "Interview Prep 03 — Concurrency"
category: "Interview Prep"
status: complete
tags:
  - swift-mastering
  - interview-prep
  - interview
---

# Interview Prep 03 — Concurrency

> 🟢 Junior · 🟡 Mid · 🔴 Senior — [cara pakai](README.md)

Kategori yang paling cepat berubah. Pastikan kamu bisa membicarakan **Swift 6
language mode**, bukan hanya `DispatchQueue`.

---

## 🟢 1. Apa bedanya serial queue dan concurrent queue?

<details><summary>Jawaban model</summary>

Serial: satu tugas selesai sebelum berikutnya mulai — urutan dijamin.
Concurrent: banyak tugas berjalan bersamaan — urutan mulai dijamin, urutan selesai tidak.

`DispatchQueue.main` adalah serial queue yang terikat main thread.
`DispatchQueue.global(qos:)` adalah concurrent.
</details>

**Follow-up:** *"Kenapa `DispatchQueue.main.sync` dari main thread deadlock?"*
→ `sync` menunggu block selesai, tapi main queue serial dan sedang menjalankan kode
yang memanggilnya. Ia menunggu dirinya sendiri.

---

## 🟢 2. `Task {}` vs `Task.detached {}`

<details><summary>Jawaban model</summary>

`Task {}` **mewarisi** actor isolation, prioritas, dan task-local values dari tempat
ia dibuat. `Task {}` di dalam method `@MainActor` mulai berjalan di main actor.

`Task.detached {}` tidak mewarisi apa pun — nonisolated, prioritas default.

Aturan praktis: hampir selalu `Task {}`. Kebutuhan `detached` biasanya tanda kamu
sedang melawan model isolasinya, dan solusi yang lebih baik biasanya `nonisolated func`
atau memindahkan pekerjaannya ke actor yang tepat.
</details>

---

## 🟡 3. Kenapa Apple membuat async/await padahal GCD sudah ada?

<details><summary>Jawaban model</summary>

Tiga masalah struktural yang tidak bisa diperbaiki library:

1. **Thread explosion.** GCD memetakan pekerjaan ke thread OS. Blok yang memblokir
   memaksa GCD membuat thread baru — sampai ~64 per QoS pool, masing-masing dengan
   stack ~1 MB. Swift Concurrency memakai cooperative pool sebesar jumlah core, dan
   `await` **men-suspend** task (melepas thread) alih-alih memblokirnya.
2. **Compiler tidak bisa memeriksa callback.** "Completion tidak pernah dipanggil"
   dan "dipanggil dua kali" adalah dua bug paling umum di kode callback, dan keduanya
   tidak terdeteksi. Fungsi `async` punya satu jalur keluar yang dijamin compiler.
3. **Tidak ada struktur.** `DispatchQueue.async` menghasilkan pekerjaan yatim: tidak
   ada induk, tidak ada pembatalan berjenjang. Structured concurrency memberi keduanya.
</details>

**Follow-up:** *"Apa yang masih lebih baik dilakukan dengan GCD?"*
→ `DispatchSource` (file/signal monitoring), kode yang benar-benar memblokir,
`concurrentPerform` untuk data-parallel CPU-bound, dan boundary dengan API Obj-C
berbasis queue.

📖 [Dari GCD ke Swift Concurrency](../articles/concurrency/01-gcd-ke-swift-concurrency.md)

---

## 🟡 4. Apa artinya `await`?

<details><summary>Jawaban model</summary>

`await` menandai **suspension point**: titik di mana fungsi ini **boleh** berhenti,
melepas thread-nya, dan dilanjutkan nanti — mungkin di thread lain.

Yang **tidak** dijamin `await`:
- Tidak dijamin lanjut di thread yang sama
- Tidak dijamin state-mu tidak berubah selama suspensi
- Tidak berarti ada penantian (kalau nilainya siap, bisa lanjut langsung)

Karena itu setiap `await` harus dibaca sebagai "titik di mana dunia bisa berubah".
Semua asumsi yang kamu buat sebelum `await` harus diperiksa ulang setelahnya.
</details>

---

## 🟡 5. Bagaimana kamu menjembatani API callback lama ke async?

<details><summary>Jawaban model</summary>

```swift
func loadImage(url: URL) async throws -> UIImage {
    try await withCheckedThrowingContinuation { continuation in
        legacyLoader.load(url) { image, error in
            if let image { continuation.resume(returning: image) }
            else { continuation.resume(throwing: error ?? LoaderError.unknown) }
        }
    }
}
```

Kontraknya keras: `resume` **tepat satu kali**.
- Nol kali → task tergantung selamanya (leak + spinner abadi). Versi `Checked`
  mencetak peringatan saat continuation di-`deinit`.
- Dua kali → **crash**: `SWIFT TASK CONTINUATION MISUSE`.

Kalau sumbernya menghasilkan **banyak** nilai (delegate lokasi, socket), continuation
adalah alat yang salah — pakai `AsyncStream`.
</details>

---

## 🟡 6. Apa yang dijamin `actor`, dan apa yang tidak?

<details><summary>Jawaban model</summary>

**Dijamin:** hanya satu task yang mengeksekusi kode terisolasi milik actor itu
pada satu waktu. Tidak ada data race pada state-nya.

**Tidak dijamin:**
- Urutan pemanggilan (task yang `await` duluan tidak dijamin dilayani duluan)
- Bahwa method berjalan sampai selesai tanpa disela
- Bahwa state-mu masih sama setelah `await`

Yang ketiga adalah **reentrancy**, dan itu sumber bug logika paling halus:

```swift
actor ImageCache {
    var cache: [URL: UIImage] = [:]
    func image(for url: URL) async throws -> UIImage {
        if let c = cache[url] { return c }
        let img = try await download(url)    // ← actor DILEPAS
        cache[url] = img                      // dua task bisa sampai sini untuk URL yang sama
        return img
    }
}
```

Tidak ada data race, tapi ada dua unduhan untuk satu URL.
Perbaikannya: simpan `Task`-nya di cache, bukan hasilnya, dan tulis **sebelum** `await`.
</details>

**Follow-up:** *"Bagaimana kamu memperbaiki cache itu?"* → `enum Entry { case inProgress(Task<UIImage, Error>); case ready(UIImage) }`,
tulis `.inProgress` sebelum `await` pertama, task kedua ikut `await t.value`.

📖 [Actor: Isolation & Reentrancy](../articles/concurrency/03-actor-isolation-reentrancy.md)

---

## 🟡 7. Apa itu `Sendable`?

<details><summary>Jawaban model</summary>

Marker protocol tanpa requirement — kontrak untuk type checker bahwa nilai bertipe ini
aman dioper melintasi batas concurrency domain.

Otomatis untuk: tipe value stdlib, `struct`/`enum` yang semua property-nya `Sendable`,
`actor`, dan tipe `@MainActor`.

Tidak otomatis untuk class dengan `var`, atau closure (yang butuh `@Sendable` eksplisit).

Tiga cara memperbaiki tipe yang gagal:
1. Immutability — `final class X: Sendable` dengan semua `let`
2. Isolasi — jadikan `actor` atau `@MainActor`
3. `@unchecked Sendable` + mekanisme penjamin (lock) **yang bisa kamu jelaskan dalam
   satu kalimat**. Kalau tidak bisa, kamu belum aman.
</details>

**Follow-up:** *"Apa bedanya `Sendable` dan `@Sendable`?"*
→ `Sendable` untuk tipe; `@Sendable` untuk closure/fungsi, memaksa semua capture-nya `Sendable`.

---

## 🔴 8. Apa itu region-based isolation?

<details><summary>Jawaban model</summary>

Aturan naif "semua yang menyeberang concurrency domain harus `Sendable`" terlalu ketat:

```swift
let client = HTTPClient()      // class biasa, tidak Sendable
await handler.use(client)      // aman! tidak ada referensi lain
```

Swift 6 (SE-0414) melacak "region" nilai yang mungkin saling terhubung, dan
mengizinkan pemindahan nilai non-`Sendable` selama regionnya **terputus** — yaitu
tidak ada jalur akses tersisa dari sisi pengirim. Kalau kamu memakai `client` lagi
setelah menyerahkannya, barulah error.

`sending` (SE-0430) membuat kontrak itu eksplisit di signature:
`func handOff(_ v: sending NonSendableThing)` = "aku mengambil alih, jangan sentuh lagi".

Ini penting untuk diketahui karena banyak tutorial lama mengajarkan kamu harus
membuat semuanya `Sendable`. Sering jawaban yang benar adalah mendesain kepemilikan
yang jelas, bukan menambal dengan `@unchecked`.
</details>

---

## 🔴 9. Bagaimana cancellation bekerja?

<details><summary>Jawaban model</summary>

**Kooperatif.** `task.cancel()` hanya menyalakan flag; ia tidak menghentikan kode apa pun.
Kode kamu yang harus memeriksanya.

```swift
if Task.isCancelled { return }        // cek diam-diam
try Task.checkCancellation()          // melempar CancellationError
```

Tempat memeriksa: **setelah setiap `await`**, dan di dalam loop panjang.

Yang paling sering dilupakan: memeriksa di blok `catch`. Saat task dibatalkan,
`URLSession` melempar `URLError.cancelled` — tanpa guard, layar akan menampilkan
pesan error dari request yang **sengaja** kamu batalkan.

```swift
} catch {
    guard !Task.isCancelled else { return }    // ← ini
    showError(...)
}
```

Pembatalan mengalir ke bawah: membatalkan induk membatalkan semua child task
(`async let`, `TaskGroup`). `Task {}` tidak terstruktur, jadi tidak ikut terbatalkan
otomatis — kamu harus menyimpannya dan `cancel()` sendiri.
</details>

---

## 🔴 10. Kamu punya 500 gambar untuk diunduh. Desain solusinya.

<details><summary>Jawaban model</summary>

Yang **salah**:

```swift
for url in urls { Task { await download(url) } }   // ❌
```

500 task tak terstruktur: tidak ada batas konkurensi (server menolak / memory spike),
tidak ada pembatalan kolektif, dan fungsi induknya selesai sebelum satu pun beres.

Yang benar — `TaskGroup` dengan **bounded concurrency**:

```swift
func downloadAll(_ urls: [URL], maxConcurrent: Int = 6) async throws -> [URL: Data] {
    try await withThrowingTaskGroup(of: (URL, Data).self) { group in
        var iterator = urls.makeIterator()
        var results: [URL: Data] = [:]

        // isi sampai batas
        for _ in 0..<maxConcurrent {
            guard let url = iterator.next() else { break }
            group.addTask { (url, try await download(url)) }
        }

        // setiap kali satu selesai, masukkan satu lagi
        while let (url, data) = try await group.next() {
            results[url] = data
            if let next = iterator.next() {
                group.addTask { (next, try await download(next)) }
            }
        }
        return results
    }
}
```

Poin yang harus kamu sebutkan:
- **6 concurrent** karena `URLSession` default `httpMaximumConnectionsPerHost` juga
  sekitar itu — melebihi itu hanya menambah antrian, bukan throughput.
- Pembatalan otomatis: kalau satu melempar, group membatalkan sisanya.
- Untuk kegagalan sebagian yang bisa ditoleransi, pakai `withTaskGroup` (non-throwing)
  dan kembalikan `Result` per item.
- Untuk retry, bungkus `download` dengan backoff eksponensial + jitter.
</details>

---

## 🔴 11. Apa yang berubah di Swift 6.2 untuk concurrency?

<details><summary>Jawaban model</summary>

Tiga hal besar:

1. **Default actor isolation.** Opsi build yang membuat seluruh module default
   `@MainActor`. Untuk app UI, ini sering menghapus mayoritas anotasi — kamu menandai
   yang **keluar** dari main actor, bukan yang masuk.
2. **`nonisolated(nonsending)`** — fungsi async nonisolated berjalan di *execution
   context pemanggil*, bukan langsung melompat ke global executor. Ini memperbaiki
   perilaku yang mengejutkan di mana memanggil fungsi async dari `@MainActor`
   diam-diam berpindah thread.
3. **`@concurrent`** — atribut untuk menyatakan secara eksplisit bahwa fungsi ini
   memang harus berjalan di executor konkuren, bukan mewarisi konteks pemanggil.

Swift 6.2 juga membawa migration tooling resmi untuk mengadopsi upcoming features
secara bertahap.

Untuk Swift 6.4 (WWDC26): `weak let` membuat class dengan weak reference immutable
bisa `Sendable` tanpa `@unchecked`; `~Sendable` untuk menolak conformance secara
eksplisit; `defer` bisa `await`; dan closure `Task` memberi peringatan untuk error
yang tidak ditangani.
</details>

---

## Latihan cepat

1. Kenapa `Thread.sleep` di dalam `Task` berbahaya tapi `Task.sleep` aman?
2. Apa yang terjadi kalau `resume` continuation dipanggil dua kali?
3. `async let a = f(); async let b = g(); let r = await (a, b)` — kapan `f()` mulai jalan?
4. Kenapa `actor` tidak menjamin urutan?
5. Sebutkan satu kasus di mana `@MainActor` lebih tepat daripada actor khusus.

<details><summary>Kunci</summary>

1. `Thread.sleep` memblokir thread cooperative pool (yang jumlahnya sebesar jumlah core);
   `Task.sleep` men-suspend task dan melepas thread-nya, dan bisa dibatalkan.
2. Crash: `SWIFT TASK CONTINUATION MISUSE`.
3. Saat baris `async let` dieksekusi, bukan saat `await`. Karena itu keduanya paralel.
4. Karena job dijadwalkan ke executor tanpa jaminan FIFO lintas suspensi; kalau urutan
   bagian dari kebenaran program, pakai `AsyncStream` atau antrian eksplisit.
5. State yang memang milik UI dan hanya disentuh dari main thread — actor terpisah
   hanya menambah executor hop dan permukaan reentrancy baru.
</details>
