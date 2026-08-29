---
title: "Error Handling: `throws`, `Result`, dan `typed throws`"
category: "Swift Language & Internals"
status: draft
tags:
  - swift-mastering
  - swift/language
  - article
---

# Error Handling: `throws`, `Result`, dan `typed throws`

> **Kategori:** Swift Language & Internals · **Level:** Menengah · **Status:** 🚧 kerangka

## Kenapa topik ini penting
Error handling adalah tempat di mana desain API paling terlihat. Interviewer memakainya
untuk menguji apakah kamu memikirkan jalur gagal seserius jalur sukses.

## Kerangka isi

### 1. Model error Swift
- `throws` bukan exception: tidak ada stack unwinding, biayanya mendekati nol pada jalur sukses
- Error dikembalikan lewat register khusus (`swifterror`) — jelaskan kenapa ini murah
- `Error` protokol kosong; `LocalizedError` untuk pesan yang ditampilkan ke user

### 2. `throws` vs `Result` vs Optional
| Bentuk | Pakai kalau |
|---|---|
| `throws` | Kegagalan punya alasan yang perlu diketahui pemanggil |
| `Result<T, E>` | Error perlu disimpan/dioper sebagai nilai (mis. ke closure lama) |
| Optional | Ketiadaan nilai adalah hasil yang wajar, bukan kegagalan |
- Sejak async/await, `Result` jauh lebih jarang dibutuhkan

### 3. `typed throws` (SE-0413, Swift 6)
- `func f() throws(NetworkError)` — error type diketahui compiler
- Kapan berguna: library boundary, embedded Swift, exhaustive `catch`
- Kapan **tidak**: API aplikasi yang error-nya akan berkembang → `any Error` lebih fleksibel
- Hubungannya dengan `rethrows` dan `throws(Never)`

### 4. Memetakan error antar lapisan
- Contoh dari project `movie`: `DecodingError` → `MusicServiceError.decodingFailed` → pesan UI
- Kritik: `decodingFailed` tanpa payload membuang informasi diagnosis
- Bentuk yang lebih baik: `case decoding(underlying: Error)` + `os.Logger`

### 5. `defer`, dan Swift 6.4 yang mengizinkan `await` di dalamnya
### 6. Kesalahan umum
- `try?` yang menelan error diam-diam
- `catch` kosong
- Melempar error teknis langsung ke UI
- Tidak membedakan error yang bisa di-retry dari yang tidak

## Cek pemahaman (draft)
1. Kenapa `throws` di Swift lebih murah dari exception di C++?
2. Kapan `typed throws` justru merugikan?
3. Bagaimana kamu mendesain error type untuk networking layer yang mendukung retry?

## Sumber untuk ditulis
- `swift/docs/ErrorHandling.md`, `swift/docs/ErrorHandlingRationale.md`
- [SE-0413 — Typed throws](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0413-typed-throws.md)
- Project `movie`: `MusicServiceError`, `iTunesService.fetch`
