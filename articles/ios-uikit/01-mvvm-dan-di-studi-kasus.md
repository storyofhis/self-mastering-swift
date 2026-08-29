---
title: "MVVM & Dependency Injection: Studi Kasus Project `movie`"
category: "iOS & UIKit"
status: complete
tags:
  - swift-mastering
  - ios/uikit
  - article
---

# MVVM & Dependency Injection: Studi Kasus Project `movie`

> **Kategori:** iOS & UIKit · **Level:** Menengah ·
> **Prasyarat:** paham UIKit dasar, closure, protocol
> **Baca ~25 menit**

---

## Kenapa artikel ini memakai satu project nyata

Artikel arsitektur biasanya memakai contoh `TodoApp` yang steril, di mana setiap pola
terlihat sempurna karena tidak ada tekanan nyata. Yang membuat kamu lulus interview
senior bukan hafal diagram MVVM — tapi bisa menjawab:

> "Kenapa kamu memilih ini, apa yang kamu tolak, dan **apa harga** yang kamu bayar?"

Project `movie` (UIKit + MVVM + async/await, klien iTunes Search API) punya
`DECISIONS.md` yang menjawab persis pertanyaan itu. Artikel ini membedahnya.

---

## 1. Anatomi lapisan

```
┌───────────────────────────────────────────────────────────────┐
│  View                    HomeViewController                    │
│                          SearchViewController                  │
│                          TrackCell, AlbumPosterCell            │
│                          ↕ (closure binding + delegate)        │
├───────────────────────────────────────────────────────────────┤
│  ViewModel               HomeViewModel                         │
│                          SearchViewModel                       │
│                          TrackRowViewModel  (display model)    │
│                          ↕ (protocol)                          │
├───────────────────────────────────────────────────────────────┤
│  Service                 MusicSearching  ──▶ iTunesService     │
│                          LibraryStore                          │
├───────────────────────────────────────────────────────────────┤
│  Model                   Track, Album, TrackSearchResponse     │
└───────────────────────────────────────────────────────────────┘
```

Aturan yang ditegakkan project ini:

- **ViewModel tidak boleh `import UIKit`.** Ia hanya `import Foundation`.
- **View tidak boleh tahu networking.** `HomeViewController` tidak punya satu pun
  string query di dalamnya.
- **ViewModel bergantung pada protokol**, bukan implementasi konkret.

### Detail yang membuktikan aturan itu ditegakkan sungguhan

```swift
/// Takes plain Int/Int instead of IndexPath on purpose: IndexPath itself is
/// Foundation, but its .section/.item accessors are added by a UIKit
/// extension — accepting IndexPath here would quietly require `import
/// UIKit` in a ViewModel that's supposed to stay UIKit-free.
func album(inShelf section: Int, at item: Int) -> Album? { ... }
```

Ini bukan pedantry. `IndexPath` memang tipe Foundation, tapi `.section` dan `.item`
datang dari kategori UIKit. Menerima `IndexPath` akan menyeret `import UIKit` ke
ViewModel — dan begitu itu terjadi, tidak ada lagi yang menghalangi orang berikutnya
menaruh `UIAlertController` di sana.

**Pelajarannya:** batas arsitektur bertahan karena hal-hal kecil seperti ini,
bukan karena diagram di README.

---

## 2. Binding: closure, bukan Combine, bukan delegate

```swift
final class HomeViewModel {
    var onStateChange: ((HomeState) -> Void)?
    private(set) var shelves: [Shelf] = []
}

