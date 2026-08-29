---
title: "Diffable Data Source & Compositional Layout: Kenapa `reloadData()` Sudah Usang"
category: "iOS & UIKit"
status: complete
tags:
  - swift-mastering
  - ios/uikit
  - article
---

# Diffable Data Source & Compositional Layout: Kenapa `reloadData()` Sudah Usang

> **Kategori:** iOS & UIKit · **Level:** Menengah ·
> **Prasyarat:** `UICollectionView` / `UITableView` dasar, `Hashable`
> **Baca ~20 menit**

---

## Masalah yang diselesaikan diffable data source

API lama berbasis index. Kamu berjanji lewat `numberOfItemsInSection`, dan UIKit
mempercayaimu. Kalau janjimu meleset satu, app-mu mati:

```
Terminating app due to uncaught exception 'NSInternalInconsistencyException',
reason: 'Invalid update: invalid number of items in section 0. The number of items
contained in an existing section after the update (5) must be equal to the number
of items contained in that section before the update (4), plus or minus the number
of items inserted or deleted from that section (0 inserted, 0 deleted)...'
```

Setiap engineer iOS pernah melihat pesan ini. Penyebabnya selalu sama:
**dua sumber kebenaran** — array kamu dan pemahaman UIKit tentang array kamu —
yang harus dijaga sinkron secara manual lewat `performBatchUpdates`.

Diffable data source menghapus masalahnya di akar: kamu tidak lagi memberi tahu UIKit
*apa yang berubah*. Kamu memberikan **snapshot keadaan sekarang**, dan UIKit
menghitung sendiri perbedaannya.

---

## 1. Bentuk dasarnya

```swift
// HomeViewController — dari project movie
private var dataSource: UICollectionViewDiffableDataSource<Int, Album>!

dataSource = UICollectionViewDiffableDataSource<Int, Album>(collectionView: collectionView) {
    collectionView, indexPath, album in
    let cell = collectionView.dequeueReusableCell(
        withReuseIdentifier: AlbumPosterCell.reuseIdentifier, for: indexPath) as! AlbumPosterCell
    cell.configure(with: album)
    return cell
}

private func applySnapshot() {
    var snapshot = NSDiffableDataSourceSnapshot<Int, Album>()
    for (index, shelf) in viewModel.shelves.enumerated() {
        snapshot.appendSections([index])
        snapshot.appendItems(shelf.albums, toSection: index)
    }
    dataSource.apply(snapshot, animatingDifferences: false)
}
```

Perhatikan apa yang **hilang** dibanding kode lama:

- Tidak ada `numberOfSections`
- Tidak ada `numberOfItemsInSection`
- Tidak ada `performBatchUpdates`
- Tidak ada `insertItems`/`deleteItems`/`moveItem`
- Tidak ada `reloadData()`

Ada satu closure untuk membuat cell, dan satu fungsi untuk menyerahkan snapshot.

---

## 2. `Hashable` bukan formalitas — ia identitas

Signature-nya `UICollectionViewDiffableDataSource<SectionIdentifier, ItemIdentifier>`,
dan keduanya harus `Hashable`. Kata **identifier** itu penting: yang disimpan snapshot
bukan datanya, tapi **identitas** item.

Diff-nya bekerja dengan membandingkan hash. Karena itu implementasi `Hashable`-mu
menentukan perilaku UI:

```swift
struct Track: Codable, Hashable, Identifiable {
    let id: Int
    let title: String
    ...
}
```

`Hashable` sintesis default memakai **semua** stored property. Konsekuensinya:

| Yang berubah | Perilaku diffable |
|---|---|
| Tidak ada | Tidak ada perubahan UI |
| Satu field (mis. `title`) | Item dianggap **berbeda** → hapus yang lama, sisipkan yang baru |
| Urutan | Item dipindahkan (animasi move) |

Perhatikan baris kedua. Kalau kamu mengubah satu field, animasinya adalah
delete + insert, bukan "reload baris ini". Untuk banyak kasus itu tidak apa-apa,
tapi kalau kamu ingin animasi yang benar, buat `Hashable` yang hanya memakai identitas:

```swift
struct Track: Hashable {
    let id: Int
    let title: String

    static func == (l: Track, r: Track) -> Bool { l.id == r.id }
    func hash(into hasher: inout Hasher) { hasher.combine(id) }
}
```

Lalu untuk memperbarui isi item yang identitasnya sama:

```swift
snapshot.reconfigureItems([track])   // iOS 15+, tanpa membuat ulang cell
snapshot.reloadItems([track])        // lebih tua, membuat ulang cell
```

`reconfigureItems` memanggil ulang cell provider pada cell yang **sudah ada** —
jauh lebih murah dan tidak menghilangkan state seperti posisi scroll di dalam cell.

### Jebakan: item duplikat = crash

```
Fatal: Supplied identifiers are not unique.
```

