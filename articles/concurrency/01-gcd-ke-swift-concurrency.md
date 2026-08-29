---
title: "Dari GCD ke Swift Concurrency: Thread Explosion dan Cooperative Pool"
category: "Concurrency"
status: complete
tags:
  - swift-mastering
  - swift/concurrency
  - article
---

# Dari GCD ke Swift Concurrency: Thread Explosion dan Cooperative Pool

> **Kategori:** Concurrency · **Level:** Menengah ·
> **Prasyarat:** [ARC](../swift-language/03-arc-dan-retain-cycle.md), pernah pakai `DispatchQueue`
> **Baca ~18 menit**

---

## Pertanyaan yang selalu keluar

> "Kenapa Apple membuat async/await padahal GCD sudah ada dan bekerja?"

Jawaban "supaya kodenya lebih rapi" tidak cukup — itu jawaban kosmetik untuk perubahan
yang alasannya **struktural**. Ada tiga masalah yang tidak bisa diperbaiki GCD tanpa
mengubah bahasanya.

---

## 1. Masalah pertama: thread explosion

GCD memetakan pekerjaan ke thread OS. Kalau sebuah thread **memblokir** (menunggu I/O,
menunggu lock, menunggu semaphore), GCD tidak bisa memakainya untuk pekerjaan lain —
jadi ia membuat thread baru.

```swift
for i in 0..<100 {
    DispatchQueue.global().async {
        let data = try? Data(contentsOf: url)   // BLOCKING
        process(data)
    }
}
```

100 blok yang memblokir → GCD berusaha membuat sampai ~64 thread (batas per QoS pool).
Setiap thread punya stack 512 KB–1 MB. Puluhan MB memori hilang untuk thread yang
**tidak melakukan apa-apa selain menunggu**, plus biaya context switch yang menggerogoti CPU.

Di perangkat dengan 6 core, menjalankan 64 thread berarti kernel menghabiskan waktu
memindah-mindahkan thread alih-alih mengerjakan pekerjaanmu.

### Bagaimana Swift Concurrency menghindarinya

Swift Concurrency memakai **cooperative thread pool** dengan jumlah thread
kira-kira **sebanyak core CPU**, dan tidak pernah lebih.

Kuncinya: `await` **tidak memblokir thread**. Saat sebuah task menunggu, thread-nya
dilepas dan dipakai task lain. Task yang menunggu disimpan sebagai struktur data di
heap, bukan sebagai thread yang tidur.

```swift
for i in 0..<100 {
    Task {
        let data = try? await URLSession.shared.data(from: url)   // SUSPENDING
        process(data)
    }
}
```

100 task, tetap ~6 thread. Itu perbedaan yang tidak bisa dicapai library — butuh
dukungan compiler untuk memotong fungsi jadi potongan-potongan yang bisa dilanjutkan.

> **Konsekuensi yang sering diuji:** kamu **tidak boleh** memblokir di dalam kode async.
> `Thread.sleep`, `DispatchSemaphore.wait()`, `sync` ke queue lain, atau `Data(contentsOf:)`
> di dalam `Task` merusak asumsi pool ini dan bisa membuat *deadlock* — karena tidak
> ada thread cadangan untuk menyelesaikan pekerjaan yang kamu tunggu.

---

## 2. Masalah kedua: nesting dan error handling

```swift
// GCD
func loadProfile(id: String, completion: @escaping (Result<Profile, Error>) -> Void) {
    fetchUser(id: id) { userResult in
        switch userResult {
        case .failure(let e): completion(.failure(e))            // ← lupa satu ini
        case .success(let user):                                  //   = callback hilang
            self.fetchAvatar(user.avatarURL) { avatarResult in
                switch avatarResult {
                case .failure(let e): completion(.failure(e))
                case .success(let avatar):
                    self.fetchPosts(user.id) { postsResult in
                        // ... tiga level, dan kita baru di tengah
                    }
                }
            }
        }
    }
}
```

Masalahnya bukan estetika. Compiler **tidak bisa memeriksa** apakah `completion`
dipanggil tepat sekali di setiap jalur. Bug "completion tidak pernah dipanggil"
(spinner berputar selamanya) dan "completion dipanggil dua kali" adalah dua bug
paling umum di codebase berbasis callback, dan keduanya tidak terdeteksi compiler.