// HomeViewController
private func bindViewModel() {
    viewModel.onStateChange = { [weak self] state in
        guard let self else { return }
        switch state {
        case .loading:            self.stateView.showLoading()
        case .loaded:             self.stateView.hide(); self.applySnapshot()
        case .empty:              self.stateView.showMessage("No albums found.")
        case .error(let message): self.stateView.showMessage(message)
        }
    }
}
```

Tiga hal yang benar di sini, dan layak kamu tiru:

**(a) State sebagai `enum`, bukan kumpulan boolean.**

```swift
enum HomeState: Equatable { case loading, loaded, empty, error(String) }
```

Bandingkan dengan `isLoading`/`isEmpty`/`errorMessage` terpisah, yang memungkinkan
state mustahil seperti "sedang loading DAN kosong DAN error". Enum membuat state
mustahil **tidak bisa direpresentasikan**, dan `switch` exhaustive memaksa kamu
menangani setiap kasus. Ini argumen desain yang sama dengan
[artikel enum layout](../swift-language/05-optional-dan-enum-layout.md).

**(b) `private(set)` pada data.** View bisa membaca `shelves`, tidak bisa menulisnya.
Satu kata kunci, satu kelas bug hilang.

**(c) `[weak self]` di closure yang **disimpan** ViewModel.** Wajib di sini:
`HomeViewController` → `viewModel` → `onStateChange` → closure → `self`.
Tanpa `weak`, view controller-nya tidak akan pernah di-`deinit`. Lihat
[artikel ARC](../swift-language/03-arc-dan-retain-cycle.md).

### Kapan closure kalah dari Combine/Observation

Closure `onStateChange` punya satu batas keras: **hanya satu pendengar**.
Assignment kedua menimpa yang pertama, diam-diam. Untuk app tiga layar, itu bukan
masalah. Begitu kamu butuh dua bagian UI yang mendengarkan state yang sama, kamu
butuh `@Published`/`AsyncStream`/`@Observable`.

Jawaban interview yang bagus: *"Closure cukup sampai kamu butuh multicast atau
operator komposisi. Menambahkan Combine sebelum itu adalah biaya tanpa manfaat."*

---

## 3. Dependency Injection tanpa framework

```swift
final class HomeViewModel {
    private let service: MusicSearching
    init(service: MusicSearching = iTunesService()) {
        self.service = service
    }
}

final class SearchViewModel {
    private let service: MusicSearching
    private let library: LibraryStore
    init(service: MusicSearching = iTunesService(), library: LibraryStore = .shared) {
        self.service = service
        self.library = library
    }
}
```

Inilah **initializer injection dengan default value** — pola DI paling praktis di iOS,
dan tidak butuh satu baris pun framework.

Yang dibeli:

| | Tanpa DI | Dengan DI |
|---|---|---|
| Produksi | `iTunesService()` di dalam method | `HomeViewModel()` — sama ringkasnya |
| Test | Harus benar-benar memanggil jaringan | `HomeViewModel(service: FakeService())` |
| Dokumentasi | Ketergantungan tersembunyi | Signature `init` mendaftarkannya |

Poin ketiga sering diremehkan, dan `DECISIONS.md` menyebutnya eksplisit:

> *"I kept it reachable only through each ViewModel's own `init(library:)` parameter
> rather than reading `.shared` inline inside methods, so every ViewModel's signature
> honestly documents 'I touch the library.'"*

Dan harga yang diakui jujur di dokumen yang sama:

> *"nothing stops a future contributor from bypassing that convention and reading
> `LibraryStore.shared` directly from inside a view controller — the protection is
> discipline, not the compiler."*

Kalimat kedua itulah yang membuat jawaban interview terdengar seperti engineer,
bukan seperti orang yang membaca artikel Medium.

### Utang yang belum dibayar

```swift
init(service: MusicSearching = iTunesService(),   // ✅ di balik protokol → bisa di-fake
     library: LibraryStore = .shared)             // ❌ tipe konkret → tidak bisa di-fake
```

`LibraryStore` tidak punya protokol. Konsekuensinya bisa diuji: kamu tidak bisa menulis
test untuk `SearchViewModel.toggleSave` tanpa menyentuh `UserDefaults` sungguhan.

Perbaikannya kecil dan jelas:

```swift
protocol LibraryStoring {
    func isSaved(_ track: Track) -> Bool
    @discardableResult func toggleSave(_ track: Track) -> Bool
    var savedTracks: [Track] { get }
}
extension LibraryStore: LibraryStoring {}

init(service: MusicSearching = iTunesService(), library: LibraryStoring = LibraryStore.shared)
```

Kalau interviewer bertanya "apa yang akan kamu perbaiki dari project ini?" —
ini jawaban yang konkret, terukur, dan menunjukkan kamu tahu bedanya
"pakai DI" dan "DI-nya lengkap".

---

## 4. Kapan membuat display model — dan kapan tidak

Project ini punya `TrackRowViewModel` tapi **tidak** punya `AlbumCardViewModel`.
Itu keputusan, bukan kelalaian.

```swift
struct TrackRowViewModel: Hashable {
    let id: Int
    let title: String
    let subtitle: String       // "Artist — Album", digabung dari dua field
    let duration: String       // 225000 ms → "3:45"
    let isSaved: Bool          // dari LibraryStore, bukan dari Track
    let artworkURL: URL?

