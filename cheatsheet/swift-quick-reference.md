---
title: "Swift Quick Reference"
category: "Cheat Sheet"
status: complete
tags:
  - swift-mastering
  - cheatsheet
---

# Swift Quick Reference

Contekan 30 menit sebelum interview. Semuanya angka dan aturan yang harus keluar
tanpa berpikir.

---

## Memory layout

| Ekspresi | Nilai | Kenapa |
|---|---|---|
| `MemoryLayout<Int>.size` | 8 | |
| `MemoryLayout<Int?>.size` / `.stride` | 9 / **16** | `Int` tanpa extra inhabitant → tag byte + padding |
| `MemoryLayout<Bool>.size` | 1 | |
| `MemoryLayout<Bool?>.size` | **1** | 254 extra inhabitants tersedia |
| `MemoryLayout<AnyObject>.size` / `AnyObject?` | 8 / **8** | pointer nol = `nil` |
| `MemoryLayout<[Int]>.size` | 8 | Array = satu pointer ke buffer |
| `MemoryLayout<String>.size` | 16 | |
| `MemoryLayout<any P>.size` (P tanpa constraint) | **40** | buffer 3 word + metadata + 1 witness table |
| `MemoryLayout<any AnyObject & P>.size` | 16 | pointer + witness table |
| Header objek class | 16 | metadata pointer + refcount word |

**`size` vs `stride`:** `size` = byte terpakai; `stride` = jarak antar elemen di array
(size dibulatkan ke kelipatan `alignment`). **Array memakai `stride`.**

**Batas existential inline:** 3 word = **24 byte**. Lebih dari itu → alokasi heap.

---

## ARC

```
strong RC → 0   ⇒ deinit dipanggil
unowned RC → 0  ⇒ memori dibebaskan
weak RC → 0     ⇒ side table dibebaskan
```

- `weak` → objek dapat **side table** (alokasi permanen, satu arah), baca lebih mahal
- `unowned` → dereference biasa, crash terdefinisi kalau target sudah mati
- `unowned(unsafe)` → nol biaya, undefined behavior

**`[weak self]` diperlukan kalau ada jalur `self → … → closure → self`:**

| Perlu | Tidak perlu |
|---|---|
| `self.closureProperty = { ... }` | `array.forEach { self.f($0) }` (non-escaping) |
| `Timer.scheduledTimer(...) { }` | `URLSession.dataTask { }` (session yang menahan) |
| `NotificationCenter.addObserver(forName:) { }` | `UIView.animate { }` |
| `dataSource.supplementaryViewProvider = { }` | |

**Copy-on-write:**
```swift
private mutating func ensureUnique() {
    if !isKnownUniquelyReferenced(&storage) { storage = storage.copy() }
}
```
`Storage` **harus** `final class`. Jangan simpan `storage` ke variabel lokal sebelum cek.

---

## Concurrency

| GCD | Swift Concurrency |
|---|---|
| `DispatchQueue.global().async` | `Task { }` |
| `DispatchQueue.main.async` | `@MainActor` / `await MainActor.run { }` |
| `DispatchGroup` | `async let` / `withTaskGroup` |
| Serial queue sebagai lock | `actor` |
| `DispatchSemaphore` | `withTaskGroup` + batas jumlah task |
| `asyncAfter` | `try await Task.sleep(for:)` |

**Jangan pernah di dalam kode async:** `Thread.sleep`, `DispatchSemaphore.wait()`,
`queue.sync`, `Data(contentsOf:)`.

**Actor menjamin** eksekusi terserialkan.
**Actor TIDAK menjamin** urutan, atomisitas melintasi `await`, atau state tetap sama
setelah `await`.

**Aturan reentrancy:** lakukan semua mutasi penjaga-invariant **sebelum** `await` pertama;
periksa ulang asumsi **setelah** setiap `await`.

**Cancellation kooperatif:**
```swift
guard !Task.isCancelled else { return }   // setelah setiap await — TERMASUK di catch
try Task.checkCancellation()
```

**Continuation:** `resume` tepat satu kali. Nol = leak. Dua = crash.

**Sendable otomatis:** value type stdlib, `struct`/`enum` dengan property `Sendable`,
`actor`, tipe `@MainActor`.
**Tidak otomatis:** class dengan `var`, closure (butuh `@Sendable`).

---

## Protocol & generics

