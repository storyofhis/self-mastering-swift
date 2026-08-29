---
title: "App & Scene Lifecycle: `UISceneDelegate` dan State Restoration"
category: "iOS & UIKit"
status: draft
tags:
  - swift-mastering
  - ios/uikit
  - article
---

# App & Scene Lifecycle: `UISceneDelegate` dan State Restoration

> **Kategori:** iOS & UIKit · **Level:** Menengah · **Status:** 🚧 kerangka

## Kerangka isi

### 1. Kenapa `UISceneDelegate` ada
- Multi-window di iPad; satu app process, banyak scene
- Pembagian tanggung jawab: `UIApplicationDelegate` = process, `UISceneDelegate` = UI

### 2. Urutan lengkap
```
application(_:didFinishLaunchingWithOptions:)
scene(_:willConnectTo:options:)          ← bangun window & root view controller
sceneWillEnterForeground(_:)
sceneDidBecomeActive(_:)
sceneWillResignActive(_:)                ← simpan draft, jeda animasi/game
sceneDidEnterBackground(_:)              ← simpan state, lepas resource mahal
sceneDidDisconnect(_:)
```
- Contoh dari project `movie`: `SceneDelegate` membangun `UITabBarController` secara
  programmatic (tanpa storyboard)

### 3. Background execution
- `beginBackgroundTask` untuk menyelesaikan pekerjaan pendek
- `BGAppRefreshTask` vs `BGProcessingTask` (BackgroundTasks framework)
- `URLSessionConfiguration.background` untuk unduhan yang bertahan setelah app ditutup
- Batas nyata: waktu, memori, dan bahwa sistem yang memutuskan kapan

### 4. State restoration
- `NSUserActivity` sebagai mekanisme modern
- `stateRestorationActivity(for:)` dan `scene(_:restoreInteractionStateWith:)`
- Kapan ini benar-benar perlu (app yang di-*jetsam* saat multitasking)

### 5. Deep link & universal link
- `scene(_:openURLContexts:)`, `scene(_:continue:)`
- Kenapa routing harus punya satu tempat (Coordinator/Router), bukan tersebar

## Cek pemahaman (draft)
1. Kenapa `application(_:didFinishLaunching:)` tidak lagi tempat membuat window?
2. Apa bedanya `sceneDidDisconnect` dan `sceneDidEnterBackground`?
3. Berapa lama app punya waktu setelah `sceneDidEnterBackground`?

## Sumber untuk ditulis
- Project `movie`: `SceneDelegate.swift`, `AppDelegate.swift`
- [Apple — Managing your app's life cycle](https://developer.apple.com/documentation/uikit/app_and_environment/managing_your_app_s_life_cycle)
