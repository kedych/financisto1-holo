# PROJECT_ANALYSIS.md — Financisto Holo

> Phase 1 現況盤點與理解報告（Tech Lead / DevOps / Security / QA 聯合輸出）

---

## 1. 專案用途

### 它解決什麼問題、給誰用

**Financisto Holo** 是一款 **離線優先（offline-first）的 Android 個人財務管理 App**。

* 目標使用者：需要精細追蹤個人/家庭支出、預算、多帳戶、多幣別的 Android 用戶。
* 核心理念：*No cloud, no online service.* 所有資料儲存於裝置本機 SQLite 資料庫，選擇性地透過 Google Drive 或 Dropbox 備份。
* 此 fork 由 `tiberiusteng` 從 Launchpad 舊版程式碼匯入，加上 Holo/Material 主題更新、文字縮放支援、通知樣板（由 SMS 樣板演進而來）、搜尋金額範圍等改進功能，並在 Google Play 以套件名 `tw.tib.financisto` 發布。

### 主要功能

| 功能模組 | 說明 |
|---|---|
| 帳戶管理 | 支援現金、銀行帳戶、信用卡、電子支付等多帳戶類型 |
| 交易流水帳（Blotter） | 新增、編輯、刪除、篩選收入/支出/轉帳；支援分帳（split）交易 |
| 預算管理 | 依分類/期間設定預算並追蹤達成率 |
| 報表 | 依分類/付款人/期間/地點等維度產生圓餅圖、長條圖 |
| 定期交易 | iCal RFC 2445 規則驅動的排程交易，支援提醒 |
| 多幣別 & 匯率 | 內建 FreeCurrency / OpenExchangeRates 下載器，支援歷史與即時匯率 |
| 備份/還原 | 本機備份（相容 Play Store 1.7.1 版格式）；Google Drive / Dropbox 備份 |
| 匯出 | QIF 格式（與主流桌面記帳軟體互通）、CSV |
| 通知樣板 | 解析 Push Notification（如銀行帳變動通知）自動建立交易 |
| 首頁小工具 | 2×1、3×1、4×1 帳戶餘額 Widget |
| 生物辨識鎖定 | 指紋/臉部解鎖 App |
| 分類屬性 | 可自訂交易屬性欄位，靈活擴充資料結構 |

---

## 2. 入口點與執行流程

### 入口點

```
AndroidManifest.xml
└── Application: androidx.multidex.MultiDexApplication
    └── Launcher Activity: tw.tib.financisto.activity.MainActivity
```

### 關鍵模組串聯

```
使用者啟動 App
    │
    ▼
MainActivity（帳戶清單 + 底部導覽）
    ├── AccountListFragment      ← 帳戶列表、餘額計算
    ├── BlotterActivity          ← 全局流水帳（含篩選、搜尋）
    ├── ReportsListFragment      ← 選擇報表類型
    ├── BudgetListFragment       ← 預算總覽
    └── PreferenceFragment       ← 設定（備份、匯率、主題等）

交易建立/編輯
    └── TransactionActivity / TransferActivity
            │
            ▼
        DatabaseAdapter（CRUD → SQLite via DatabaseHelper）
            │
            ▼
        financisto.db（本機 SQLite, DB version 231）

背景服務
    ├── IntentService            ← 處理 Intent 驅動的交易建立（通知樣板）
    ├── SmsReceiver              ← 監聽 SMS（若啟用）
    ├── NotificationListener     ← 監聽 Push Notification
    ├── RecurrenceScheduler      ← AlarmManager 驅動的定期交易
    └── DailyAutoBackupScheduler ← WorkManager 驅動的自動每日備份

備份/同步
    ├── DatabaseExport / DatabaseImport    ← 本機備份
    ├── DropboxBackupTask / DropboxRestoreTask
    └── Drive*Task（Google Drive）
```

---

## 3. 依賴與環境需求

### 語言 / 版本

| 項目 | 版本 |
|---|---|
| 語言 | Java 17 |
| minSdkVersion | 21（Android 5.0） |
| targetSdkVersion | 36（Android 16） |
| compileSdk | 36 |
| Android Gradle Plugin | 8.11.0 |
| Gradle Wrapper | 見 `gradle/wrapper/gradle-wrapper.properties` |

### 核心依賴

