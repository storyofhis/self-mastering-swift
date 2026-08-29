---
title: "Interview Prep 02 — Memory Management & ARC"
category: "Interview Prep"
status: complete
tags:
  - swift-mastering
  - interview-prep
  - interview
---

# Interview Prep 02 — Memory Management & ARC

> 🟢 Junior · 🟡 Mid · 🔴 Senior — [cara pakai](README.md)

Ini kategori yang paling sering menentukan hasil interview iOS level mid.
Hampir semua pewawancara punya minimal satu soal retain cycle.

---

## 🟢 1. Apa itu ARC? Apa bedanya dengan garbage collector?

<details><summary>Jawaban model</summary>

ARC = Automatic Reference Counting. Compiler menyisipkan `retain`/`release`
di **waktu kompilasi**, berdasarkan analisis lifetime.

Bedanya dengan GC:

| | ARC | Garbage Collector |
|---|---|---|
| Kapan bekerja | Compile time (kode disisipkan) | Runtime (proses terpisah) |
| Determinisme | Deterministik — `deinit` jalan tepat saat refcount 0 | Tidak — objek dibersihkan "nanti" |
| Overhead | Tersebar merata (retain/release traffic) | Pause saat collection |
| Siklus | **Tidak terdeteksi** — bocor | Terdeteksi (mark & sweep) |

Trade-off intinya: ARC memberi determinisme dan latensi yang dapat diprediksi
(penting untuk UI 120 Hz), dengan harga programmer harus memutus siklus sendiri.
</details>

---

## 🟢 2. Kapan pakai `weak`, kapan `unowned`?

<details><summary>Jawaban model</summary>

- **`weak`** kalau target bisa mati lebih dulu, atau lifetime-nya tidak pasti.
  Otomatis jadi `nil`, jadi tipenya selalu `Optional`.
- **`unowned`** kalau target dijamin hidup selama pemegangnya hidup.
  Bukan optional, dan crash terdefinisi kalau asumsinya salah.

Alasan memilih `unowned` bukan performa, tapi **dokumentasi invariant**:
kalau suatu hari asumsinya rusak, kamu ingin crash keras dan segera —
bukan `nil` diam-diam yang menimbulkan bug lain tiga layar kemudian.
</details>

