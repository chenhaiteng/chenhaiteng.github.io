---
layout: post
title:  "透過 JitPack 發佈 Android 函式庫"
date:   2025-01-06 23:00:00 +0800
categories: android
tags: kotlin android gradle jitpack maven-publish
---

## 發佈函式庫的優點
- 分享成果
    - 程式複用
    - 團隊合作
- 簡化/優化程式架構
    - 抽象化/介面化/模組化。
    - 降低依賴複雜性。
    - 拆分測試範圍，便於測試。
- 增進開發效能
    - 在開發產品的不同階段，減少其執行量
        - 開發(Implementating/Coding): 使用可複用函式庫，減少開發時間
        - 建構(Build): 不需重新建構函式庫
        - 測試: 不需測試已發佈之函式庫

---

## Android 如何發佈函式庫
- 理論上，只要有一個相容於 Apache Maven/Ant 標準的倉儲，即可將函式庫發布於其上。
    - 在目前的 Android 專案中，可透過 maven/ivy API或DSL，來指定需要的倉儲位置。
- 常用的 Maven 倉儲:
    - Maven Central
        - Android 預設倉儲之一
        - 目前由 Sonatype 管理
        - 需要註冊(或用GitHub/Google帳號登入)
        - 發佈新函式庫前，需要開票申請
    - JitPack
        - 非 Android 預設倉儲，使用者需要自行在專案中設定
        - 使用GitHub帳號登入，註冊
        - 直接與GitHub連動，發佈時不需另外開票申請
        - 也可以另行設定其他Git Host
    - 其他選擇:
        - Azure
        - AWS
- 選擇建議:
    - 大型團隊/公司: Maven Central
        - 有人力去處理申請開票過程
        - 函式庫使用者/團隊只要處理相依設定，不需設定額外倉儲
    - 小型團隊/獨立開發者: JitPack
        - 省事，只需要設定專案與Git Host
        - 對使用者而言多一道設定步驟

---

## 函式庫創建、發佈流程:
### 開發環境:
    - Macbook Air M2/macOS 15.2
    - Android Studio Ladybug 2024.2.1 Patch 3
    - Gradle Plugin 8.7.3
    - Gradle 8.9
    - Git Host: GitHub
    - Git Client: Fork

### 創建函式庫專案:
1. 登入GitHub帳號，創建一個新的公開倉儲
    - .gitignore 選擇 Android
    - 版權部分依照自己的需求選擇
    - 也可選擇加入Read Me
2. 創建完後，透過Git Client 或命令列建立本地端的倉儲.
    - `git clone git@github.com:${account}/${repository name}.git`
    - 後續每一步驟完成，確認無誤後，可以先提交(commit)到本地倉儲中，以避免意外改動。
3. 打開 Android Studio，建立函式庫專案
    - Android Studio的專案範本中，並沒有函式庫/模組的選項，建議依照需求，選求對應的專案即可。
    - 新專案預設的App，可以用在後續的驗證中。
    - 專案存放的路徑，指定為步驟 2 中，本地端的存放路徑。
    - 建立好後，檢視一下檔案結構，確保 GitHub 所生成的 .gitignore，LICENSE，README.md 與 Android Studio 生成的 'app' 及其他檔案，位於同一層級:
        - Repository Root
            - .gitignore
            - `app/`
            - ...其他檔案及路徑
            - LICENSE
            - README.md
4. 在剛才產生的 Android Studio 專案中，新增模組如下:
    1. File -> New -> New Module
    ![Fig 1](/assets/images/android_studio_new_module_menu.png)
    2. 設定模組
    ![Fig 2](/assets/images/android_studio_new_module_dialog.png)
    3. 模組建立完成後，應該會在專案根目錄下看到多一個目錄 `${module_name}/`
    4. 在Android Studio 的專案瀏覽中，切換到Android模式，也會看到多一個 `${module_name}` 區塊，同時，`Gradle Scripts` 區塊下，會多一個模組專屬的 `build.gradle.kts(Module: ${module_name})` 檔案。