| 函式庫 | 版本 | 用途 |
|---|---|---|
| AndroidAnnotations | 4.8.0 | DI / Code generation（`@EBean`, `@EActivity`…） |
| GreenRobot EventBus | 3.3.1 | App 內事件匯流排 |
| MPAndroidChart | 3.1.0 | 圖表（圓餅/長條） |
| Google API Client Android | 2.7.1 | Google Drive API |
| Google Drive API v3 | v3-rev20241206-2.0.0 | Drive 備份/還原 |
| google-auth-library | 1.30.1 | OAuth2 認證 |
| Dropbox Core SDK | 7.0.0 | Dropbox 備份 |
| OkHttp3 | 3.11.0 | HTTP（匯率下載） |
| Play Services Auth | 20.7.0 | Google 帳號登入 |
| Glide | 4.16.0 | 圖片載入 |
| RxJava2 / RxAndroid | 2.1.9 / 2.0.2 | 非同步操作 |
| AndroidX 套件 | 各版本 | UI 元件、Biometric、WorkManager 等 |
| commons-io | 2.5 | 檔案 I/O |
| rfc2445-no-joda.jar | 本機 lib | iCal 規則解析（定期交易） |
| trove-3.0.2-long.jar | 本機 lib | 高效能原始型別集合 |

### 外部服務（選用 / 可插拔）

| 服務 | 必要性 | 設定方式 |
|---|---|---|
| Google Drive | 選用 | OAuth2 via Google Sign-In（App 設定頁啟用） |
| Dropbox | 選用 | OAuth2 via Dropbox SDK（App 設定頁啟用） |
| OpenExchangeRates | 選用 | 在設定中填入 App ID |
| FreeCurrencyAPI | 選用 | 免費 API，無需 Key（預設） |

> **無任何硬編碼 API Key 或密碼**。所有雲端服務均以「可插拔」方式設計，未設定時不影響核心功能。

---

## 4. 重要資料流與狀態

### 資料怎麼進來

1. **手動輸入**：`TransactionActivity` → `DatabaseAdapter.insertOrUpdate()`
2. **通知解析**：`NotificationListener` → `SmsTransactionProcessor` → `DatabaseAdapter`
3. **備份還原**：`DatabaseImport` → 讀取備份檔案 → `DatabaseAdapter`
4. **QIF 匯入**：`QifImportActivity` → `QifParser` → `DatabaseAdapter`
5. **定期交易觸發**：`RecurrenceScheduler` (AlarmManager) → `IntentService` → `DatabaseAdapter`

### 資料怎麼存

* 本機 SQLite 檔案：`/data/data/tw.tib.financisto/databases/financisto.db`
* 主要資料表：`transactions`, `account`, `currency`, `category`, `budget`, `project`, `payee`, `attributes`, `sms_template`, `currency_exchange_rate`, `delete_log`, `ccard_closing_date`, `locations`
* 大量 SQL View（`v_blotter`, `v_account`, `v_report_*` 等）用於高效查詢
* 資料庫版本：**231**（由 `DatabaseSchemaEvolution` 自動執行增量 migration script）

### 資料怎麼出去

1. **螢幕顯示**：CursorLoader → `BlotterListAdapter` / `AccountListAdapter` → RecyclerView/ListView
2. **本機備份**：`DatabaseExport` → 寫入 `.backup` 檔（gzip 壓縮文字格式）
3. **Google Drive**：`DriveBackupTask` → Drive API v3 上傳
4. **Dropbox**：`DropboxBackupTask` → Dropbox Core SDK 上傳
5. **QIF 匯出**：`QifExportActivity` → `.qif` 檔
6. **CSV 匯出**：`CsvExportActivity` → `.csv` 檔
7. **首頁 Widget**：`AccountWidget*` → `AppWidgetManager` 更新餘額顯示

---

## 5. 目錄結構導覽