    init(track: Track, isSaved: Bool) { ... }
}
```

Ada **transformasi nyata** di sini: milidetik jadi `"3:45"`, dua field opsional
digabung jadi satu subtitle, dan state dari sumber lain (`LibraryStore`) ikut masuk.
Hasilnya: `TrackCell` tidak perlu tahu apa itu `Track`, apa itu `LibraryStore`,
atau bagaimana memformat durasi. Ia hanya merender enam field siap pakai.

`Album` tidak mendapat perlakuan yang sama karena tidak ada yang perlu diubah —
satu-satunya "formatting" (upgrade URL artwork dari 100×100 ke 600×600) sudah hidup
sebagai computed property di model:

```swift
var artworkURL: URL? {
    guard let artworkPath else { return nil }
    let higherRes = artworkPath.replacingOccurrences(of: "100x100bb", with: "600x600bb")
    return URL(string: higherRes)
}
```

**Aturannya:** buat display model kalau ada transformasi, penggabungan sumber, atau
formatting. Jangan buat kalau ia cuma menyalin field satu-ke-satu — itu
"mapper chain that adds nothing".

Bonus: `TrackRowViewModel: Hashable` bukan kebetulan. Ia dibuat `Hashable` supaya
bisa langsung jadi item di diffable data source. Lihat
[artikel diffable](02-diffable-data-source-compositional-layout.md).

---

## 5. Delegation: satu-satunya arah komunikasi cell → dunia

```swift
protocol TrackCellDelegate: AnyObject {
    func trackCell(_ cell: TrackCell, didTapSaveFor trackID: Int)
}

final class TrackCell: UITableViewCell {
    weak var delegate: TrackCellDelegate?
    private var trackID: Int?

    @objc private func saveTapped() {
        guard let trackID else { return }
        delegate?.trackCell(self, didTapSaveFor: trackID)
    }
}
```

Tiga hal yang benar:

1. **`: AnyObject`** — supaya `weak var delegate` bisa dikompilasi
   ([kenapa](../swift-language/03-arc-dan-retain-cycle.md#6-sumber-leak-lain-yang-bukan-closure)).
2. **`weak`** — cell tidak memiliki delegate-nya; view controller memiliki cell.
3. **Cell tidak memutuskan apa pun.** Ia tidak memanggil `LibraryStore`, tidak
   mengubah ikonnya sendiri. Ia melapor, lalu menunggu dikonfigurasi ulang.

Poin ketiga adalah yang paling sering dilanggar orang. Cell yang mengubah state-nya
sendiri akan tidak sinkron dengan data source begitu ia di-reuse — dan itu bug
"tombol save-nya nyala di baris yang salah" yang klasik.

Alurnya benar-benar satu arah:

```
tap → TrackCell.delegate → SearchViewController
                              → SearchViewModel.toggleSave(at:)
                                 → LibraryStore.toggleSave
                                 → rows[index] diperbarui
                              → onRowsChange → tableView reload baris itu
                           → TrackCell.configure(with:) dipanggil ulang