| | `any P` | `some P` / `<T: P>` |
|---|---|---|
| Tipe ditentukan | runtime | compile time |
| Representasi | existential container (≥40 B) | tipe konkret |
| Dispatch | witness table (indirect) | bisa dispesialisasi → langsung |
| Koleksi heterogen | ✅ | ❌ |
| Type identity antar parameter | ❌ | ✅ (`<T>`) |

**Spesialisasi terjadi kalau:** file sama, atau satu module + WMO, atau lintas module
dengan `@inlinable`. Di Debug (`-Onone`): tidak pernah.

---

## Optional & enum

```swift
enum Optional<Wrapped> { case none; case some(Wrapped) }
```

| Strategi layout | Kapan |
|---|---|
| Integer kecil | Enum tanpa payload |
| Extra inhabitant payload | Enum dengan tepat satu case ber-payload (termasuk `Optional`) |
| Tag terpisah / spare bits | Enum dengan banyak case ber-payload |

Ukuran enum = payload **terbesar** + tag, bukan jumlah semua payload.

---

## UIKit

**Lifecycle:** `loadView` → `viewDidLoad` → `viewWillAppear` → `viewWillLayoutSubviews`
→ `viewDidLayoutSubviews` → `viewDidAppear` → `viewWillDisappear` → `viewDidDisappear` → `deinit`

- Setup sekali → `viewDidLoad`
- Refresh data → `viewWillAppear`
- Apa pun yang bergantung `bounds` → `viewDidLayoutSubviews`

**`frame` vs `bounds`:** frame = koordinat superview; bounds = koordinat sendiri.
Dengan transform, frame = bounding box, bounds tidak berubah.
`scrollView.contentOffset == scrollView.bounds.origin`.

**Layout:**
- `setNeedsLayout()` — asinkron, layout pass berikutnya
- `layoutIfNeeded()` — sinkron, sekarang (wajib di dalam blok animasi constraint)
- `setNeedsDisplay()` — gambar ulang (`draw(_:)`), bukan layout

**Hugging** = menolak membesar. **Compression resistance** = menolak mengecil.

**Diffable:**
- Item & section identifier harus `Hashable` dan **unik** (duplikat = crash)
- `reconfigureItems` (iOS 15+) > `reloadItems` untuk memperbarui isi
- `apply` harus konsisten thread-nya
- Compositional: Item → Group → Section → Layout;
  `section.orthogonalScrollingBehavior = .continuous` untuk shelf horizontal
- `.estimated` untuk apa pun yang tingginya bergantung teks (Dynamic Type)

**`prepareForReuse` harus membersihkan:** request gambar (cancel), image/text,
delegate, animasi, observer, state seleksi custom.

**`URLSession` tidak melempar untuk 4xx/5xx** — periksa `statusCode` sendiri.

---

## Angka yang layak dihafal

| | |
|---|---|
| Header objek class | 16 byte |
| Batas existential inline | 24 byte (3 word) |
| Ukuran `any P` minimum | 40 byte |
| Thread stack (GCD) | ~512 KB – 1 MB |
| Batas thread GCD per QoS pool | ~64 |
| Ukuran cooperative pool | ≈ jumlah core CPU |
| `httpMaximumConnectionsPerHost` default | 6 (iOS) |
| Timeout `URLSession` default | 60 detik |
| Frame budget 60 Hz / 120 Hz | 16,7 ms / 8,3 ms |

---

## Kalimat siap pakai

**ARC:** *"Bukan garbage collector — retain/release yang disisipkan compiler saat
kompilasi. Deterministik, dan karena itu buta terhadap siklus."*

**`weak`:** *"Objeknya dapat side table; weak variable menunjuk ke sana, bukan ke objek —
itu sebabnya ia bisa jadi nil dengan aman."*

**COW:** *"Value type menunda penyalinan sampai ada mutasi, dengan mengecek
`isKnownUniquelyReferenced` sebelum menulis."*

**Actor:** *"Menjamin eksekusi terserialkan, bukan urutan, dan bukan atomisitas
melintasi await — itu reentrancy."*

**`any` vs `some`:** *"`any` membungkus dalam existential container minimal 40 byte
dengan dispatch lewat witness table; `some` bisa dispesialisasi jadi tipe konkret
kalau optimizer melihatnya."*

**Diffable:** *"Ia menghapus kelas bug 'invalid number of items' dengan menghapus
penyebabnya — dua sumber kebenaran."*

**Arsitektur:** *"Saya pilih X, menolak Y karena Z, dan harga yang saya bayar adalah W —
itu akan mulai terasa saat kondisi V terpenuhi."*
