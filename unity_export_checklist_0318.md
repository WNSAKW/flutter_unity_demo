# Unity 重新 Export 後操作 Checklist

每次從 Unity 重新輸出 Android Project 後，依序完成以下步驟。

---

## Step 1：覆蓋 unityLibrary 資料夾

將 Unity 輸出的 `unityLibrary` 資料夾覆蓋到：
```
flutter_unity_demo/android/unityLibrary/
```

---

## Step 2：修改 [android/unityLibrary/build.gradle](file:///d:/awz398/aifree/flutter_unity/flutter_unity_demo/android/unityLibrary/build.gradle)

找到 `android { ... }` 區塊，確認以下設定：

```groovy
android {
    namespace "com.unity3d.player"
    ndkPath "D:/UnityIDLE/2022.3.62f2/Editor/Data/PlaybackEngines/AndroidPlayer/NDK"
    ndkVersion "23.1.7779620"
    compileSdkVersion 36
    buildToolsVersion '34.0.0'

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_11
        targetCompatibility JavaVersion.VERSION_11
    }

    defaultConfig {
        minSdkVersion 22
        targetSdkVersion 36
        ndk {
            abiFilters 'armeabi-v7a', 'arm64-v8a'
        }
        versionCode 1
        versionName '0.1'
        consumerProguardFiles 'proguard-unity.txt'
    }

    lintOptions {
        abortOnError false
    }

    aaptOptions {
        noCompress = ['.unity3d', '.ress', '.resource', '.obb', '.bundle', '.unityexp']
        ignoreAssetsPattern = "!.svn:!.git:!.ds_store:!*.scc:!CVS:!thumbs.db:!picasa.ini:!*~"
    }

    packagingOptions {
        doNotStrip '*/armeabi-v7a/*.so'
        doNotStrip '*/arm64-v8a/*.so'
        jniLibs {
            useLegacyPackaging true
        }
    }
}
```

> [!IMPORTANT]
> 1. Unity 預設輸出通常是 Java 8（`VERSION_1_8`），**必須手動改成 Java 11**，否則會有 JVM target 不相容的 build error。
> 2. `ndkPath` 常在匯出時被 Unity 註解掉，**必須取消註解或加回去**（包含 `ndkVersion "23.1.7779620"`），否則會拋出 `NDK is not installed` 錯誤。

---

## Step 3：確認 `android/settings.gradle` 路徑正確

確保 `unityLibrary` 的引用存在且路徑正確：

```groovy
include ':unityLibrary'
project(':unityLibrary').projectDir = file('unityLibrary')
```

---

## Step 4：不需要更動的檔案

以下檔案**不需要**因為重新 export 而修改：

| 檔案 | 原因 |
|---|---|
| `pubspec.yaml` | Flutter 套件設定，與 Unity 無關 |
| [android/build.gradle.kts](file:///d:/awz398/aifree/flutter_unity/flutter_unity_demo/android/build.gradle.kts) | Flutter 主專案設定，不受 Unity export 影響 |
| [android/gradle.properties](file:///d:/awz398/aifree/flutter_unity/flutter_unity_demo/android/gradle.properties) | 全域 Gradle 設定，不受 Unity export 影響 |
| `lib/` (Flutter Dart 程式碼) | Flutter UI 層，不受 Unity export 影響 |

---

## Step 5：重新建置確認

```bash
flutter clean
flutter pub get
flutter run
```

如果出現 build error，優先檢查 Step 2 的 [build.gradle](file:///d:/awz398/aifree/flutter_unity/flutter_unity_demo/android/unityLibrary/build.gradle) 設定是否正確套用。
