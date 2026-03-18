# Flutter x Unity 整合修復報告

**專案**：`flutter_unity_demo`  
**Unity 版本**：2022.3.62f2  
**Flutter 版本**：3.x（含 Flutter 3.24+）  
**插件**：`flutter_unity_widget`  
**日期**：2026-03-18

---

## 問題總覽

| # | 錯誤 | 根本原因 |
|---|------|---------|
| 1 | `Inconsistent JVM Target (1.8 vs 17)` | Kotlin 與 Java 編譯目標版本不一致 |
| 2 | `sourceCompatibility has been finalized` | 嘗試在 Android Extension 已鎖定後修改編譯設定 |
| 3 | `--release option is not supported` | `options.release.set(17)` 與 Android Gradle Plugin 不相容 |
| 4 | `Cannot run afterEvaluate when project is already evaluated` | 在已評估的子專案上呼叫 `afterEvaluate` |
| 5 | `'lifecycle' hides member of supertype 'LifecycleOwner'` | 插件 2022.2.1 與 Flutter 3.24+ 的 AndroidX Lifecycle 介面衝突 |
| 6 | `Platform declaration clash: getLifecycle()` | patch 後出現 JVM 方法簽名重複衝突 |
| 7 | `NDK is not installed` | Unity IL2CPP 找不到 Android NDK |
| 8 | `android.ndkVersion ... refers to a different version` | `ndkPath` 與 `ndkVersion` 版本號不一致 |

---

## 修復內容

### 1. `android/build.gradle.kts`（根 Gradle 設定）

**問題**：嘗試多種方式強制 JVM 17，反覆遇到「已鎖定」或「評估順序」錯誤。

**最終解法**：在 `subprojects` 區塊中，使用懶加載的 Task 設定（不觸碰 Android Extension）：

```kotlin
subprojects {
    // Force JVM 17 for all Java compilation tasks
    tasks.withType<JavaCompile>().configureEach {
        sourceCompatibility = "17"
        targetCompatibility = "17"
    }
    // Force JVM 17 for all Kotlin compilation tasks
    tasks.withType<KotlinCompile>().configureEach {
        compilerOptions {
            jvmTarget.set(JvmTarget.JVM_17)
        }
    }
}
```

---

### 2. `android/gradle.properties`

**問題**：Kotlin 的 JVM 目標驗證在某些路徑上仍觸發錯誤。

**修復**：加入以下設定，將驗證模式從 error 降為 warning：

```properties
kotlin.jvm.target.validation.mode=warning
```

---

### 3. `android/unityLibrary/build.gradle`

**問題 A**：`compileOptions` 未設定，Java 編譯目標仍為 1.8。

**修復**：明確設定為 JVM 17：
```groovy
compileOptions {
    sourceCompatibility JavaVersion.VERSION_17
    targetCompatibility JavaVersion.VERSION_17
}
```

**問題 B**：NDK 路徑被注釋掉 → NDK 找不到。

**問題 C**：`ndkPath` 與 `ndkVersion` 版本號不一致。

**修復**：解注釋 `ndkPath` 並加入對應的 `ndkVersion`：
```groovy
ndkPath "D:/UnityIDLE/2022.3.62f2/Editor/Data/PlaybackEngines/AndroidPlayer/NDK"
ndkVersion "23.1.7779620"
```

---

### 4. `android/app/src/main/AndroidManifest.xml`

**問題**：手動加入的 `OverrideUnityActivity` 宣告重複，導致 Manifest Merger 失敗。

**修復**：移除重複的 `<activity>` 宣告，只保留一個。

---

### 5. `pubspec.yaml`（最關鍵的修復）

**問題**：`flutter_unity_widget: ^2022.2.1`（pub.dev 版本）與 Flutter 3.24+ 的 AndroidX Lifecycle 介面不相容，且此版本已不再更新。

**修復**：改用 GitHub master branch 上的最新版本（2022.3.0），已包含 Flutter 3.24+ 修復：

```yaml
flutter_unity_widget:
  git:
    url: https://github.com/juicycleff/flutter-unity-view-widget.git
    ref: master
```

---

## Flutter ↔ Unity 通訊架構

| Flutter 端 | Unity 端 |
|-----------|---------|
| `_unityWidgetController.postMessage('Cube', 'SetRotationSpeed', value)` | `Rotate.cs` 中的 `SetRotationSpeed(string message)` 方法 |

> ⚠️ Unity 場景中掛載 `Rotate.cs` 的物件名稱**必須是 `Cube`**，否則訊息無法送達。

---

## 注意事項

- **Pub cache 的手動 patch** 已失效（因為改用 git 版本），不需要維護。
- 每次執行 `flutter clean` 後，`pub get` 會重新從 git 拉取最新 master，patch 會自動帶入。
- 若之後 `NDK` 路徑因 Unity 更新而改變，請同步更新 `unityLibrary/build.gradle` 中的 `ndkPath` 和 `ndkVersion`。
