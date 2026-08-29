---
title: "Interview Prep 04 — UIKit & iOS Platform"
category: "Interview Prep"
status: complete
tags:
  - swift-mastering
  - interview-prep
  - interview
---

# Interview Prep 04 — UIKit & iOS Platform

> 🟢 Junior · 🟡 Mid · 🔴 Senior — [cara pakai](README.md)

---

## 🟢 1. Urutan lifecycle `UIViewController`, dan apa yang dilakukan di masing-masing

<details><summary>Jawaban model</summary>

```
init(coder:) / init(nibName:bundle:)
   ↓
loadView()                 ← buat view hierarchy secara manual (jarang di-override)
   ↓
viewDidLoad()              ← SEKALI. Setup yang tidak bergantung ukuran: subview,
                             constraint, binding, register cell
   ↓
viewWillAppear(_:)         ← SETIAP KALI muncul. Refresh data, mulai observer
   ↓
viewWillLayoutSubviews()
viewDidLayoutSubviews()    ← ukuran sudah final. Layer/corner radius/gradient di sini
   ↓
viewDidAppear(_:)          ← animasi mulai, analytics, request izin
   ↓
viewWillDisappear(_:) / viewDidDisappear(_:)  ← hentikan timer, simpan draft
   ↓
deinit
```

Kesalahan paling umum: menaruh sesuatu yang bergantung ukuran (mis. `layer.cornerRadius`
berdasarkan `bounds.width`) di `viewDidLoad`, di mana `bounds` masih ukuran dari
storyboard/nib, bukan ukuran layar sebenarnya.
</details>

**Follow-up:** *"Kenapa `viewDidLayoutSubviews` bisa dipanggil berkali-kali?"*
→ Setiap rotasi, perubahan safe area, keyboard muncul, atau `setNeedsLayout` memicunya.
Jadi jangan menaruh kode yang mahal atau tidak idempoten di sana.

---

## 🟢 2. Bagaimana cell reuse bekerja, dan bug apa yang paling sering muncul?

<details><summary>Jawaban model</summary>

`UITableView`/`UICollectionView` hanya membuat cell sebanyak yang muat di layar plus
sedikit buffer. Saat cell keluar layar, ia masuk reuse queue dan dikeluarkan lagi
lewat `dequeueReusableCell` — **dengan isi lama masih menempel**.

Bug klasik: **gambar ketuker saat scroll cepat.**

```swift
func configure(with vm: TrackRowViewModel) {
    titleLabel.text = vm.title
    imageTask = URLSession.shared.dataTask(with: vm.artworkURL!) { data, _, _ in
        DispatchQueue.main.async { self.artworkImageView.image = UIImage(data: data!) }
    }
    imageTask?.resume()
}
```

Kalau cell di-reuse sebelum request selesai, gambar untuk baris lama muncul di baris baru.

Dua perbaikan yang harus disebutkan bersama:

1. **Batalkan di `prepareForReuse`:**
```swift
override func prepareForReuse() {
    super.prepareForReuse()
    imageTask?.cancel()
    artworkImageView.image = nil
    titleLabel.text = nil
}
```

2. **Verifikasi identitas saat hasil datang** — karena pembatalan bisa terlambat:
```swift
let requestedID = vm.id
... { image in
    guard self.trackID == requestedID else { return }   // cell sudah dipakai baris lain
    self.artworkImageView.image = image
}
```
</details>

**Follow-up:** *"Apa lagi yang harus dibersihkan di `prepareForReuse`?"*
→ delegate, state seleksi custom, accessory view, animasi yang sedang jalan,
observer/KVO, dan constraint yang ditambahkan secara kondisional.

---

## 🟡 3. `frame` vs `bounds`

<details><summary>Jawaban model</summary>

- `frame` = posisi + ukuran view dalam koordinat **superview**-nya.
- `bounds` = posisi + ukuran dalam koordinat **dirinya sendiri**; `origin` biasanya
  `(0,0)` kecuali view-nya scroll.

Perbedaan yang paling sering diuji: **saat view dirotasi (transform)**, `frame` menjadi
bounding box dari view yang miring (jadi lebih besar), sementara `bounds` tidak berubah.
Karena itu jangan pernah menyetel `frame` pada view yang punya transform non-identity.

