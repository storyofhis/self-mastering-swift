---
title: "Networking: `URLSession`, `Codable`, dan Lossy Decoding"
category: "iOS & UIKit"
status: complete
tags:
  - swift-mastering
  - ios/uikit
  - article
---

# Networking: `URLSession`, `Codable`, dan Lossy Decoding

> **Kategori:** iOS & UIKit · **Level:** Menengah ·
> **Prasyarat:** [async/await](../concurrency/02-async-await-continuation.md), `Codable` dasar
> **Baca ~20 menit**

---

## Networking layer yang benar itu kecil

Kamu tidak butuh Alamofire untuk sebagian besar app. Kamu butuh empat hal:

1. Membangun URL dengan aman
2. Menjalankan request dan memeriksa status code
3. Men-decode respons menjadi tipe domain
4. Memetakan kegagalan menjadi error yang **bisa ditampilkan ke user**

Project `movie` melakukan keempatnya dalam ~80 baris tanpa dependency.
Artikel ini membedahnya lalu menunjukkan apa yang perlu ditambahkan untuk produksi.

---

## 1. Membangun URL: `URLComponents`, bukan string interpolation

```swift
// ❌ rusak begitu term berisi spasi, &, atau karakter non-ASCII
let url = URL(string: "https://itunes.apple.com/search?term=\(term)")!
```

```swift
// ✅ movie/Services/iTunesService.swift
private func makeSearchURL(term: String, entity: MediaEntity, limit: Int) throws -> URL {
    guard var components = URLComponents(url: baseURL.appendingPathComponent("search"),
                                          resolvingAgainstBaseURL: false) else {
        throw MusicServiceError.invalidURL
    }
    components.queryItems = [
        URLQueryItem(name: "term", value: term),
        URLQueryItem(name: "media", value: "music"),
        URLQueryItem(name: "entity", value: entity.rawValue),
        URLQueryItem(name: "limit", value: "\(limit)")
    ]
    guard let url = components.url else { throw MusicServiceError.invalidURL }
    return url
}
```

`URLComponents` meng-*escape* query value secara otomatis. Cari "Taylor Swift & friends"
akan bekerja; versi interpolasi akan menghasilkan `nil` URL atau query yang salah.

Perhatikan juga `MediaEntity` sebagai enum, bukan `String`:

```swift
enum MediaEntity: String { case song; case album }
```

Typo `"sonh"` tidak akan compile. Ini pola kecil yang terbayar setiap kali.

---

## 2. Satu fungsi generic untuk semua request

```swift
private func fetch<Response: Decodable>(_ url: URL) async throws -> Response {
    let (data, response) = try await URLSession.shared.data(from: url)

    guard let httpResponse = response as? HTTPURLResponse,
          (200...299).contains(httpResponse.statusCode) else {
        throw MusicServiceError.requestFailed
    }

    do {
        return try JSONDecoder().decode(Response.self, from: data)
    } catch {
        throw MusicServiceError.decodingFailed
    }
}
```

Tiga hal yang benar:

**(a) Status code diperiksa.** Ini yang paling sering dilupakan. `URLSession` **tidak**
melempar untuk 404 atau 500 — itu respons HTTP yang sukses secara jaringan.
Tanpa `guard` ini, kamu akan mencoba men-decode halaman error HTML sebagai JSON
dan mendapat pesan error yang menyesatkan.

**(b) Return type generic yang di-infer dari pemanggil.**

```swift
let response: AlbumSearchResponse = try await fetch(url)   // tipe menentukan decoding
```

Ini generic dengan spesialisasi — nol overhead di Release
([kenapa](../swift-language/04-protocol-existential-vs-generics.md)).

**(c) Error decoding dipetakan ke error domain.** Layer di atasnya tidak perlu tahu
apa itu `DecodingError.keyNotFound`.

### Yang hilang dari versi ini (dan perlu ditambahkan untuk produksi)

```swift
// 1. JSONDecoder dibuat ulang setiap request — mahal kalau punya konfigurasi
private let decoder: JSONDecoder = {
    let d = JSONDecoder()
    d.dateDecodingStrategy = .iso8601
    return d
}()

// 2. Error asli hilang total — tidak bisa di-log untuk diagnosis
enum MusicServiceError: Error {
    case invalidURL
    case http(statusCode: Int)            // ← simpan status code-nya
    case decoding(underlying: Error)      // ← simpan error aslinya
    case transport(underlying: Error)
}

// 3. Tidak ada timeout eksplisit; default URLSession 60 detik terasa selamanya di UI
let config = URLSessionConfiguration.default
config.timeoutIntervalForRequest = 15
```

Poin 2 penting untuk debugging: `case decodingFailed` tanpa payload berarti saat
API berubah, kamu tahu "decoding gagal" tapi tidak tahu field mana. `DecodingError`
sebenarnya sangat informatif — jangan buang.

---

## 3. `Codable` di dunia nyata