```swift
// Swift Concurrency
func loadProfile(id: String) async throws -> Profile {
    let user = try await fetchUser(id: id)
    let avatar = try await fetchAvatar(user.avatarURL)
    let posts = try await fetchPosts(user.id)
    return Profile(user: user, avatar: avatar, posts: posts)
}
```

Satu jalur keluar, error dipropagasi otomatis, compiler menjamin fungsi ini
mengembalikan nilai atau melempar — tidak ada jalur "diam".

---

## 3. Masalah ketiga: tidak ada yang menjaga struktur

Di GCD, setiap `async` adalah pekerjaan yatim. Tidak ada yang tahu siapa induknya,
tidak ada cara membatalkannya secara berjenjang, dan kalau kamu meninggalkan layar
di tengah lima request, kelimanya tetap jalan sampai selesai lalu memanggil
callback yang hasilnya dibuang.

Swift Concurrency memperkenalkan **structured concurrency**: task punya induk,
pembatalan mengalir ke bawah, dan induk tidak selesai sebelum anaknya selesai.

```swift
// HomeViewModel.load() — dari project movie
async let madeForYou = self.service.searchAlbums(term: "pop", limit: 10)
async let newAlbums  = self.service.searchAlbums(term: "2026", limit: 10)
let (a, b) = try await (madeForYou, newAlbums)
```

Dua request jalan **paralel**, hasilnya ditunggu bersama, dan kalau salah satu melempar,
yang lain otomatis dibatalkan. Versi GCD-nya butuh `DispatchGroup`, variabel hasil
yang di-*capture*, dan penanganan error manual — sekitar 25 baris untuk perilaku
yang sama, dengan tiga cara untuk salah.

---

## 4. Peta padanan GCD → Swift Concurrency

| GCD | Swift Concurrency | Catatan |
|---|---|---|
| `DispatchQueue.global().async { }` | `Task { }` / `Task.detached { }` | `Task {}` mewarisi konteks & prioritas; `detached` tidak |
| `DispatchQueue.main.async { }` | `await MainActor.run { }` atau anotasi `@MainActor` | Anotasi lebih baik: dicek compiler |
| `DispatchGroup` | `async let` / `withTaskGroup` | Dengan pembatalan otomatis |
| Serial queue sebagai lock | `actor` | Isolation dicek compiler, bukan konvensi |
| `DispatchSemaphore` (bounded concurrency) | `withTaskGroup` + batasi jumlah task aktif | **Jangan** pakai semaphore di kode async |
| `DispatchWorkItem` + `cancel()` | `Task` + `Task.isCancelled` / `checkCancellation()` | |
| `DispatchQueue.asyncAfter` | `try await Task.sleep(for: .seconds(1))` | Versi async bisa dibatalkan |
| `qos:` | `Task(priority:)` | |

### Contoh migrasi nyata: debounce

Project `movie` masih memakai bentuk GCD untuk debounce, dan itu keputusan yang
masuk akal untuk kodenya saat itu:

```swift
// SearchViewModel — versi GCD
private var pendingSearch: DispatchWorkItem?
private var searchTask: Task<Void, Never>?

func searchTextDidChange(_ text: String) {
    pendingSearch?.cancel()
    let work = DispatchWorkItem { [weak self] in self?.performSearch(query: query) }
    pendingSearch = work
    DispatchQueue.main.asyncAfter(deadline: .now() + 0.4, execute: work)
}
```

Versi Swift Concurrency murni menghapus satu dari dua mekanisme pembatalan:

```swift
private var searchTask: Task<Void, Never>?

func searchTextDidChange(_ text: String) {
    searchTask?.cancel()                      // satu mekanisme, bukan dua
    let query = text.trimmingCharacters(in: .whitespacesAndNewlines)
    guard !query.isEmpty else { reset(); return }

    searchTask = Task { [weak self] in
        guard let self else { return }
        try? await Task.sleep(for: .milliseconds(400))   // debounce
        guard !Task.isCancelled else { return }          // ← sleep-nya ikut dibatalkan
        await self.performSearch(query: query)
    }
}
```

Keuntungannya konkret: `Task.sleep` **bisa dibatalkan**, jadi `searchTask?.cancel()`
membatalkan penundaan *dan* request yang mungkin sudah jalan. Di versi GCD, kamu perlu
membatalkan `pendingSearch` **dan** `searchTask` secara terpisah — dua state yang harus
selalu konsisten, dan itu permukaan bug.