Dan `UIScrollView.bounds.origin` adalah `contentOffset` — itu cara scroll view bekerja:
ia menggeser sistem koordinatnya sendiri, bukan memindahkan subview-nya.
</details>

---

## 🟡 4. Auto Layout: content hugging vs compression resistance

<details><summary>Jawaban model</summary>

- **Content hugging** = seberapa kuat view menolak **membesar** melebihi intrinsic size.
  Prioritas tinggi → "jangan lebarkan aku".
- **Compression resistance** = seberapa kuat view menolak **mengecil** di bawah
  intrinsic size. Prioritas tinggi → "jangan potong aku".

Kasus klasik: dua label bersebelahan, salah satunya harus di-truncate. Naikkan
compression resistance label yang **harus** utuh, atau turunkan yang boleh terpotong.

Contoh nyata di sebuah track cell: tombol save diberi
`setContentHuggingPriority(.required, for: .horizontal)` supaya ia tidak melar mengisi
ruang sisa — ruang itu diberikan ke label judul.
</details>

**Follow-up:** *"Apa arti constraint dengan priority 999 vs 1000?"* → 1000 (`.required`)
tidak boleh dilanggar; kalau konflik, kamu dapat log "Unable to simultaneously satisfy
constraints" dan Auto Layout membuang salah satunya. 999 boleh dilanggar sedikit —
sering dipakai untuk constraint yang harus mengalah saat ada temporary conflict
(mis. cell yang sedang berubah tinggi).

---

## 🟡 5. `setNeedsLayout` vs `layoutIfNeeded` vs `setNeedsDisplay`

<details><summary>Jawaban model</summary>

- `setNeedsLayout()` — menandai view butuh layout ulang. **Asinkron**: dikerjakan
  di layout pass berikutnya (akhir run loop).
- `layoutIfNeeded()` — **sinkron**: paksa layout pass sekarang juga. Dipakai saat kamu
  butuh ukuran final segera, atau di dalam blok animasi supaya perubahan constraint
  ikut teranimasi.
- `setNeedsDisplay()` — menandai perlu **menggambar ulang** (memicu `draw(_:)`),
  bukan layout.

Pola animasi constraint yang benar:
```swift
heightConstraint.constant = 200
UIView.animate(withDuration: 0.3) { self.view.layoutIfNeeded() }   // ← wajib
```
Tanpa `layoutIfNeeded()` di dalam blok, perubahan constraint akan "meloncat" bukan
teranimasi.
</details>

---

## 🟡 6. Kenapa `reloadData()` bukan lagi cara yang benar?

<details><summary>Jawaban model</summary>

Karena API index-based punya **dua sumber kebenaran**: array kamu dan pemahaman UIKit
tentang array kamu. Menjaga keduanya sinkron lewat `performBatchUpdates` manual adalah
sumber crash `NSInternalInconsistencyException: Invalid update: invalid number of items`.

`UICollectionViewDiffableDataSource` menghapus penyebabnya: kamu menyerahkan **snapshot
keadaan sekarang**, UIKit menghitung sendiri diff-nya, dan animasinya benar tanpa kamu
menghitung insert/delete/move.

Syaratnya: section identifier dan item identifier harus `Hashable`, dan **unik** —
item duplikat menghasilkan crash "Supplied identifiers are not unique".
</details>

**Follow-up:** *"Kalau kamu mengubah satu field item, apa yang terjadi?"*
→ Dengan `Hashable` sintesis (semua property), item dianggap **berbeda** → delete + insert.
Untuk memperbarui isi tanpa mengganti cell: implementasikan `Hashable` berdasarkan `id`
saja, lalu `snapshot.reconfigureItems([item])` (iOS 15+).

📖 [Diffable Data Source](../articles/ios-uikit/02-diffable-data-source-compositional-layout.md)

---

## 🟡 7. `URLSession` tidak melempar untuk HTTP 500. Apa implikasinya?

<details><summary>Jawaban model</summary>

404 dan 500 adalah respons HTTP yang **berhasil** dari sisi jaringan — `URLSession`
hanya melempar untuk kegagalan transport (tidak ada koneksi, timeout, DNS gagal).

Jadi kamu **wajib** memeriksa status code sendiri:

```swift
guard let http = response as? HTTPURLResponse,
      (200...299).contains(http.statusCode) else {
    throw ServiceError.http(statusCode: (response as? HTTPURLResponse)?.statusCode ?? -1)
}
```

Tanpa ini, kamu akan mencoba men-decode body error (sering HTML) sebagai JSON dan
melaporkan "decoding failed" — pesan yang menyesatkan saat debugging produksi.
</details>

---

## 🟡 8. Bagaimana kamu menangani API yang mengembalikan satu entri rusak di tengah array?

<details><summary>Jawaban model</summary>

`Array`'s `Decodable` melempar begitu **satu** elemen gagal — seluruh layar jatuh
karena satu entri nyasar.

**Lossy decoding:** bungkus setiap elemen dalam tipe yang `init(from:)`-nya tidak
pernah melempar.

```swift
private struct FailableTrack: Decodable {
    let track: Track?
    init(from decoder: Decoder) throws { track = try? Track(from: decoder) }
}

struct TrackSearchResponse: Decodable {
    let results: [Track]
    init(from decoder: Decoder) throws {
        let c = try decoder.container(keyedBy: CodingKeys.self)
        results = try c.decode([FailableTrack].self, forKey: .results).compactMap(\.track)
    }
}
```

**Harganya, dan ini harus kamu sebutkan:** lossy decoding menyembunyikan kegagalan.
Kalau API berubah total, app menampilkan daftar kosong secara diam-diam. Mitigasinya:
log setiap elemen yang dibuang, dan perlakukan "semua elemen gagal" sebagai error,
bukan sebagai hasil kosong.
</details>

📖 [Networking & Lossy Decoding](../articles/ios-uikit/03-networking-urlsession-codable.md)

---

## 🔴 9. App kamu lambat saat scroll. Bagaimana kamu mendiagnosisnya?

<details><summary>Jawaban model</summary>

**Ukur dulu, jangan menebak.** Instruments → Time Profiler + Core Animation.

Urutan tersangka, dari yang paling sering:

1. **Kerja di main thread saat `cellForRow`** — decoding gambar, parsing tanggal,
   `NSAttributedString` dari HTML. Pindahkan ke background, cache hasilnya.
2. **Gambar tidak di-*downsample*.** Memuat JPEG 4000×3000 ke `UIImageView` 100×100
   berarti decoding penuh + resize per frame. Pakai
   `CGImageSourceCreateThumbnailAtIndex` dengan `kCGImageSourceThumbnailMaxPixelSize`.
3. **Offscreen rendering** — `layer.shadowPath` tidak diset (shadow dihitung dari
   alpha channel tiap frame), `masksToBounds` + `cornerRadius` pada view yang kompleks.
   Core Animation instrument punya opsi "Color Offscreen-Rendered Yellow".
4. **Blended layers** — view transparan bertumpuk. "Color Blended Layers" menampilkan
   merah. Set `isOpaque = true` + `backgroundColor` solid.
5. **Auto Layout yang terlalu dalam** — solver bekerja per frame. Untuk cell yang
   sangat kompleks, kadang layout manual atau `UICollectionViewCompositionalLayout`
   dengan ukuran `.absolute` lebih cepat.
6. **Self-sizing cell dengan `.estimated` yang jauh meleset** — memaksa perhitungan ulang.

Yang membedakan jawaban senior: menyebutkan **urutan mengukur** dan bahwa nomor 1
biasanya penyebabnya, bukan langsung melompat ke micro-optimization.
</details>

---

## 🔴 10. Desain image loader untuk feed dengan ribuan gambar

<details><summary>Jawaban model</summary>

Komponennya:

1. **Dua lapis cache.** `NSCache<NSURL, UIImage>` untuk gambar yang sudah di-decode
   (in-memory, otomatis dibuang saat memory pressure), plus cache disk untuk data
   mentah. `URLCache` menangani lapisan HTTP secara gratis kalau server mengirim header
   cache yang benar.
2. **Deduplikasi request in-flight.** Dua cell yang meminta URL sama harus berbagi satu
   unduhan. Simpan `Task` di dictionary — ini kasus buku teks untuk
   [reentrancy actor](../articles/concurrency/03-actor-isolation-reentrancy.md):
   tulis entri `.inProgress` **sebelum** `await` pertama.