Snapshot tidak boleh punya dua item dengan hash yang sama. Kalau API-mu mengembalikan
dua entri dengan `id` yang sama (dan iTunes Search bisa!), kamu harus men-dedup
sebelum `apply`:

```swift
var seen = Set<Int>()
let unique = tracks.filter { seen.insert($0.id).inserted }
```

Ini pertanyaan follow-up yang umum: *"Apa yang terjadi kalau server mengirim duplikat?"*
Jawaban "crash, jadi saya dedup di layer ViewModel" menunjukkan kamu pernah
mengalaminya sungguhan.

---

## 3. `apply` itu aman dipanggil dari mana saja — tapi konsisten

Sejak iOS 15, `apply(_:animatingDifferences:)` boleh dipanggil dari background thread,
**asal kamu konsisten**: kalau snapshot pertama dipanggil dari main, semuanya harus
dari main. Mencampur akan memicu assertion.

Aturan praktis paling aman: tandai view controller-mu `@MainActor` dan selalu
`apply` dari sana. Compiler yang akan menegakkannya, bukan ingatanmu.

Perhatikan juga di kode `movie`:

```swift
dataSource.apply(snapshot, animatingDifferences: false)
```

`false` untuk pemuatan pertama itu tepat — menganimasikan 20 item yang muncul dari
ketiadaan terlihat berantakan. Pakai `true` untuk perubahan berikutnya, di mana
animasi menyampaikan informasi ("satu item baru masuk di atas").

---

## 4. Supplementary view: bagian yang mudah salah

```swift
dataSource.supplementaryViewProvider = { [weak self] collectionView, kind, indexPath in
    guard let self, kind == UICollectionView.elementKindSectionHeader else { return nil }
    let header = collectionView.dequeueReusableSupplementaryView(
        ofKind: kind,
        withReuseIdentifier: ShelfHeaderView.reuseIdentifier,
        for: indexPath) as! ShelfHeaderView
    header.configure(title: self.viewModel.shelves[indexPath.section].title)
    return header
}
```

Dua hal:

1. **`[weak self]` wajib** — closure ini disimpan oleh `dataSource`, dan `dataSource`
   dimiliki view controller. Tanpa `weak`: siklus.
2. **Judul header dibaca dari view model, bukan dari snapshot.** Ini bekerja karena
   section identifier-nya `Int` (indeks), dan `shelves` selalu di-update sebelum
   `applySnapshot()`. Ini titik lemah yang halus: kalau suatu saat snapshot dan
   `shelves` bisa tidak sinkron, header akan menampilkan judul yang salah.

Desain yang lebih tahan banting: jadikan section identifier-nya tipe yang **memuat**
judulnya, sehingga tidak ada dua sumber kebenaran.

```swift
struct ShelfID: Hashable { let index: Int; let title: String }
var dataSource: UICollectionViewDiffableDataSource<ShelfID, Album>!
// header.configure(title: dataSource.sectionIdentifier(for: indexPath.section)?.title ?? "")
```

---

## 5. Compositional Layout: menyusun dari empat konsep

```
Item     → satu sel
  ↓
Group    → susunan item (horizontal / vertical / custom)
  ↓
Section  → kumpulan group + perilaku (scroll, inset, header/footer)
  ↓
Layout   → kumpulan section
```

Contoh "shelf" ala Netflix/Spotify dari project `movie`:

```swift
private static func makeShelfSection() -> NSCollectionLayoutSection {
    let item = NSCollectionLayoutItem(
        layoutSize: NSCollectionLayoutSize(widthDimension: .absolute(120),
                                           heightDimension: .absolute(170)))

    let group = NSCollectionLayoutGroup.horizontal(
        layoutSize: NSCollectionLayoutSize(widthDimension: .absolute(120),
                                           heightDimension: .absolute(170)),
        subitems: [item])

    let section = NSCollectionLayoutSection(group: group)
    section.orthogonalScrollingBehavior = .continuous     // ← baris ajaibnya
    section.interGroupSpacing = 12
    section.contentInsets = NSDirectionalEdgeInsets(top: 8, leading: 16, bottom: 24, trailing: 16)
    section.boundarySupplementaryItems = [
        NSCollectionLayoutBoundarySupplementaryItem(
            layoutSize: NSCollectionLayoutSize(widthDimension: .fractionalWidth(1),
                                               heightDimension: .estimated(36)),
            elementKind: UICollectionView.elementKindSectionHeader,
            alignment: .top)
    ]
    return section
}
```

**`orthogonalScrollingBehavior = .continuous`** adalah alasan utama compositional layout
ada. Sebelum iOS 13, "collection view horizontal di dalam sel collection view vertikal"
butuh nested collection view, dengan data source terpisah, delegate terpisah, dan
manajemen state scroll manual. Sekarang: satu baris.

### Tiga jenis dimensi