```swift
// movie/Model/Track.swift
struct Track: Codable, Hashable, Identifiable {
    let id: Int
    let title: String
    let artistName: String?
    let albumName: String?
    let artworkPath: String?
    let durationMillis: Int?

    private enum CodingKeys: String, CodingKey {
        case id = "trackId"
        case title = "trackName"
        case artistName
        case albumName = "collectionName"
        case artworkPath = "artworkUrl100"
        case durationMillis = "trackTimeMillis"
    }
}
```

Empat keputusan bagus dalam 15 baris:

1. **`CodingKeys` memisahkan nama wire dari nama domain.** `trackTimeMillis` adalah
   istilah iTunes; `durationMillis` adalah istilah app-mu. Kalau API berubah, hanya
   `CodingKeys` yang disentuh.
2. **`CodingKeys` `private`.** Detail decoding tidak bocor ke luar tipe.
3. **Field yang benar-benar opsional ditandai `?`;** `id` dan `title` tidak.
   Kalau `trackId` hilang, decoding **harus** gagal — itu bukan track yang valid.
4. **Semua `let`.** Konsekuensinya: `Track` otomatis `Sendable`
   ([kenapa itu penting](../concurrency/04-sendable-data-race-safety.md)).

> `.keyDecodingStrategy = .convertFromSnakeCase` menghapus kebutuhan `CodingKeys`
> hanya untuk perbedaan snake_case↔camelCase. Untuk perbedaan **nama** seperti
> `trackName` → `title`, kamu tetap butuh `CodingKeys`. Jangan mengganti nama domain
> agar cocok dengan wire — itu membiarkan API mendikte model kamu.

---

## 4. Lossy decoding: satu entri rusak tidak boleh menjatuhkan layar

Ini bagian paling menarik dari project ini, dan masalah yang benar-benar terjadi
di API produksi mana pun.

**Masalahnya:** endpoint `/lookup?id=X&entity=song` mengembalikan tracklist album,
tapi entri **pertama**-nya adalah albumnya sendiri (`wrapperType: "collection"`,
tanpa `trackId`). Men-decode `[Track]` akan melempar di elemen pertama dan
menjatuhkan seluruh layar.

**Solusi naif** yang biasanya dipilih orang: special-case di service —
`results.dropFirst()`, atau filter berdasarkan `wrapperType`. Keduanya rapuh:
mereka mengasumsikan posisi atau field yang bisa berubah.

**Solusi project ini** — wrapper yang men-decode setiap elemen secara individual:

```swift
// movie/Model/SearchResponses.swift
private struct FailableTrack: Decodable {
    let track: Track?
    init(from decoder: Decoder) throws {
        track = try? Track(from: decoder)      // gagal → nil, bukan throw
    }
}

struct TrackSearchResponse: Decodable {
    let results: [Track]
    private enum CodingKeys: String, CodingKey { case results }

    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        let failables = try container.decode([FailableTrack].self, forKey: .results)
        results = failables.compactMap(\.track)
    }
}
```

Kenapa ini bekerja: `Array`'s `Decodable` conformance melempar begitu **satu** elemen
gagal. Dengan membungkus tiap elemen dalam tipe yang `init(from:)`-nya tidak pernah
melempar, array tetap ter-decode utuh, dan `compactMap` membuang yang kosong.

Hasilnya, seperti dicatat di komentar `iTunesService.albumTracks`:

> *"The lossy decoding in TrackSearchResponse drops it automatically since it can't
> decode as a Track — no special-casing needed here."*

**Ini pola yang layak kamu bawa ke mana-mana.** Setiap kali API mengembalikan array
heterogen atau bisa menambah tipe entri baru di masa depan, lossy decoding membuat
app kamu tahan terhadap perubahan itu.

### Harganya, yang harus kamu sebutkan

Lossy decoding **menyembunyikan** kegagalan. Kalau API berubah dan semua field
`trackName` berganti nama, aplikasi tidak crash — ia menampilkan daftar kosong,
diam-diam, dan kamu tidak akan tahu sampai user melapor.

Mitigasinya di produksi:

```swift
private struct FailableTrack: Decodable {
    let track: Track?
    init(from decoder: Decoder) throws {
        do { track = try Track(from: decoder) }
        catch {
            Logger.decoding.warning("dropped entry: \(error)")   // ← jangan diam
            track = nil
        }
    }
}
```

Dan sebuah alarm: kalau **semua** elemen gagal, itu bukan entri nyasar — itu
API yang berubah. Perlakukan sebagai error.

---

## 5. Protocol boundary: kenapa `MusicSearching` ada

```swift
protocol MusicSearching {
    func searchAlbums(term: String, limit: Int) async throws -> [Album]
    func searchTracks(term: String, limit: Int) async throws -> [Track]
    func albumTracks(albumID: Int) async throws -> [Track]
}
```

Boundary ini membuat test mungkin tanpa jaringan:

```swift
struct FakeMusicService: MusicSearching {
    var albums: [Album] = []
    var error: Error?

    func searchAlbums(term: String, limit: Int) async throws -> [Album] {
        if let error { throw error }
        return albums
    }
    // ...
}

@Test func homeViewModel_menampilkanErrorSaatRequestGagal() async {
    let vm = HomeViewModel(service: FakeMusicService(error: MusicServiceError.requestFailed))
    var captured: HomeState?
    vm.onStateChange = { captured = $0 }
    vm.load()
    // tunggu Task selesai...
    #expect(captured == .error("Couldn't load albums. Pull down to try again."))
}
```

Tanpa protokol ini, test yang sama butuh `URLProtocol` stub atau server lokal —
jauh lebih rumit untuk manfaat yang sama.

Catatan: `MusicSearching` di sini adalah **existential**, dan itu pilihan yang benar
di boundary I/O. Alasannya dibahas di
[artikel existential vs generics §6](../swift-language/04-protocol-existential-vs-generics.md).

---

## 6. Yang perlu ditambahkan untuk produksi

| Kebutuhan | Cara |
|---|---|
| Retry dengan backoff | Bungkus `fetch` dengan loop + `Task.sleep(for:)` eksponensial + jitter |
| Timeout | `URLSessionConfiguration.timeoutIntervalForRequest` |
| Caching | `URLCache` (HTTP cache) untuk GET; `NSCache` untuk gambar yang sudah di-decode |
| Auth | Header di `URLRequest`; refresh token lewat actor supaya tidak ada refresh ganda |
| Cancellation | Sudah gratis — `Task.cancel()` membatalkan `URLSession.data` |
| Offline | `NWPathMonitor` untuk membedakan "tidak ada internet" dari "server error" |
| Logging | `os.Logger` dengan privacy level; **jangan** log token atau body berisi PII |

Poin auth layak diperhatikan: refresh token adalah kasus buku teks untuk
[reentrancy actor](../concurrency/03-actor-isolation-reentrancy.md) — lima request
yang gagal 401 bersamaan akan memicu lima refresh kalau kamu tidak menyimpan
`Task` yang sedang berjalan.

---

## 7. Cek pemahaman

**Q1.** Kenapa `guard let httpResponse = response as? HTTPURLResponse, (200...299).contains(...)`
tidak boleh dilewat?
<details><summary>Jawaban</summary>

Karena `URLSession` tidak melempar untuk status error. 404 dan 500 adalah respons yang
"berhasil" dari sisi jaringan. Tanpa pengecekan, kamu akan mencoba men-decode body error
(sering HTML) sebagai JSON dan melaporkan "decoding failed" padahal masalahnya server.
</details>

**Q2.** Kapan lossy decoding jadi ide buruk?
<details><summary>Jawaban</summary>

Saat setiap elemen bermakna dan kehilangannya diam-diam berbahaya — daftar transaksi
keuangan, misalnya. Di sana kamu ingin gagal keras dan memberi tahu user, bukan
menampilkan daftar yang tidak lengkap tanpa tanda apa pun.
</details>

**Q3.** `Track.artworkURL` adalah computed property yang mengubah `100x100bb` jadi
`600x600bb`. Kenapa ini di model, bukan di ViewModel?
<details><summary>Jawaban</summary>

Karena itu bukan formatting untuk tampilan — itu memperbaiki keterbatasan API
(iTunes selalu mengembalikan thumbnail 100×100). Semua konsumen `Track` butuh URL
yang benar, bukan hanya satu layar. Kalau transformasinya bergantung konteks tampilan
(mis. ukuran berbeda per layar), barulah ia milik display model.
</details>

---

## Ringkasan

- `URLComponents` + `URLQueryItem`, bukan string interpolation. Enum untuk parameter
  yang nilainya terbatas.
- **Selalu periksa status code** — `URLSession` tidak melempar untuk 4xx/5xx.
- Satu `fetch<Response: Decodable>` generic melayani semua endpoint, nol overhead
  setelah spesialisasi.
- `CodingKeys` memisahkan nama wire dari nama domain; jangan biarkan API mendikte model.
- **Lossy decoding** membuat satu entri rusak tidak menjatuhkan layar — tapi harus
  disertai logging, kalau tidak ia menyembunyikan perubahan API.
- Protocol boundary di service = test tanpa jaringan; existential di sini gratis
  dibanding latensi I/O.

## Selanjutnya

→ [Interview Prep: UIKit & iOS Platform](../../interview-prep/04-uikit-ios.md)
→ [Actor & reentrancy](../concurrency/03-actor-isolation-reentrancy.md) — untuk pola token refresh

## Sumber

- Project `movie`: `iTunesService`, `SearchResponses`, `Track`, `MusicSearching`
- [URLSession — Apple Developer](https://developer.apple.com/documentation/foundation/urlsession)
- [WWDC21 — Use async/await with URLSession](https://developer.apple.com/videos/play/wwdc2021/10095/)