5. 在專案中設定發佈資訊
    1. 在 Android Studio中，打開 `build.gradle.kts(Module: ${module_name})`
    2. 在 plugin 區塊，加入 `maven-publish` 組件如下:
        ```gradle
        plugins {
            alias(libs.plugins.android.library)
            alias(libs.plugins.kotlin.android)
            alias(libs.plugins.kotlin.compose)
            id("maven-publish")
        }
        ```
    3. 在 `build.gradle.kts(Module: ${module_name})` 中，`android` 區塊的最後面，加上 `publishing` 區塊，描述發佈資訊如下:
        ```gradle
        publishing {
            publications {
                register<MavenPublication>("release") {
                    groupId = "com.github.${github_account}"
                    artifactId = "${name_of_artifact}"
                    version = "0.0.1"
                    afterEvaluate {
                        from(components["release"])
                    }
                }
            }
        }
        ```
        groupId 和 artifactId 是用來識別函式庫的[<sup>(註1)</sup>](#ref_1)，需要保持獨一無二，以便與他人的成果作出區別。
        通常 groupId 會採用反向域名表示所屬組織/個人，artifactId 則與專案名相同。本例中，因為使用GitHub做為Git host，所以直接用GitHub網域當做範例。
    4. 設定使用函式庫時的限制(非必要)
        ```gradle
        android {
            defaultConfig {
                aarMetadata {
                    minCompileSdk = 29
                }
                minSdk = 29

                testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
                consumerProguardFiles("consumer-rules.pro")
            }
        }
        ```
        透過在 `aarMetatdata`[<sup>(註2)</sup>](#ref_2) 區塊中填加限制，可以在其他人取用此函式庫時，要求其符合條件。以上例而言，若取用函式庫的專案，使用的sdk版本小於29，Gradle會發出警告要求專案昇級。

    完成上述設定步驟後，並將成果推到GitHub倉儲後，發佈相關設定就完成了。理論上，此時的專案，應可以透過 Apach Maven 相容的服務發佈。

    但實務上，因為開發環境與發佈環境不同，還會有一些額外工作要處理。

6. 整合環境處理
    1. JDK 相容性[<sup>(註5)</sup>](#ref_5):
        - Android Studio Ladybug 產生的 `build.gradle.kts`，預設 JDK 版本為 11
        - 目前預設 Android SDK 是 34，最低需求 JDK 版本為 17
        - JitPack 環境的預設 JDK 為 8[<sup>(註6)</sup>](#ref_6)
        - 如果要使用函式庫的專案有使用CI/CD服務，或是有另外的建構環境，也需要一起考慮。
        - 為整合JDK環境，需要做以下幾件事:
            1. 調整 `build.gradle.kts(Module: ${module_name})` 以及 `build.gradle.kts(Module: app)`：
            在 `android` 區塊中，找到 `compileOptions` 與 `kotlinOptions`，改寫內容如下:
                ```gradle
                android {
                    compileOptions {
                        sourceCompatibility = JavaVersion.VERSION_17
                        targetCompatibility = JavaVersion.VERSION_17
                    }
                    kotlinOptions {
                        jvmTarget = "17"
                    }
                }
                ```
            2. 在專案的根目錄下，加入 `jitpack.yml`[<sup>(註7)</sup>](#ref_7) 檔案如下:
                ```yml
                jdk:
                    - openjdk17
                ```
            基本上，JitPack會依照設定，自動切換環境，如果要做更細緻的設定(例如使用 17.0.1 而非最新版本 17.0.12)，可以參考官方文件[<sup>(註7)</sup>](#ref_7)再進行設定。
            3. 設定主專案的CI/CD環境(非必要)，以CircleCI為例：
            如果主專案與Circle CI做過整合，在 `專案根目錄/.circleci/` 下會有 `config.yml` 描述檔。
            在檔案開頭會有一段指令如下，告訴Circle CI要用那一種硬體來建構程式:
                ```yml
                orbs:
                    android: circleci/android@3.0.2
                ```
            目前，Android Orb 的 3.0.0+ 版本[<sup>(註9)</sup>](#ref_9)，預設JDK為 OpenJDK 17，以這次的例子來說，不需要改動。
            但如果想要使用更新版本的JDK來建構程式，可以在 `config.yml` 裡面，每一個job裡面，加上以下步驟:
                ```yml
                jobs:
                    ${job_name}:
                        executor:
                        // 設定執行者
                        steps:
                            - checkout
                            - android/change_java_version:
                                java_version: 23
                            - // 後續步驟
                ```
            如果選擇的Android Orb比較舊，則預設的JDK版本為 OpenJDK 11，此時則需要改寫如下:
                ```yml
                jobs:
                    ${job_name}:
                        executor:
                        // 設定執行者
                        steps:
                            - checkout
                            // 注意！舊版本使用折線連接字符
                            - android/change-java-version:
                                java-version: 17
                            - // 後續步驟
                ```
    2. Android Jetpack 與 Android SDK相容性:
        - Android Studio Ladbybug 預設 SDK 34
        - 引入 Android Jetpack 1.15.0，最低需求 SDK 35[<sup>(註8)</sup>](#ref_8)
        處理的方式有兩個方向:
        1. 降級Jetpack，在 `build.gradle.kts(Module: ${module_name})` 加入以下區塊:
                ```gradle
                configurations.all {
                    resolutionStrategy {
                        force "androidx.core:core:1.13.1" //降回前一穩定版本
                    }
                }
                ```
        2. 昇級SDK到35
                ```gradle
                android {
                    compileSdk = 35
                    defaultConfig {
                        // Other settings
                        targetSdk = 35 // 非必要，但最好與compileSdk一致.
                    }
                }
            ```

### 測試發佈函式庫
在創建函式庫專案的設定工作完成後，就可以實際發布函式庫了。
不過因為函式庫尚未實作，以下的步驟只是要確認發佈流程沒有問題，之後才可以專注在函式庫的開發上。

1. 將設定好的專案推送(push)到 GitHub 服務上。
2. 用 GitHub 帳號，登入 [JitPack](https://jitpack.io) 網站如下:

   ![Fig 3](/assets/images/jitpack_homepage.png)

   左側，會顯示登入帳號在 GitHub 上所有的公開倉儲，不論是否為 Java/Kotlin/Android 函式庫。

   中間的輸入欄位，可以用來搜尋所有在 JitPack 上有紀錄的倉儲。搜尋到後，會顯示如下畫面:

   ![Fig 4](/assets/images/jitpack_more_shapes_compose.png)

   因為目前專案還是初始狀態，所以只能看到提交紀錄，以及目前的狀態快照(SNAPSHOT)。

3. 點擊最後一筆提交紀錄的`Get it`按鈕，JitPack就會開始建構函式庫，過程大約5-10分鐘左右。完成後，會顯示如下狀態：

    ![Fig 5](/assets/images/jitpack_built_success.png)

    將滑鼠指標移至Log圖示上，會顯示出此次發佈函式是成功，還是失敗。點擊Log圖示，則可以看到詳細的紀錄。

    一但確定測試發佈成功，接下來就可以專注在函式庫本身的實作上。

### 在函式庫引入Jetpack Compose(非必要)
若想要在函式庫內，引入並使用 JetPack Compose 相關功能，設定的方式與一般應用程式相同[<sup>(註10)</sup>](#ref_10)，直接參照官方文件設定即可。
注意，官方的文件在依賴管理的部分寫法並不一致。大部分文件採取在`build.gradle.kts`中直接宣告，但部分文件教學會使用 Gradle版本目錄(Gradle Version Catelogs)[<sup>(註11)</sup>](#ref_11)的寫法，將宣告統一寫在 'libs.versions.toml' 檔案中，再由 `build.gradle.kts` 引用。基本上兩種寫法是等效的，不過以目前開發環境的發展來看，建議儘量使用 Gradle版本目錄，而不要直接宣告依賴。[<sup>(註12)</sup>](#ref_12)

### 撰寫範例程式(非必要)
完成函式庫功能後，在正式發佈前，最好能提供範例程式，用來說明函式庫功能，同時也兼做本地端測試，確保函式庫能被正確引入。

範例程式可以直接使用創建專案時產生的 app 模組。為了讓 app 模組能直接存取函式庫，在 'build.gradle.kts(Module: app)' 中，加入以下依賴宣告:
```gradle
dependencies {
    // 其他依賴宣告
    // ...
    implementation(project(":${module_name}"))
}
```

接著，檢查Java環境以及SDK版本，是否與函式庫設定相同。
同步後沒有問題，就可以直接開始撰寫範例程式，撰寫的過程中若發現問題，也可以隨時更正，修改函式庫。

### 正式發佈
函式庫開發告一段落後，便可以考慮透過 JitPack 正式發佈。
發佈的方式，大致上與[測試發佈函式庫](#測試發佈函式庫)相同。唯一差異點在於發佈前，要先在 GitHub 服務上發佈對應的版本。
1. 透過 GitHub 網頁發佈版本
    1. 登入GitHub，並進到對應的倉儲。
    2. 在頁面的右側，可以看到 Release 區塊:

        ![Fig 6](/assets/images/github_release_block.png)

        點擊 Create a new release，開始創建發佈版本。
    3. 如下圖所示，產生新的版本，必需對應到某個標簽(Tag)，如果尚未有對應的標簽，可以在輸入標簽文字後，點擊 `Create new tag` 字樣，產生新的標簽。

        ![Fig 7](/assets/images/github_create_release.png)
    4. 選好標簽後，依序填入版本標題(Title)，內容描述(Description)，將畫面拉到最下方，點擊 `Publish Release` 按鈕，即可完成 GitHub 上的版本發佈。
2. 透過 GitHub CLI[<sup>(註13)</sup>](#ref_13) 發佈版本
    對於想進一步將流程自動化的開發者，推薦使用 GitHub CLI 來幫助發佈版本。
    - 安裝指令: 
    ```zsh
    brew install gh
    ```
    另外也可透過 MacPorts, Conda, Spack, Webi等不同的套件管理工具安裝。 
    - 安裝完成後，需要先執行設定: 
    ```zsh
    gh auth login
    ```
    執行後，依照畫面上的互動式指令設定，給予 gh 執行權限操作GitHub倉儲。
    - 發佈 GitHub 版本: 
    ```zsh
    gh release create 0.0.1 --notes "My Release"
    ```
    GitHub CLI 還有其他更進階的功能，有興趣可以直接參閱官方文件[<sup>(註13)</sup>](#ref_13)。

在 GitHub 版本發佈完成後，登入JitPack網站，選擇函式庫，應該會看到 Release 分頁下，多出一個版本列表。同時，會看到 JitPack 自動觸發建構版本的工作。

### 在主程式中引用自建函式庫
在 JitPack 函式庫版本建構完成後，點擊對應版本的 `Get it` 按鈕，或是將畫面拉到最下方，就可以看到如何在其他專案中引用此函式庫的教學。
主要有兩個步驟：
1. 修改主專案 `settings.gradle.kts`，加入以下內容：
    ```gradle
    dependencyResolutionManagement {
        repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
        repositories {
            google()
            mavenCentral()
            maven(url = "https://jitpack.io")
        }
    }
    ```
    注意，JitPack提供的教學比較舊，在新版Android Studio中，應該如上設定。
2. 在要用到函式庫的模組中(通常是app)，宣告函式庫的依賴：
    ```gradle
    dependencies {
        // 直接宣告，形式為 groupId:artifactId:version
        implementation 'com.github.${github_account}:${name_of_artifact}:${version}'
        // Gradle Kotlin DSL 語法則如下:
        // implementation("com.github.${github_account}:${name_of_artifact}:${version}")
    }
    ```
    當然，也可使用 Gradle版本目錄 ：
    ```toml
    [versions]
    libversion = "${version}"

    [libraries]
    my-library = { module = "${groupId}:${artifactId}", version.ref = libversion }
    ```
    然後在 build.gradle.kts 中引用:
    ```gradle
    dependencies {
        implmentation(libs.my.library) //前綴 libs.，函式別名用點取代折線
    }
    ```
到此，自建的函式庫就可以透過 Maven + JitPack，在不同專案中複用。

---
# 附錄：函式庫發佈檢查表:
- [ ] 建立 Android 函式庫專案
    - [ ] 包含範例程式
    - [ ] 包含函式庫模組
        - [ ] 已加入 'maven-publish'
        - [ ] 已加入 'publishing' 區塊及資訊
        - [ ] 有沒有需要加上額外限制 (aarMetadata)
    - [ ] 已建立 GitHub 倉儲
- [ ] 檢查環境
    - [ ] JVM/JDK 版本是否一致
        - [ ] 是否需要設定JitPack Java版本(加入 `jitpack.yml` )
        - [ ] CI/CD服務上是否需要設定Java環境 
    - [ ] Android SDK版本是否適合
- [ ] 發佈測試
    - [ ] JitPack 能否建立函式庫
        - [ ] 是否有顯示對應的倉儲
        - [ ] 第一次建構是否成功
    - [ ] 範例程式是否能引用函式庫
- [ ] 正式發佈
    - [ ] 成功建立 GitHub 版本
    - [ ] JitPack 成功構建對應版本
- [ ] 主專案引用函式庫
    - [ ] 是否有設定 `maven(url = "https://jitpack.io")`
    - [ ] 是否有在依賴區塊中，宣告函式庫

# 附錄：[JitPackPublishHelpers](https://github.com/chenhaiteng/JitPackPublishHelpers)
    用來協助處理 "建立 Android 函式庫專案" 以及 "檢查環境" 兩項工作的小工具。

# 參考文件:
1. <a name="ref_1"/>[Guide to naming conventions on groupId, artifactId, and version](https://maven.apache.org/guides/mini/guide-naming-conventions.html)

2. <a name="ref_2"></a>[AarMetadata](https://developer.android.com/reference/tools/gradle-api/7.4/com/android/build/api/dsl/AarMetadata)


3. <a name="ref_3"></a>[Publish your library - Android Developers](https://developer.android.com/build/publish-library)

4. <a name="ref_4"></a>[Publish an Android library - JitPack](https://docs.jitpack.io/android/)

5. <a name="ref_5"/>[Java versions in Android builds - Android Developers](https://developer.android.com/build/jdks)

6. <a name="ref_6"/>[Building:Java Version - JitPack](https://docs.jitpack.io/building/#java-version)

7. <a name="ref_7"/>[Custom commands - JitPack](https://jitpack.io/docs/BUILDING/#custom-commands)

8. <a name="ref_8"/>[androidx.core - Android Developers](https://developer.android.com/jetpack/androidx/releases/core#1.15.0-alpha01)

9. <a name="ref_9"/>[Android Orb - Circle CI](https://github.com/circleci-public/android-orb)

10. <a name="ref_10"/>[Set up Compose for an existing app - Android Developers](https://developer.android.com/develop/ui/compose/setup#setup-compose)

11. <a name="ref_11"/>[Using a Version catalog - Gradle](https://docs.gradle.org/current/userguide/centralizing_dependencies.html#sub:using-catalogs)

12. <a name="ref_12"/>[Migrate your build to version catalogs - Android Developers](https://developer.android.com/build/migrate-to-catalogs)

13. <a name="ref_13"/>[GitHub CLI manual](https://cli.github.com/manual)