| | Arti | Pakai untuk |
|---|---|---|
| `.absolute(120)` | 120 pt persis | Poster, thumbnail berukuran tetap |
| `.fractionalWidth(0.5)` | 50% dari lebar **container** | Grid responsif |
| `.estimated(36)` | Tebakan awal; ukuran final dari Auto Layout | Teks yang tingginya bergantung isi & Dynamic Type |

`.estimated` penting untuk aksesibilitas: header dengan tinggi `.absolute(36)`
akan terpotong saat user menaikkan ukuran teks sistem. Project ini memakai
`.estimated(36)` untuk header — pilihan yang benar.

### Grid responsif

```swift
let item = NSCollectionLayoutItem(layoutSize: .init(
    widthDimension: .fractionalWidth(1.0), heightDimension: .fractionalHeight(1.0)))
item.contentInsets = .init(top: 4, leading: 4, bottom: 4, trailing: 4)

let group = NSCollectionLayoutGroup.horizontal(
    layoutSize: .init(widthDimension: .fractionalWidth(1.0), heightDimension: .absolute(120)),
    repeatingSubitem: item, count: 3)              // ← 3 kolom
```

Untuk jumlah kolom yang berubah menurut lebar layar, pakai closure
`UICollectionViewCompositionalLayout { sectionIndex, environment in ... }` dan baca
`environment.container.effectiveContentSize.width`. Itu cara benar mendukung iPad
dan Split View — bukan mengecek `UIDevice.current.userInterfaceIdiom`.

---

## 6. Kapan TIDAK memakai diffable

Jujur, ini jarang dibahas:

| Situasi | Alasan |
|---|---|
| Daftar sangat besar (>10.000) yang berubah total tiap kali | Perhitungan diff sendiri jadi biaya. Kadang `reloadData()` memang lebih cepat |
| Item tidak punya identitas stabil | Diffable akan menganggap semuanya baru setiap kali |
| Deployment target < iOS 13 | Tidak tersedia |

Untuk daftar besar dengan perubahan total, ukur dulu: `apply(animatingDifferences: false)`
pada snapshot besar sudah cukup cepat di kebanyakan kasus, karena UIKit mengambil
jalur cepat saat animasi dimatikan.

---

## 7. Cek pemahaman

**Q1.** Kenapa item model untuk diffable sebaiknya `struct`, bukan `class`?
<details><summary>Jawaban</summary>

Karena `Hashable` sintesis pada class memakai identitas referensi kalau kamu tidak
mengimplementasikannya, dan lebih penting: dua cell bisa memegang objek yang **sama**,
sehingga mutasi lewat satu cell terlihat di cell lain tanpa snapshot baru. Value type
membuat setiap snapshot benar-benar merepresentasikan satu keadaan.
</details>

**Q2.** Apa bedanya `reloadItems` dan `reconfigureItems`?
<details><summary>Jawaban</summary>

`reloadItems` membuang cell dan membuat ulang lewat cell provider (state internal cell
hilang, ada animasi fade). `reconfigureItems` (iOS 15+) memanggil ulang cell provider
pada cell yang **sudah terpasang** — lebih murah dan mempertahankan state. Pakai
`reconfigure` kecuali kamu benar-benar butuh cell baru.
</details>

**Q3.** Kamu punya section header yang harus menampilkan jumlah item di section itu.
Di mana kamu membacanya?
<details><summary>Jawaban</summary>

Dari `dataSource.snapshot().numberOfItems(inSection:)`, bukan dari array view model.
Snapshot adalah kebenaran yang sedang ditampilkan; array view model bisa sudah berubah
sebelum `apply` berikutnya.
</details>

---

## Ringkasan

- Diffable menghapus kelas bug "invalid number of items" dengan menghapus penyebabnya:
  dua sumber kebenaran.
- `Hashable` bukan formalitas — ia mendefinisikan identitas. `Hashable` yang memakai
  semua field membuat perubahan konten jadi delete+insert.
- Item duplikat = crash. Dedup di ViewModel.
- `reconfigureItems` > `reloadItems` untuk memperbarui isi item yang identitasnya sama.
- Compositional layout: Item → Group → Section → Layout.
  `orthogonalScrollingBehavior` menghapus kebutuhan nested collection view.
- Pakai `.estimated` untuk apa pun yang tingginya bergantung teks — itu syarat Dynamic Type.
- Closure cell provider & supplementary provider disimpan data source → butuh `[weak self]`.

## Selanjutnya

→ [03 — Networking: `URLSession`, `Codable`, dan Lossy Decoding](03-networking-urlsession-codable.md)

## Sumber

- Project `movie`: `HomeViewController`, `SearchViewController`, `AlbumPosterCell`
- [WWDC19 — Advances in UI Data Sources](https://developer.apple.com/videos/play/wwdc2019/220/)
- [WWDC19 — Advances in Collection View Layout](https://developer.apple.com/videos/play/wwdc2019/215/)
- [WWDC21 — Make blazing fast lists and collection views](https://developer.apple.com/videos/play/wwdc2021/10252/)
