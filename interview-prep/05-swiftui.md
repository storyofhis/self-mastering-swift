---
title: "Interview Prep 05 — SwiftUI"
category: "Interview Prep"
status: draft
tags:
  - swift-mastering
  - interview-prep
  - interview
---

# Interview Prep 05 — SwiftUI

> **Status:** 🚧 kerangka — pertanyaan sudah terdaftar, jawaban model belum ditulis penuh

## Daftar pertanyaan yang harus disiapkan

### 🟢 Dasar
1. Kenapa `View` adalah `struct`, bukan `class`?
2. Apa bedanya `@State`, `@Binding`, `@StateObject`, `@ObservedObject`, `@EnvironmentObject`?
3. Kapan `body` dipanggil ulang?
4. Apa fungsi `.id()`?

### 🟡 Menengah
5. Kenapa `@ObservedObject` di tempat yang seharusnya `@StateObject` menyebabkan bug?
   *(objek dibuat ulang setiap evaluasi body parent)*
6. Kenapa `if isEditing { TextField() } else { TextField() }` kehilangan fokus dan state?
   *(structural identity berubah → `_ConditionalContent` dua cabang berbeda)*
7. Apa bedanya `@Observable` dan `ObservableObject`, dan kenapa yang pertama
   mengurangi re-render? *(granularitas per-property yang dibaca)*
8. Kapan pakai `task(id:)` dan bukan `onAppear`?
9. Bagaimana `ViewBuilder` menangani `if`/`for`? *(`buildOptional`/`buildEither`/`buildArray`)*

### 🔴 Lanjut
10. Bagaimana SwiftUI memutuskan view mana yang perlu di-render ulang?
    *(dependency graph dari property yang benar-benar dibaca; `withObservationTracking`)*
11. Kapan `EquatableView` / `.equatable()` berguna, dan kapan justru merugikan?
12. Bagaimana kamu men-debug view yang di-render ulang terlalu sering?
    *(`Self._printChanges()`, Instruments SwiftUI template)*
13. Bagaimana memigrasikan layar UIKit ke SwiftUI secara bertahap tanpa menulis
    ulang navigasi? *(`UIHostingController` per layar daun)*
14. Kenapa `List` bisa lebih cepat dari `ScrollView` + `LazyVStack` untuk data besar?

### Soal desain
15. Desain layar feed dengan pull-to-refresh, pagination, dan error state di SwiftUI.
16. Bagaimana kamu menangani form kompleks dengan validasi lintas field?

## Yang harus ditulis
- Jawaban model tiga lapis (definisi → mekanisme → trade-off)
- Follow-up untuk setiap nomor
- Referensi silang ke [artikel SwiftUI](../articles/swiftui/)