**Follow-up:** *"Apa yang terjadi kalau kamu mengakses `unowned` setelah objeknya mati?"*
→ crash terdefinisi (*"attempted to read an unowned reference but object was already
deallocated"*), bukan use-after-free acak — karena alokasinya belum dibebaskan selama
unowned refcount masih > 0.

---

## 🟡 3. Bagaimana `weak` diimplementasikan?

<details><summary>Jawaban model</summary>

Objek Swift punya **tiga** refcount: strong, unowned, dan weak.

Normalnya refcount disimpan **inline** di header objek. Begitu satu `weak` reference
dibuat, objek mendapat **side table** — alokasi terpisah yang menyimpan pointer ke
objek + ketiga refcount. Weak variable menunjuk ke **side table**, bukan ke objeknya.

Karena itu weak bisa jadi `nil` dengan aman: side table tetap hidup meski objeknya
sudah dibebaskan, dan pembacaan weak menemukan flag "sudah mati" lalu mengembalikan `nil`.

Mendapat side table adalah operasi **satu arah** — objek yang punya side table tidak
pernah kehilangannya (untuk menghindari race).

Urutan matinya:
- strong → 0: `deinit` jalan
- unowned → 0: memori objek dibebaskan
- weak → 0: side table dibebaskan
</details>

**Follow-up:** *"Jadi `weak` lebih mahal?"* → ya: satu alokasi permanen begitu weak
pertama dibuat, plus setiap pembacaan harus lewat side table dengan operasi atomik.
`unowned` cuma dereference biasa. Untuk hot path yang dipanggil ribuan kali per detik,
itu terukur.

📖 [ARC & Retain Cycle](../articles/swift-language/03-arc-dan-retain-cycle.md)

---

## 🟡 4. Temukan retain cycle-nya

```swift
final class ImageDownloader {
    var completion: ((UIImage) -> Void)?
    func download(url: URL) { ... }
}

final class ProfileViewController: UIViewController {
    let downloader = ImageDownloader()

    override func viewDidLoad() {
        super.viewDidLoad()
        downloader.completion = { image in
            self.imageView.image = image
        }
        downloader.download(url: url)
    }
}
```

<details><summary>Jawaban model</summary>

Siklus: `ProfileViewController` → `downloader` (strong property) → `completion`
(strong closure) → `self`.

View controller tidak akan pernah di-`deinit` setelah di-pop, dan setiap kali user
membuka layar ini satu instance bocor beserta seluruh view hierarchy-nya.

Perbaikan:

```swift
downloader.completion = { [weak self] image in
    guard let self else { return }
    self.imageView.image = image
}
```

`guard let self` mengambil satu strong reference untuk seluruh durasi closure, jadi
`self` tidak bisa mati di tengah eksekusi — tapi karena strong-nya temporer dan tidak
disimpan, siklusnya tidak kembali.
</details>

**Follow-up:** *"Kalau `downloader` bukan property tapi variabel lokal, apakah masih siklus?"*
→ Tidak ada siklus (tidak ada jalur balik dari self ke downloader), tapi `self` tetap
ditahan hidup sampai download selesai. Kadang itu justru yang kamu mau.

---

## 🟡 5. Sebutkan sumber leak yang bukan closure

<details><summary>Jawaban model</summary>

1. **`Timer` target-action** — `Timer` menahan `target` dengan strong, dan run loop
   menahan `Timer`. `[weak self]` tidak berlaku. Wajib `invalidate()`.
2. **`NotificationCenter.addObserver(forName:queue:using:)`** — varian closure menahan
   closure sampai observer dihapus. Simpan token-nya.
3. **`CADisplayLink`** — sama seperti `Timer`.
4. **Delegate tanpa `weak`** — dan protokolnya harus `: AnyObject` supaya `weak` bisa dipakai.
5. **`URLSession` yang tidak di-`invalidateAndCancel()`** — session custom menahan
   delegate-nya dengan **strong**, ini pengecualian dari aturan delegate biasa.
6. **Child view controller yang tidak dilepas** dengan `removeFromParent()`.
7. **Abandoned memory** — cache/array yang terus tumbuh. Bukan siklus, tapi tetap
   memori yang tidak akan pernah dipakai lagi.
</details>

**Follow-up:** *"Nomor 7 tidak akan terdeteksi Instruments → Leaks. Bagaimana menemukannya?"*
→ Allocations dengan Mark Generation: buka layar, tutup, mark, ulangi 5×. Kalau
"Persistent" naik terus, ada yang menumpuk.

---

## 🟡 6. Bagaimana kamu membuktikan ada leak?

<details><summary>Jawaban model</summary>

Berurutan, dari yang paling murah:

1. **`deinit { print(...) }`** di setiap view controller dan view model. 30 detik,
   menangkap 80% kasus.
2. **Memory Graph Debugger** (Xcode debug bar). Jalankan, pop layar, tekan tombolnya.
   Objek yang seharusnya mati akan terlihat, dan kamu bisa klik untuk melihat
   **siapa yang menahannya**. Badge ungu menandai siklus yang terdeteksi runtime.
3. **Instruments → Leaks** untuk leak yang tidak terlihat di graph.
4. **Instruments → Allocations + Mark Generation** untuk abandoned memory.

Poin pentingnya: mulai dari yang murah, dan **buktikan** sebelum memperbaiki.
Menaburkan `[weak self]` tanpa mengukur adalah cara paling umum membuat kode
lebih sulit dibaca tanpa memperbaiki apa pun.
</details>

---

## 🔴 7. Apa yang terjadi di sini?

```swift
final class A {
    var b: B?
    deinit { print("A deinit") }
}
final class B {
    unowned var a: A
    init(a: A) { self.a = a }
    deinit { print("B deinit") }
}

var a: A? = A()
a?.b = B(a: a!)
a = nil
```

<details><summary>Jawaban model</summary>

Mencetak `A deinit` lalu `B deinit`.

`a = nil` → strong RC `A` jadi 0 (karena `B.a` unowned, tidak menghitung) →
`A.deinit` jalan → `A.b` dilepas → strong RC `B` jadi 0 → `B.deinit`.

Tidak ada leak. Tapi ini desain yang rapuh: kalau seseorang menyimpan `B` di tempat
lain sehingga `B` hidup lebih lama dari `A`, akses `b.a` akan crash. `unowned` di sini
menyatakan "B tidak akan pernah hidup tanpa A" — pastikan itu benar-benar invariant
yang dijaga, bukan kebetulan.
</details>

---

## 🔴 8. Kenapa `[weak self]` bisa menyebabkan bug, bukan memperbaikinya?

<details><summary>Jawaban model</summary>

Karena `[weak self]` mengubah pertanyaan "apakah ini leak" jadi
"apakah pekerjaan ini boleh tidak selesai".

```swift
func saveDraft() {
    Task { [weak self] in
        guard let self else { return }        // ⚠️ kalau VC sudah di-pop, draft HILANG
        await self.repository.save(self.draft)
    }
}
```

Kalau user menutup layar sebelum penyimpanan selesai, `self` sudah `nil` dan
draft-nya tidak pernah disimpan — diam-diam.

Untuk pekerjaan yang **harus** selesai terlepas dari nasib UI, jangan capture `self`
sama sekali; capture nilai yang dibutuhkan:

```swift
func saveDraft() {
    let repository = self.repository       // ambil apa yang perlu
    let draft = self.draft
    Task { await repository.save(draft) }  // tidak bergantung self sama sekali
}
```

Ini pola yang benar: pisahkan "pekerjaan yang memperbarui UI" (boleh `weak`)
dari "pekerjaan yang harus selesai" (jangan bergantung pada lifetime UI).
</details>

---

## 🔴 9. Apa itu retain/release traffic, dan kapan ia jadi masalah performa?

<details><summary>Jawaban model</summary>

Setiap kali reference type dioper, disimpan, atau di-capture, compiler menyisipkan
`swift_retain`/`swift_release` — dan keduanya adalah operasi **atomik**, yang jauh
lebih mahal dari increment biasa karena melibatkan cache coherency antar core.

Ia jadi masalah di:
- Loop ketat yang mengoper objek class
- `[any Protocol]` besar yang di-iterate (setiap akses existential punya value witness)
- Bridging Swift↔Objective-C yang sering (setiap `NSString`/`NSArray` bridge punya biaya)

Cara mengurangi:
- Pakai value type untuk data yang mengalir di hot path
- `final` pada class agar dispatch jadi statis dan optimizer bisa menghilangkan
  retain/release yang bisa dibuktikan tidak perlu
- Whole Module Optimization agar optimizer melihat lintas file

Cara mengukurnya: Instruments → Time Profiler, cari `swift_retain`/`swift_release`
di call tree. Kalau keduanya masuk 10 besar, itu sinyalnya.
</details>

---

## 🔴 10. `autoreleasepool` — masih relevan di Swift?

<details><summary>Jawaban model</summary>

Ya, di satu situasi spesifik: loop panjang yang membuat banyak objek **Objective-C**
(atau memicu bridging), di mana objek autorelease baru dilepas saat pool di-drain —
yaitu di akhir iterasi run loop, bukan di akhir iterasi loop-mu.

```swift
for url in thousandsOfImageURLs {
    autoreleasepool {
        let image = UIImage(contentsOfFile: url.path)   // Obj-C, autoreleased
        process(image)
    }                                                    // pool drained tiap iterasi
}
```

Tanpa pool, memori naik terus sampai loop selesai — dan di perangkat itu berarti
jetsam (app dibunuh sistem).

Untuk kode Swift murni tanpa bridging, ARC melepas objek segera saat refcount 0,
jadi `autoreleasepool` tidak diperlukan.
</details>

---

## Latihan cepat

1. Objek punya berapa refcount, dan apa yang terjadi saat masing-masing mencapai nol?
2. Kenapa `weak var delegate` butuh protokol `: AnyObject`?
3. `URLSession(configuration:delegate:delegateQueue:)` — apa yang tidak biasa dari
   kepemilikan delegate-nya?
4. Apa bedanya leak dan abandoned memory, dan alat mana untuk masing-masing?
5. Kenapa `guard let self else { return }` lebih baik dari `self?.foo()` berulang kali?

<details><summary>Kunci</summary>

1. Tiga. strong→0: `deinit`. unowned→0: memori dibebaskan. weak→0: side table dibebaskan.
2. Karena `weak` butuh reference counting, dan protokol tanpa constraint kelas bisa
   diadopsi value type.
3. Ia menahan delegate-nya dengan **strong** (kebalikan dari konvensi delegate biasa),
   sampai kamu memanggil `invalidateAndCancel()` atau `finishTasksAndInvalidate()`.
4. Leak = tidak ada lagi yang menunjuk, memori tak terjangkau (Instruments → Leaks).
   Abandoned = masih terjangkau tapi tidak akan pernah dipakai (Allocations + Mark Generation).
5. `self?` mengecek ulang di setiap pemakaian — `self` bisa hidup di baris 1 dan mati
   di baris 3, membuat closure berjalan setengah. `guard let self` mengambil satu strong
   temporer untuk seluruh durasi closure.
</details>