3. **Downsampling saat decode**, bukan setelah. `CGImageSourceCreateThumbnailAtIndex`
   dengan `kCGImageSourceShouldCacheImmediately: true` supaya decoding terjadi di
   background thread, bukan saat pertama kali dirender.
4. **Prefetching.** `UICollectionViewDataSourcePrefetching` untuk memulai unduhan
   beberapa baris sebelum terlihat, dan `cancelPrefetchingForItemsAt` untuk membatalkannya.
5. **Pembatalan di `prepareForReuse`** + verifikasi identitas saat hasil datang.
6. **Bounded concurrency** — 4–6 unduhan bersamaan; lebih dari itu hanya mengisi antrian
   `URLSession`.

Kalau ditanya "kenapa tidak pakai Kingfisher/SDWebImage?": jawaban yang baik adalah
"untuk produksi saya kemungkinan pakai, karena mereka sudah menyelesaikan keenam hal
ini plus edge case yang tidak terpikirkan. Yang penting adalah saya tahu apa yang
mereka lakukan, supaya bisa men-debug saat berperilaku aneh."
</details>

---

## 🔴 11. Kapan kamu memilih Core Data, SwiftData, GRDB, atau `UserDefaults`?

<details><summary>Jawaban model</summary>

| Pilihan | Untuk | Hindari kalau |
|---|---|---|
| `UserDefaults` | Preferensi, flag, data kecil (< ~100 KB) | Data terstruktur, banyak, atau perlu query |
| File JSON/Codable | Data kecil-menengah, skema sederhana, mudah di-debug | Butuh query parsial, atau data besar (harus dibaca seluruhnya) |
| GRDB (SQLite) | Kontrol penuh, query SQL, migrasi eksplisit, performa terprediksi | Tim tidak nyaman dengan SQL |
| Core Data | Graf objek kompleks, relasi, `NSFetchedResultsController`, sinkronisasi CloudKit | Skema sederhana (overhead-nya besar) |
| SwiftData | Project SwiftUI baru, skema sederhana-menengah | Butuh kontrol migrasi yang rumit, atau target iOS lama |

Yang menunjukkan pengalaman: menyebut **migrasi** sebagai kriteria utama.
Core Data lightweight migration menyelesaikan banyak kasus tapi gagal diam-diam pada
perubahan yang tidak didukung; GRDB memaksamu menulis migrasi eksplisit — lebih banyak
kerja di awal, jauh lebih sedikit kejutan di produksi.

Di project kecil, `UserDefaults` + `Codable` sering jawaban yang benar:

```swift
private func persist() {
    guard let data = try? JSONEncoder().encode(savedTracks) else { return }
    UserDefaults.standard.set(data, forKey: defaultsKey)
}
```

Batasnya jelas dan harus kamu sebutkan: seluruh array ditulis ulang setiap perubahan,
tidak ada query, tidak ada migrasi. Untuk daftar "lagu tersimpan" berukuran puluhan,
itu tepat. Untuk ribuan, tidak.
</details>

---

## Latihan cepat

1. Apa bedanya `viewWillAppear` dan `viewDidLoad` untuk memuat ulang data?
2. Di mana kamu menaruh `layer.cornerRadius = bounds.width / 2`?
3. Kenapa `UIScrollView.contentOffset` sebenarnya `bounds.origin`?
4. Apa yang harus dibersihkan di `prepareForReuse`?
5. Kenapa Instruments → Leaks tidak menangkap array yang terus tumbuh?

<details><summary>Kunci</summary>

1. `viewDidLoad` sekali seumur hidup VC; `viewWillAppear` setiap kali muncul —
   termasuk saat kembali dari layar lain, yang biasanya justru saat data perlu di-refresh.
2. `viewDidLayoutSubviews` (atau `layoutSubviews` di custom view) — di `viewDidLoad`,
   `bounds` belum final.
3. Karena scroll view menggeser sistem koordinatnya sendiri alih-alih memindahkan
   subview-nya; menaikkan `bounds.origin.y` membuat konten tampak naik.
4. Batalkan request gambar, kosongkan image/text, lepas delegate, hentikan animasi,
   lepas observer, reset state seleksi/accessory custom.
5. Karena itu bukan leak — memorinya masih terjangkau. Itu *abandoned memory*, dan
   ditemukan dengan Allocations + Mark Generation, bukan Leaks.
</details>