```

---

## 6. Yang jujur diakui kurang

`DECISIONS.md` menyebut satu gap yang nyata:

> *"VIPER would have given navigation an explicit home (a Router) instead of letting
> `HomeViewController.collectionView(_:didSelectItemAt:)` call
> `navigationController?.pushViewController` directly — that's a real, honest gap in
> this MVVM implementation."*

```swift
extension HomeViewController: UICollectionViewDelegate {
    func collectionView(_ cv: UICollectionView, didSelectItemAt indexPath: IndexPath) {
        guard let album = viewModel.album(inShelf: indexPath.section, at: indexPath.item) else { return }
        navigationController?.pushViewController(AlbumDetailViewController(album: album), animated: true)
    }
}
```

View controller membangun view controller berikutnya dan mendorongnya sendiri.
Untuk satu edge navigasi (Home → AlbumDetail), itu keputusan yang benar.
Kalau layar keempat datang dengan percabangan navigasi, biayanya akan terasa sebagai
`pushViewController` yang tersebar di banyak file — dan **saat itulah** Coordinator
dibayar, bukan sebelumnya.

Ini cara berpikir yang dicari interviewer senior: bukan "pola X selalu lebih baik",
tapi "pola X mulai dibayar pada kondisi Y, dan kita belum di sana".

---

## 7. Cara menceritakan ini di interview (5 menit)

Struktur yang bekerja:

1. **Konteks (20 detik).** "App tiga tab, klien iTunes Search API, UIKit programmatic,
   MVVM, async/await."
2. **Satu keputusan + alternatif yang ditolak (90 detik).** "Saya pilih MVVM, bukan VIPER.
   VIPER akan memberi Router untuk navigasi — dan itu memang gap yang saya punya —
   tapi lima file per layar untuk tiga layar dengan satu edge navigasi tidak sepadan."
3. **Satu masalah teknis nyata + solusinya (90 detik).** "iTunes `/lookup` mengembalikan
   entri album di tengah daftar lagu. Alih-alih special-case, saya buat wrapper
   `FailableTrack` yang men-decode tiap elemen secara individual dan membuang yang gagal —
   jadi satu entri rusak tidak menjatuhkan seluruh layar."
4. **Harga yang saya bayar (60 detik).** "`LibraryStore` tidak di balik protokol, jadi
   belum bisa di-fake di test. Itu hal pertama yang akan saya perbaiki."
5. **Apa yang saya pelajari (30 detik).**

Poin 4 adalah yang paling sering dilewatkan kandidat, dan paling sering jadi
pembeda. Menyebutkan kelemahan sendiri secara spesifik menunjukkan kamu benar-benar
memahami sistemnya, bukan sekadar menyelesaikannya.

---

## 8. Cek pemahaman

**Q1.** Kenapa `HomeViewModel.album(inShelf:at:)` menerima dua `Int` alih-alih `IndexPath`?
<details><summary>Jawaban</summary>

Karena `.section` dan `.item` adalah extension UIKit pada `IndexPath`. Menerimanya
akan memaksa `import UIKit` di ViewModel dan meruntuhkan batas lapisan.
</details>

**Q2.** Apa yang rusak kalau `[weak self]` dihapus dari `bindViewModel()`?
<details><summary>Jawaban</summary>

Retain cycle: `HomeViewController` → `viewModel` → `onStateChange` → closure → `self`.
View controller-nya tidak akan pernah di-`deinit` saat di-pop, dan setiap kali user
membuka layar itu, satu instance bocor.
</details>

**Q3.** Kapan kamu akan menambahkan Coordinator ke project ini?
<details><summary>Jawaban</summary>

Saat ada lebih dari satu edge navigasi, atau saat satu layar bisa dicapai dari
beberapa tempat (deep link, notification, tab lain). Sebelum itu, Coordinator hanya
memindahkan satu baris `pushViewController` ke file baru.
</details>

---

## Ringkasan

- Batas lapisan bertahan karena detail kecil (menolak `IndexPath` di ViewModel),
  bukan karena diagram.
- State sebagai `enum` menghapus state mustahil secara struktural; `private(set)`
  menutup jalur mutasi dari view.
- DI = initializer injection dengan default value. Ia mendokumentasikan ketergantungan
  di signature, dan membuat fake mungkin — asalkan **semua** dependency di balik protokol.
- Buat display model kalau ada transformasi nyata; jangan kalau cuma menyalin field.
- Cell melapor lewat delegate `: AnyObject` + `weak`, dan tidak pernah memutuskan
  apa pun sendiri.
- Jawaban interview terbaik menyebutkan **alternatif yang ditolak** dan **harga yang dibayar**.

## Selanjutnya

→ [02 — Diffable Data Source & Compositional Layout](02-diffable-data-source-compositional-layout.md)
→ [Interview Prep: Architecture & System Design](../../interview-prep/06-architecture-system-design.md)

## Sumber

- Project `movie`: `DECISIONS.md`, `REFLECTION.md`, `HomeViewModel`, `SearchViewModel`,
  `TrackRowViewModel`, `TrackCell`, `LibraryStore`