```
financisto1-holo/
├── build.gradle                    # 根專案 Gradle 設定
├── settings.gradle                 # 模組宣告（只有 :app）
├── gradle.properties               # JVM 堆疊設定、AndroidX 啟用
├── license.txt                     # GPL v2 授權條款
├── README.md                       # 專案簡介（現況）
├── docs/
│   └── screenshots/                # App 截圖（accounts, blotter, transaction, autocomplete）
└── app/
    ├── build.gradle                # App 模組依賴與 Android 設定
    ├── proguard-rules.txt          # R8/ProGuard 混淆規則
    ├── proguard-google-api-client.txt
    ├── libs/
    │   ├── rfc2445-no-joda.jar     # iCal RFC 2445 解析器（本機 lib）
    │   └── trove-3.0.2-long.jar   # 高效能 long 型別集合
    └── src/main/
        ├── AndroidManifest.xml     # 元件宣告、權限、版本資訊
        ├── assets/db/              # SQL migration 腳本（update_*.sql）
        ├── res/                    # 佈局、字串、圖示、主題資源
        └── java/
            └── tw/tib/financisto/
                ├── activity/       # Activity + Fragment（UI 入口）
                ├── adapter/        # RecyclerView/ListView Adapter
                ├── backup/         # 備份/還原邏輯
                ├── blotter/        # 流水帳篩選器與工具類
                ├── bus/            # EventBus 事件定義
                ├── db/             # SQLite 資料存取層（DatabaseAdapter, DatabaseHelper）
                ├── dialog/         # 自訂對話框
                ├── export/         # QIF/CSV/Dropbox/Drive 匯出
                ├── filter/         # 交易篩選條件
                ├── graph/          # 圖表資料模型
                ├── http/           # HTTP 包裝器
                ├── model/          # 資料模型（Account, Transaction, Budget…）
                ├── preference/     # SharedPreferences 包裝
                ├── rates/          # 匯率提供者與下載器
                ├── recur/          # 定期交易 iCal 規則
                ├── report/         # 報表邏輯
                ├── service/        # 背景服務（IntentService, Scheduler…）
                ├── utils/          # 通用工具類
                ├── view/           # 自訂 View 元件
                ├── widget/         # 首頁 Widget
                └── worker/         # WorkManager Worker（自動備份）
```

---

## 6. 已知問題清單

### 可維護性

| # | 問題 | 說明 |
|---|---|---|
| M1 | 大量使用 `AndroidAnnotations`（@EBean / @EActivity） | 此框架已多年未積極維護，升級路徑受限，IDE 支援較差 |
| M2 | 部分 Activity 同時承擔 ViewModel 職責 | 違反 MVP/MVVM 分離原則，測試困難 |
| M3 | 大型 `DatabaseAdapter.java` | 單一類別承擔所有資料存取操作，違反 SRP |
| M4 | 本機 JAR 依賴 | `rfc2445-no-joda.jar`, `trove-3.0.2-long.jar` 無法透過 Gradle 版本管理 |
| M5 | `AsyncTask`（部分殘留） | Android 已棄用，應遷移至 `coroutine` / `WorkManager` |

### 測試

| # | 問題 |
|---|---|
| T1 | 測試覆蓋率極低，僅有少量 Instrumented Test（`androidTest`），無單元測試（`test`）目錄 |
| T2 | 核心業務邏輯（`DatabaseAdapter`, `TransactionsTotalCalculator`）缺乏自動化測試保護 |

### 效能

| # | 問題 |
|---|---|
| P1 | 部分查詢在主執行緒執行（舊版 `CursorAdapter` 模式），可能造成 ANR |
| P2 | 大量資料（數萬筆交易）時，流水帳頁面捲動效能待優化 |

### 安全性

| # | 問題 | 嚴重度 | 修補建議 |
|---|---|---|---|
| S1 | SQLite 查詢使用字串拼接（部分地方） | 中 | 統一改用 `?` 參數化查詢 |
| S2 | Dropbox Access Token 儲存於 `SharedPreferences`（未加密） | 低-中 | 改用 `EncryptedSharedPreferences`（AndroidX Security） |
| S3 | Google Drive OAuth Token 委由 Google Sign-In 管理，風險較低，但 Refresh Token 持久化存放 | 低 | 使用 Android Keystore 加密存放 |
| S4 | `allowBackup="true"` 在 AndroidManifest | 低 | 若不需要 ADB 備份，建議設為 `false` 以防資料外洩 |
| S5 | 備份檔案無加密 | 低-中 | 提供可選加密備份功能 |

### 可靠性

| # | 問題 |
|---|---|
| R1 | 自動備份若在裝置儲存空間不足時無告警 |
| R2 | Google Drive / Dropbox 同步失敗時，錯誤通知不夠明顯 |

### 部署

| # | 問題 |
|---|---|
| D1 | 缺少 CI/CD 設定（無 GitHub Actions、Fastlane 等） |
| D2 | `signingConfig` 在 release build 仍使用 `debug` 簽名，發布前需手動更改 |
| D3 | 版本號（`versionCode`, `versionName`）需手動維護於 `AndroidManifest.xml` |

---

*報告產出時間：2026-03-02*