---

## 5. Yang GCD masih lebih baik

Jangan menjawab "GCD sudah mati" di interview. Itu salah, dan interviewer yang
berpengalaman akan menggali.

| Kasus | Kenapa GCD/lain masih dipakai |
|---|---|
| Deployment target < iOS 13 | Swift Concurrency butuh iOS 13+ (dan back-deploy runtime) |
| `DispatchSource` untuk file/signal monitoring | Tidak ada padanan langsung |
| Kode yang benar-benar memblokir (sinkron, C library) | Harus dijalankan di thread sendiri, bukan cooperative pool |
| `DispatchQueue.concurrentPerform` untuk data-parallel loop CPU-bound | Masih bentuk paling langsung |
| Interop dengan API Objective-C berbasis queue | Sering lebih mudah tetap di GCD di boundary-nya |

Dan yang paling penting: **mencampur keduanya adalah sumber deadlock**.
Kalau kamu memanggil `DispatchSemaphore.wait()` dari dalam fungsi `async`, kamu bisa
mengunci thread cooperative pool yang dibutuhkan untuk menyelesaikan hal yang kamu tunggu.
Di pool berukuran jumlah-core, kamu hanya butuh beberapa kejadian untuk membekukan app.

---

## 6. Cek pemahaman

**Q1.** Kenapa `Thread.sleep(forTimeInterval: 1)` di dalam `Task` berbahaya, sementara
`try await Task.sleep(for: .seconds(1))` aman?
<details><summary>Jawaban</summary>

`Thread.sleep` **memblokir** thread cooperative pool — thread itu tidak bisa mengerjakan
task lain selama 1 detik. Dengan pool sebesar jumlah core, beberapa kejadian bersamaan
bisa melumpuhkan seluruh concurrency app. `Task.sleep` **men-suspend** task: thread-nya
dilepas dan dipakai task lain, dan penundaannya bisa dibatalkan.
</details>

**Q2.** Apa bedanya `Task { }` dan `Task.detached { }`?
<details><summary>Jawaban</summary>

`Task {}` mewarisi konteks aktor, prioritas, dan task-local values dari tempat ia dibuat —
jadi `Task {}` di dalam method `@MainActor` mulai berjalan di main actor.
`Task.detached {}` tidak mewarisi apa pun; ia mulai nonisolated dengan prioritas default.
Aturan praktis: hampir selalu pakai `Task {}`. `detached` biasanya tanda kamu sedang
melawan model isolasinya.
</details>

**Q3.** Kamu punya 200 URL untuk diunduh. Kenapa `for url in urls { Task { await download(url) } }`
bukan jawaban yang baik?
<details><summary>Jawaban</summary>

Itu membuat 200 task tak terstruktur sekaligus: tidak ada batas konkurensi (server bisa
menolak / memory spike), tidak ada cara membatalkan semuanya, dan fungsi induknya selesai
sebelum satu pun unduhan beres. Yang benar: `withTaskGroup` dengan jumlah task aktif
dibatasi — lihat [artikel structured concurrency](05-structured-concurrency-cancellation.md).
</details>

---

## Ringkasan

- GCD memetakan pekerjaan ke thread OS; blocking → thread explosion (≈64 thread × ~1 MB stack).
- Swift Concurrency memakai cooperative pool sebesar jumlah core; `await` **men-suspend**,
  tidak memblokir.
- Callback tidak bisa diperiksa compiler; `async` bisa — satu jalur keluar, error terpropagasi.
- Structured concurrency memberi induk, pembatalan berjenjang, dan jaminan lifetime yang
  tidak dimiliki `DispatchQueue.async`.
- Jangan memblokir di dalam kode async: `Thread.sleep`, `DispatchSemaphore.wait`, `queue.sync`.
- GCD belum mati — ia masih tempatnya untuk `DispatchSource`, kode blocking, dan boundary Obj-C.

## Selanjutnya

→ [02 — `async/await` Dibedah: Continuation & Suspension Point](02-async-await-continuation.md)

## Sumber

- [WWDC21 — Swift concurrency: Behind the scenes](https://developer.apple.com/videos/play/wwdc2021/10254/)
- `swift/stdlib/public/Concurrency/` — `GlobalExecutor.cpp`, `CooperativeGlobalExecutor.cpp`
- Project `movie`: `SearchViewModel.searchTextDidChange`, `HomeViewModel.load`
