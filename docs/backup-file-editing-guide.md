# Financisto Backup 檔案完整操作指南

> 本文件詳細說明 Financisto1-Holo 專案的帳本（Account）操作方式、SQLite 資料表結構、`.backup` 檔案的匯出/還原邏輯，以及如何在 Google Sheets 或 Excel 等試算表環境中正確編輯 `.backup` 檔案。

---

## 目錄

1. [系統架構總覽](#1-系統架構總覽)
2. [SQLite 資料表結構](#2-sqlite-資料表結構)
   - [2.1 currency（幣別）](#21-currency幣別)
   - [2.2 account（帳本/帳戶）](#22-account帳本帳戶)
   - [2.3 category（分類）](#23-category分類)
   - [2.4 project（專案）](#24-project專案)
   - [2.5 payee（收款人/付款對象）](#25-payee收款人付款對象)
   - [2.6 locations（地點）](#26-locations地點)
   - [2.7 transactions（交易）](#27-transactions交易)
   - [2.8 budget（預算）](#28-budget預算)
   - [2.9 attributes / category_attribute / transaction_attribute（自訂屬性）](#29-attributes--category_attribute--transaction_attribute自訂屬性)
   - [2.10 currency_exchange_rate（匯率）](#210-currency_exchange_rate匯率)
   - [2.11 ccard_closing_date（信用卡結帳日）](#211-ccard_closing_date信用卡結帳日)
   - [2.12 sms_template（簡訊範本）](#212-sms_template簡訊範本)
   - [2.13 running_balance（執行餘額快取）](#213-running_balance執行餘額快取)
3. [帳本操作流程](#3-帳本操作流程)
   - [3.1 新增帳本](#31-新增帳本)
   - [3.2 新增一般交易（支出/收入）](#32-新增一般交易支出收入)
   - [3.3 帳本間轉帳](#33-帳本間轉帳)
   - [3.4 分拆交易（Split Transaction）](#34-分拆交易split-transaction)
   - [3.5 餘額計算邏輯](#35-餘額計算邏輯)
   - [3.6 刪除帳本](#36-刪除帳本)
4. [.backup 檔案格式詳解](#4-backup-檔案格式詳解)
   - [4.1 匯出（Export）流程](#41-匯出export流程)
   - [4.2 檔案格式結構](#42-檔案格式結構)
   - [4.3 還原（Import）流程](#43-還原import流程)
   - [4.4 還原時的特殊處理](#44-還原時的特殊處理)
5. [在試算表中編輯 .backup 檔案](#5-在試算表中編輯-backup-檔案)
   - [5.1 前置準備](#51-前置準備)
   - [5.2 解壓縮 .backup 檔案](#52-解壓縮-backup-檔案)
   - [5.3 將 .backup 轉換為 CSV/試算表格式](#53-將-backup-轉換為-csv試算表格式)
   - [5.4 在試算表中編輯](#54-在試算表中編輯)
   - [5.5 將試算表轉回 .backup 格式](#55-將試算表轉回-backup-格式)
   - [5.6 重新壓縮與匯入](#56-重新壓縮與匯入)
   - [5.7 完整 Python 工具腳本](#57-完整-python-工具腳本)
6. [注意事項與常見問題](#6-注意事項與常見問題)

---

## 1. 系統架構總覽

Financisto 是一個 Android 個人財務管理應用程式，使用 **SQLite** 作為本地資料庫，並採用**複式記帳（Double-Entry Bookkeeping）** 的概念來記錄交易。

### 核心概念

| 概念 | 說明 |
|------|------|
| **帳本（Account）** | 代表一個資金存放處，如現金、銀行帳戶、信用卡等 |
| **幣別（Currency）** | 每個帳本綁定一個幣別 |
| **交易（Transaction）** | 記錄資金的進出，包含來源帳本和目標帳本 |
| **轉帳（Transfer）** | 兩個帳本之間的資金移動，`to_account_id > 0` 即為轉帳 |
| **分拆交易（Split）** | 一筆交易分成多個子交易，各自對應不同分類 |
| **分類（Category）** | 使用巢狀集合模型（Nested Set Model）組織階層分類 |
| **備份（Backup）** | 以純文字 key:value 格式匯出所有表格資料，可選 GZip 壓縮 |

### 備份涵蓋的表格

```
account, attributes, category_attribute, transaction_attribute,
budget, category, currency, locations, project, transactions,
payee, ccard_closing_date, sms_template, split, currency_exchange_rate
```

> **注意：** `running_balance` 和 `delete_log` 表格**不包含**在備份中，還原後會自動重建。

---

## 2. SQLite 資料表結構

### 金額存儲規則

> **重要：** Financisto 中所有金額欄位都以 **整數（integer/long）** 儲存，實際值需除以 100。
> 例如：`from_amount = -15000` 代表 **-150.00**（支出 150 元）。
> 收入為正數，支出為負數。

### 2.1 currency（幣別）

| 欄位 | 型別 | 說明 |
|------|------|------|
| `_id` | INTEGER PK | 幣別 ID（自動遞增） |
| `name` | TEXT NOT NULL | 幣別代碼（如 `TWD`, `USD`） |
| `title` | TEXT NOT NULL | 幣別名稱（如 `新台幣`, `US Dollar`） |
| `symbol` | TEXT NOT NULL | 幣別符號（如 `$`, `NT$`） |
| `is_default` | INTEGER | 是否為預設幣別（`1` 是 / `0` 否） |
| `decimals` | INTEGER | 小數位數（預設 `2`） |
| `decimal_separator` | TEXT | 小數分隔符號 |
| `group_separator` | TEXT | 千分位分隔符號 |
| `symbol_format` | TEXT | 符號位置（`RS` 右+空格, `R` 右, `LS` 左+空格, `L` 左） |
| `number_format` | TEXT | 數字格式 |
| `is_active` | BOOLEAN | 是否啟用（預設 `1`） |
| `sort_order` | INTEGER | 排序順序 |
| `update_exchange_rate` | INTEGER | 是否自動更新匯率（預設 `1`） |

### 2.2 account（帳本/帳戶）

| 欄位 | 型別 | 說明 |
|------|------|------|
| `_id` | INTEGER PK | 帳本 ID（自動遞增） |
| `title` | TEXT NOT NULL | 帳本名稱 |
| `creation_date` | LONG NOT NULL | 建立時間（Unix 毫秒） |
| `currency_id` | INTEGER NOT NULL | 綁定的幣別 ID（外鍵→ `currency._id`） |
| `total_amount` | INTEGER | 目前餘額（整數，除以100為實際值） |
| `type` | TEXT | 帳本類型（見下表） |
| `card_issuer` | TEXT | 卡片發行商 |
| `issuer` | TEXT | 銀行/發行機構 |
| `number` | TEXT | 帳號/卡號 |
| `total_limit` | INTEGER | 信用額度（信用卡用） |
| `closing_day` | INTEGER | 結帳日（信用卡用） |
| `payment_day` | INTEGER | 繳款日（信用卡用） |
| `is_include_into_totals` | BOOLEAN | 是否納入總計（預設 `1`） |
| `is_active` | BOOLEAN | 是否啟用（預設 `1`） |
| `sort_order` | INTEGER | 排序順序 |
| `last_category_id` | LONG | 最近使用的分類 ID |
| `last_account_id` | LONG | 最近轉帳的目標帳本 ID |
| `last_transaction_date` | LONG | 最近交易時間 |
| `note` | TEXT | 備註 |
| `icon` | TEXT | 圖示名稱 |
| `accent_color` | TEXT | 強調顏色 |

#### 帳本類型（AccountType）

| 值 | 說明 |
|----|------|
| `CASH` | 現金 |
| `BANK` | 銀行帳戶 |
| `DEBIT_CARD` | 簽帳卡 |
| `CREDIT_CARD` | 信用卡 |
| `ELECTRONIC` | 電子支付（PayPal 等） |
| `ASSET` | 資產 |
| `LIABILITY` | 負債 |
| `OTHER` | 其他 |

### 2.3 category（分類）

使用 **巢狀集合模型（Nested Set Model）** 儲存階層結構。

| 欄位 | 型別 | 說明 |
|------|------|------|
| `_id` | INTEGER PK | 分類 ID |
| `title` | TEXT NOT NULL | 分類名稱 |
| `left` | INTEGER | 巢狀集合左值 |
| `right` | INTEGER | 巢狀集合右值 |
| `type` | INTEGER | 類型（`0` 支出 / `1` 收入） |
| `last_location_id` | LONG | 最近使用的地點 ID |
| `last_project_id` | LONG | 最近使用的專案 ID |
| `sort_order` | INTEGER | 排序順序 |
| `is_active` | BOOLEAN | 是否啟用 |

> **特殊 ID：** `_id = 0` 為「無分類」、`_id = -1` 為「分拆分類（Split Category）」。

### 2.4 project（專案）

| 欄位 | 型別 | 說明 |
|------|------|------|
| `_id` | INTEGER PK | 專案 ID |
| `title` | TEXT | 專案名稱 |
| `is_active` | BOOLEAN | 是否啟用 |
| `sort_order` | INTEGER | 排序順序 |

### 2.5 payee（收款人/付款對象）

| 欄位 | 型別 | 說明 |
|------|------|------|
| `_id` | INTEGER PK | 收款人 ID |
| `title` | TEXT | 收款人名稱 |
| `last_category_id` | LONG | 最近使用的分類 ID |
| `is_active` | BOOLEAN | 是否啟用 |
| `sort_order` | INTEGER | 排序順序 |

### 2.6 locations（地點）

| 欄位 | 型別 | 說明 |
|------|------|------|
| `_id` | INTEGER PK | 地點 ID |
| `title` | TEXT NOT NULL | 地點名稱（原欄位名為 `name`，後改為 `title`） |
| `datetime` | LONG NOT NULL | 建立時間 |
| `provider` | TEXT | GPS 提供者 |
| `accuracy` | FLOAT | GPS 精確度 |
| `latitude` | DOUBLE | 緯度 |
| `longitude` | DOUBLE | 經度 |
| `is_payee` | INTEGER | 是否為收款人（`0`/`1`） |
| `resolved_address` | TEXT | 反向地理編碼地址 |
| `count` | INTEGER | 使用次數 |
| `is_active` | BOOLEAN | 是否啟用 |
| `sort_order` | INTEGER | 排序順序 |

### 2.7 transactions（交易）

這是最核心的表格，記錄所有的財務交易。

| 欄位 | 型別 | 說明 |
|------|------|------|
| `_id` | INTEGER PK | 交易 ID（自動遞增） |
| `parent_id` | LONG | 父交易 ID（分拆交易時使用，`0` 代表非子交易） |
| `from_account_id` | LONG NOT NULL | 來源帳本 ID（外鍵→ `account._id`） |
| `to_account_id` | LONG | 目標帳本 ID（轉帳時使用，`0` 代表非轉帳） |
| `category_id` | LONG | 分類 ID（外鍵→ `category._id`） |
| `project_id` | LONG | 專案 ID |
| `payee_id` | LONG | 收款人 ID |
| `location_id` | LONG | 地點 ID |
| `note` | TEXT | 交易備註 |
| `from_amount` | INTEGER NOT NULL | 來源帳本金額（負數=支出，正數=收入） |
| `to_amount` | INTEGER NOT NULL | 目標帳本金額（轉帳時使用） |
| `datetime` | LONG NOT NULL | 交易時間（Unix 毫秒） |
| `original_currency_id` | LONG | 原始幣別 ID（外幣交易時使用） |
| `original_from_amount` | LONG | 原始幣別金額 |
| `is_template` | INTEGER | `0`=一般交易, `1`=範本, `2`=排程交易 |
| `template_name` | TEXT | 範本名稱 |
| `recurrence` | TEXT | 重複規則 |
| `notification_options` | TEXT | 通知設定 |
| `status` | TEXT | 交易狀態（見下表） |
| `attached_picture` | TEXT | 附加圖片路徑 |
| `is_ccard_payment` | INTEGER | 是否為信用卡還款 |
| `last_recurrence` | LONG | 最近執行的排程時間 |
| `blob_key` | TEXT | 二進制大物件金鑰 |
| `provider` | TEXT | GPS 提供者 |
| `accuracy` | FLOAT | GPS 精確度 |
| `latitude` | DOUBLE | 緯度 |
| `longitude` | DOUBLE | 經度 |

#### 交易狀態（TransactionStatus）

| 值 | 說明 |
|----|------|
| `UR` | 未對帳（Unreconciled）— 預設值 |
| `PN` | 待處理（Pending） |
| `CL` | 已結算（Cleared） |
| `RC` | 已對帳（Reconciled） |
| `RS` | 已還原（Restored） |

#### 交易類型判斷

| 條件 | 交易類型 |
|------|----------|
| `to_account_id = 0` | **一般交易**（支出或收入） |
| `to_account_id > 0` | **帳本間轉帳** |
| `category_id = -1` | **分拆交易父項** |
| `parent_id > 0` | **分拆交易子項** |
| `is_template = 1` | **範本交易** |
| `is_template = 2` | **排程交易** |

### 2.8 budget（預算）

| 欄位 | 型別 | 說明 |
|------|------|------|
| `_id` | INTEGER PK | 預算 ID |
| `title` | TEXT | 預算名稱 |
| `category_id` | TEXT | 分類 ID（可為逗號分隔的多個 ID） |
| `project_id` | TEXT | 專案 ID（可為逗號分隔的多個 ID） |
| `currency_id` | LONG | 幣別 ID |
| `budget_currency_id` | LONG | 預算幣別 ID |
| `budget_account_id` | LONG | 預算帳本 ID |
| `amount` | INTEGER | 預算金額 |
| `include_subcategories` | INTEGER | 是否包含子分類 |
| `include_credit` | INTEGER | 是否包含收入 |
| `expanded` | INTEGER | 是否展開 |
| `start_date` | LONG | 開始日期 |
| `end_date` | LONG | 結束日期 |
| `recur` | TEXT | 重複規則 |
| `recur_num` | INTEGER | 重複次數 |
| `is_current` | INTEGER | 是否為目前的 |
| `parent_budget_id` | LONG | 父預算 ID |
| `sort_order` | INTEGER | 排序順序 |

### 2.9 attributes / category_attribute / transaction_attribute（自訂屬性）

#### attributes 表

| 欄位 | 型別 | 說明 |
|------|------|------|
| `_id` | INTEGER PK | 屬性 ID |
| `type` | INTEGER | 屬性類型（`1`=文字, `2`=數字, `3`=列表選擇, `4`=核取方塊, `5`=日期） |
| `title` | TEXT NOT NULL | 屬性名稱（原欄位名為 `name`，後改為 `title`） |
| `list_values` | TEXT | 列表值（分號分隔） |
| `default_value` | TEXT | 預設值 |
| `sort_order` | INTEGER | 排序順序 |

#### category_attribute 表

| 欄位 | 型別 | 說明 |
|------|------|------|
| `category_id` | INTEGER | 分類 ID |
| `attribute_id` | INTEGER | 屬性 ID |

#### transaction_attribute 表

| 欄位 | 型別 | 說明 |
|------|------|------|
| `transaction_id` | INTEGER | 交易 ID |
| `attribute_id` | INTEGER | 屬性 ID |
| `value` | TEXT | 屬性值 |

### 2.10 currency_exchange_rate（匯率）

| 欄位 | 型別 | 說明 |
|------|------|------|
| `from_currency_id` | INTEGER NOT NULL | 來源幣別 ID（複合主鍵之一） |
| `to_currency_id` | INTEGER NOT NULL | 目標幣別 ID（複合主鍵之二） |
| `rate_date` | LONG NOT NULL | 匯率日期（複合主鍵之三） |
| `rate` | FLOAT NOT NULL | 匯率 |

> **主鍵：** `(from_currency_id, to_currency_id, rate_date)`

### 2.11 ccard_closing_date（信用卡結帳日）

| 欄位 | 型別 | 說明 |
|------|------|------|
| `account_id` | LONG NOT NULL | 帳本 ID |
| `period` | INTEGER NOT NULL | 期間（MMYYYY，MM 從 0 開始） |
| `closing_day` | INTEGER NOT NULL | 結帳日 |

### 2.12 sms_template（簡訊範本）

| 欄位 | 型別 | 說明 |
|------|------|------|
| `_id` | INTEGER PK | 範本 ID |
| `title` | TEXT NOT NULL | 範本標題（發送者號碼） |
| `template` | TEXT NOT NULL | 範本內容（正則表達式） |
| `category_id` | INTEGER NOT NULL | 分類 ID |
| `account_id` | INTEGER | 帳本 ID |
| `to_account_id` | INTEGER | 目標帳本 ID |
| `payee_id` | INTEGER | 收款人 ID |
| `project_id` | INTEGER | 專案 ID |
| `is_income` | BOOLEAN | 是否為收入 |
| `note` | TEXT | 備註 |
| `is_active` | BOOLEAN | 是否啟用 |
| `sort_order` | INTEGER | 排序順序 |

### 2.13 running_balance（執行餘額快取）

> **此表不包含在備份中**，還原後自動重新計算。

| 欄位 | 型別 | 說明 |
|------|------|------|
| `account_id` | INTEGER NOT NULL | 帳本 ID（複合主鍵之一） |
| `transaction_id` | INTEGER NOT NULL | 交易 ID（複合主鍵之二） |
| `datetime` | LONG NOT NULL | 交易時間 |
| `balance` | INTEGER NOT NULL | 截至該交易的累計餘額 |

---

## 3. 帳本操作流程

### 3.1 新增帳本

**步驟：**

1. 先確認 `currency` 表中有對應幣別，若無則先新增
2. 在 `account` 表中插入新紀錄

**對應 SQL：**

```sql
-- 1. 新增幣別（若不存在）
INSERT INTO currency (_id, name, title, symbol, is_default, decimals, symbol_format)
VALUES (1, 'TWD', '新台幣', 'NT$', 1, 2, 'LS');

-- 2. 新增帳本
INSERT INTO account (_id, title, creation_date, currency_id, total_amount, type,
                     is_include_into_totals, is_active, sort_order)
VALUES (1, '我的錢包', 1709395200000, 1, 0, 'CASH', 1, 1, 1);
```

**在 .backup 格式中：**

```
$ENTITY:currency
_id:1
name:TWD
title:新台幣
symbol:NT$
is_default:1
decimals:2
symbol_format:LS
$$
$ENTITY:account
_id:1
title:我的錢包
creation_date:1709395200000
currency_id:1
total_amount:0
type:CASH
is_include_into_totals:1
is_active:1
sort_order:1
$$
```

### 3.2 新增一般交易（支出/收入）

**支出範例：** 從「我的錢包」支出 150 元買午餐

```sql
-- from_amount 為負數代表支出，金額 x100
INSERT INTO transactions
  (_id, from_account_id, to_account_id, category_id, from_amount, to_amount,
   datetime, is_template, status)
VALUES
  (1, 1, 0, 5, -15000, 0, 1709481600000, 0, 'UR');

-- 同步更新帳本餘額
UPDATE account SET total_amount = total_amount + (-15000) WHERE _id = 1;
```

**收入範例：** 收到 50000 元薪水

```sql
INSERT INTO transactions
  (_id, from_account_id, to_account_id, category_id, from_amount, to_amount,
   datetime, is_template, status)
VALUES
  (2, 1, 0, 10, 5000000, 0, 1709481600000, 0, 'UR');

-- 同步更新帳本餘額
UPDATE account SET total_amount = total_amount + 5000000 WHERE _id = 1;
```

### 3.3 帳本間轉帳

**轉帳：** 從「我的錢包」(ID=1) 轉 1000 元到「銀行帳戶」(ID=2)

```sql
INSERT INTO transactions
  (_id, from_account_id, to_account_id, category_id,
   from_amount, to_amount, datetime, is_template, status)
VALUES
  (3, 1, 2, 0, -100000, 100000, 1709481600000, 0, 'UR');

-- 更新來源帳本（減少）
UPDATE account SET total_amount = total_amount + (-100000) WHERE _id = 1;
-- 更新目標帳本（增加）
UPDATE account SET total_amount = total_amount + 100000 WHERE _id = 2;
```

**關鍵規則：**
- `from_amount` 為**負數**（來源帳本扣款）
- `to_amount` 為**正數**（目標帳本入帳）
- `to_account_id > 0` 標示此為轉帳交易
- 若兩個帳本幣別不同，`from_amount` 和 `to_amount` 的絕對值可以不同（代表匯率轉換）

**跨幣別轉帳範例：** 從台幣帳戶轉 3000 TWD 到美金帳戶（匯率約 1:30）

```sql
INSERT INTO transactions
  (_id, from_account_id, to_account_id, category_id,
   from_amount, to_amount, datetime, is_template, status)
VALUES
  (4, 1, 3, 0, -300000, 10000, 1709481600000, 0, 'UR');
  -- from_amount = -3000.00 TWD
  -- to_amount   =   100.00 USD
```

### 3.4 分拆交易（Split Transaction）

將一筆交易拆分到多個分類。例如：超市消費 500 元，其中食品 300 + 日用品 200。

```sql
-- 1. 父交易：category_id = -1 (SPLIT_CATEGORY_ID)
INSERT INTO transactions
  (_id, parent_id, from_account_id, to_account_id, category_id,
   from_amount, to_amount, datetime, is_template, status)
VALUES
  (5, 0, 1, 0, -1, -50000, 0, 1709568000000, 0, 'UR');

-- 2. 子交易 1：食品 300 元
INSERT INTO transactions
  (_id, parent_id, from_account_id, to_account_id, category_id,
   from_amount, to_amount, datetime, is_template, status)
VALUES
  (6, 5, 1, 0, 6, -30000, 0, 1709568000000, 0, 'UR');

-- 3. 子交易 2：日用品 200 元
INSERT INTO transactions
  (_id, parent_id, from_account_id, to_account_id, category_id,
   from_amount, to_amount, datetime, is_template, status)
VALUES
  (7, 5, 1, 0, 7, -20000, 0, 1709568000000, 0, 'UR');
```

**關鍵規則：**
- 父交易的 `category_id = -1`
- 子交易的 `parent_id` 指向父交易 `_id`
- 子交易金額總和必須等於父交易金額
- 帳本餘額只根據父交易更新（子交易不重複計算，但子交易中的轉帳目標帳本會被更新）

### 3.5 餘額計算邏輯

帳本餘額有兩個層面：

1. **`account.total_amount`**：帳本的總餘額，每次交易時即時更新
2. **`running_balance`**：每筆交易時刻的累計餘額快取（用於顯示交易清單中的餘額）

**還原時重建餘額（IntegrityFix）：**

```
1. restoreSystemEntities()   — 還原系統內建項目
2. recalculateAccountsBalances() — 重新計算所有帳本的 total_amount
3. rebuildRunningBalances()   — 重建 running_balance 表
```

> **重要：** 在手動編輯 `.backup` 檔案時，`account.total_amount` 的值不需要精確，因為還原後會自動重新計算。但交易的 `from_amount` 和 `to_amount` 必須正確，因為餘額是基於這些值計算的。

### 3.6 刪除帳本

刪除帳本時的處理邏輯：

```sql
-- 1. 把指向該帳本為目標的轉帳交易，改為普通交易
UPDATE transactions SET to_account_id=0, to_amount=0 WHERE to_account_id=?;

-- 2. 把該帳本為來源的轉帳交易，改為目標帳本的普通交易
UPDATE transactions SET from_account_id=to_account_id, from_amount=to_amount,
       to_account_id=0, to_amount=0, parent_id=0
WHERE from_account_id=? AND to_account_id>0;

-- 3. 刪除剩餘相關紀錄
DELETE FROM transaction_attribute WHERE transaction_id IN
       (SELECT _id FROM transactions WHERE from_account_id=?);
DELETE FROM transactions WHERE from_account_id=?;
DELETE FROM account WHERE _id=?;
```

---

## 4. .backup 檔案格式詳解

### 4.1 匯出（Export）流程

**原始碼：** `DatabaseExport.java`

匯出流程依序為：

```
1. 寫入檔案標頭（Header）
2. 依序匯出每個 BACKUP_TABLES 的資料
3. 寫入檔案結尾（Footer）
4. 選擇性 GZip 壓縮
```

**匯出表格順序：**

```
account → attributes → category_attribute → transaction_attribute →
budget → category → currency → locations → project → transactions →
payee → ccard_closing_date → sms_template → split → currency_exchange_rate
```

**各表的匯出條件：**

| 表格 | 過濾系統 ID（`_id > 0`） | 依 `sort_order` 排序 |
|------|:-:|:-:|
| `account` | ✗ | ✓（自訂排序） |
| `attributes` | ✓ | ✓ |
| `category_attribute` | ✗ | ✗ |
| `transaction_attribute` | ✗ | ✗ |
| `budget` | ✗ | ✓ |
| `category` | ✓ | ✗ |
| `currency` | ✗ | ✓ |
| `locations` | ✓ | ✓ |
| `project` | ✓ | ✓ |
| `transactions` | ✗ | ✗ |
| `payee` | ✗ | ✓ |
| `ccard_closing_date` | ✗ | ✗ |
| `sms_template` | ✗ | ✓ |
| `currency_exchange_rate` | ✗ | ✗ |

> **注意：** 具有 `sort_order` 的表格匯出時會排序，但 `sort_order` 欄位值本身**不會**被寫入備份（除了 `account` 表例外）。還原時會按讀取順序自動分配新的 `sort_order`。

### 4.2 檔案格式結構

`.backup` 檔案是純文字格式（UTF-8 編碼），通常以 GZip 壓縮。

```
PACKAGE:tw.tib.financisto1
VERSION_CODE:126
VERSION_NAME:1.8.3
DATABASE_VERSION:238
#START
$ENTITY:account
_id:1
title:我的錢包
creation_date:1709395200000
currency_id:1
total_amount:50000
type:CASH
is_include_into_totals:1
is_active:1
sort_order:1
$$
$ENTITY:account
_id:2
title:銀行帳戶
creation_date:1709395200000
currency_id:1
total_amount:1000000
type:BANK
is_active:1
sort_order:2
$$
$ENTITY:currency
_id:1
name:TWD
title:新台幣
symbol:NT$
is_default:1
decimals:2
$$
$ENTITY:transactions
_id:1
from_account_id:1
to_account_id:0
category_id:5
from_amount:-15000
to_amount:0
datetime:1709481600000
is_template:0
status:UR
$$
#END
```

**格式規則：**

| 標記 | 說明 |
|------|------|
| `PACKAGE:` | 應用程式套件名稱 |
| `VERSION_CODE:` | 應用程式版本號 |
| `VERSION_NAME:` | 應用程式版本名稱 |
| `DATABASE_VERSION:` | 資料庫架構版本 |
| `#START` | 資料區塊開始 |
| `$ENTITY:表名` | 實體開始，後接表格名稱 |
| `欄位名:值` | 欄位名與值以冒號分隔，一行一個 |
| `$$` | 實體結束 |
| `#END` | 檔案結束 |

**重要格式細節：**
- 值中的換行符 (`\n`) 會被替換為空格
- 值為 `null` 的欄位**不會**被寫入
- 欄位之間以換行分隔
- 沒有引號包圍值

### 4.3 還原（Import）流程

**原始碼：** `DatabaseImport.java` + `FullDatabaseImport.java`

```
1. 自動偵測並解壓縮 GZip（若有壓縮）
2. 開始資料庫交易（BEGIN TRANSACTION）
3. 清空所有備份表格 + running_balance
4. 逐行讀取 .backup 檔案
5. 遇到 $ENTITY: 開始收集欄位
6. 遇到 $$ 結束一筆記錄，清理後 INSERT
7. 完成後執行還原修復腳本（RESTORE_SCRIPTS）
8. 提交交易（COMMIT）
9. 執行完整性修復（IntegrityFix）
   - 還原系統實體
   - 重新計算所有帳本餘額
   - 重建 running_balance
10. 初始化幣別快取
11. 排程所有排程交易
```

### 4.4 還原時的特殊處理

| 處理項目 | 說明 |
|----------|------|
| **系統實體過濾** | `_id <= 0` 的記錄會被跳過（不還原） |
| **移除欄位** | `updated_on` 和 `remote_key` 欄位會被移除 |
| **地點欄位重新命名** | `locations` 表的 `name` 欄位會複製到 `title` 欄位 |
| **屬性欄位重新命名** | `attributes` 表的 `name` 欄位會改為 `title` 欄位 |
| **未知欄位移除** | 不存在於目前資料庫架構的欄位會被自動忽略 |
| **自動排序** | 若備份中缺少 `sort_order`，會自動按讀取順序分配遞增值 |
| **帳本類型修復** | `PAYPAL` → `ELECTRONIC`, 舊卡片類型 → `DEBIT_CARD` |
| **範本分拆修復** | 確保分拆交易的子項繼承範本/排程標記 |

---

## 5. 在試算表中編輯 .backup 檔案

### 5.1 前置準備

**需要的工具：**
- Python 3.x（用於格式轉換腳本）
- Google Sheets 或 Microsoft Excel
- 文字編輯器（如 VS Code，用於檢查原始檔案）

**工作流程概觀：**

```
.backup (GZip) → 解壓縮 → 純文字 → Python 轉 CSV → 試算表編輯 → Python 轉回 → 壓縮 → .backup
```

### 5.2 解壓縮 .backup 檔案

Financisto 的 `.backup` 檔案通常以 GZip 壓縮。可用以下方式解壓：

**Linux/macOS：**
```bash
# 方法 1：使用 gunzip（需先複製，因為 gunzip 會刪除原檔）
cp my_backup.backup my_backup.backup.gz
gunzip my_backup.backup.gz
# 產生 my_backup.backup（已解壓的純文字）

# 方法 2：使用 Python
python3 -c "
import gzip, sys
with gzip.open('my_backup.backup', 'rb') as f:
    with open('my_backup.txt', 'wb') as out:
        out.write(f.read())
print('Done')
"
```

**Windows PowerShell：**
```powershell
# 使用 Python
python -c "import gzip; open('my_backup.txt','wb').write(gzip.open('my_backup.backup','rb').read())"
```

> **提示：** 如果 `gunzip` 報錯，檔案可能未壓縮。先用文字編輯器打開看看是否為純文字。

### 5.3 將 .backup 轉換為 CSV/試算表格式

以下 Python 腳本會將 `.backup` 檔案解析為每個表格一個 CSV 檔案：

```python
#!/usr/bin/env python3
"""backup_to_csv.py - 將 Financisto .backup 轉為 CSV 檔案"""

import csv
import gzip
import os
import sys
from collections import OrderedDict

def read_backup(filepath):
    """讀取 .backup 檔案，自動偵測是否為 GZip"""
    try:
        with gzip.open(filepath, 'rt', encoding='utf-8') as f:
            return f.read()
    except (gzip.BadGzipFile, OSError):
        with open(filepath, 'r', encoding='utf-8') as f:
            return f.read()

def parse_backup(content):
    """
    解析 .backup 內容，回傳：
    - header: dict（檔頭資訊）
    - tables: dict，key=表名，value=list of OrderedDict（每列資料）
    """
    header = {}
    tables = {}
    current_table = None
    current_record = None
    in_data = False

    for line in content.splitlines():
        if not in_data:
            if line == '#START':
                in_data = True
            elif ':' in line:
                key, _, value = line.partition(':')
                header[key] = value
            continue

        if line == '#END':
            break

        if line.startswith('$ENTITY:'):
            current_table = line[8:]  # len('$ENTITY:') == 8
            current_record = OrderedDict()
        elif line == '$$':
            if current_table and current_record is not None:
                if current_table not in tables:
                    tables[current_table] = []
                tables[current_table].append(current_record)
            current_table = None
            current_record = None
        elif current_record is not None and ':' in line:
            key, _, value = line.partition(':')
            current_record[key] = value

    return header, tables

def export_to_csv(tables, output_dir):
    """將每個表格匯出為 CSV 檔案"""
    os.makedirs(output_dir, exist_ok=True)

    for table_name, records in tables.items():
        if not records:
            continue

        # 收集所有欄位名稱（保持順序）
        all_columns = []
        seen = set()
        for record in records:
            for key in record.keys():
                if key not in seen:
                    all_columns.append(key)
                    seen.add(key)

        filepath = os.path.join(output_dir, f'{table_name}.csv')
        with open(filepath, 'w', newline='', encoding='utf-8-sig') as f:
            writer = csv.DictWriter(f, fieldnames=all_columns, extrasaction='ignore')
            writer.writeheader()
            for record in records:
                writer.writerow(record)

        print(f'  匯出: {filepath} ({len(records)} 筆)')

    # 儲存檔頭資訊
    return all_columns

def main():
    if len(sys.argv) < 2:
        print('用法: python3 backup_to_csv.py <backup_file> [output_dir]')
        sys.exit(1)

    backup_file = sys.argv[1]
    output_dir = sys.argv[2] if len(sys.argv) > 2 else 'backup_csv'

    print(f'讀取: {backup_file}')
    content = read_backup(backup_file)

    print('解析 .backup 檔案...')
    header, tables = parse_backup(content)

    print(f'\n檔頭資訊:')
    for k, v in header.items():
        print(f'  {k}: {v}')

    print(f'\n找到 {len(tables)} 個表格:')
    export_to_csv(tables, output_dir)

    # 儲存 header
    header_path = os.path.join(output_dir, '_header.txt')
    with open(header_path, 'w', encoding='utf-8') as f:
        for k, v in header.items():
            f.write(f'{k}:{v}\n')
    print(f'\n檔頭已存為: {header_path}')

    # 儲存表格順序
    order_path = os.path.join(output_dir, '_table_order.txt')
    with open(order_path, 'w', encoding='utf-8') as f:
        for table_name in tables.keys():
            f.write(f'{table_name}\n')
    print(f'表格順序已存為: {order_path}')

    print('\n完成！現在可以用 Google Sheets 或 Excel 開啟 CSV 檔案進行編輯。')

if __name__ == '__main__':
    main()
```

### 5.4 在試算表中編輯

#### 開啟 CSV 檔案

**Google Sheets：**
1. 前往 [Google Sheets](https://sheets.google.com)
2. 檔案 → 匯入 → 上傳 → 選擇 CSV 檔案
3. 選擇「取代目前的工作表」和「自動偵測」分隔符號
4. **重要：** 匯入時選擇「將文字轉為數字、日期…」設為**否**，避免長數字被截斷

**Excel：**
1. 開啟 Excel → 資料 → 從文字/CSV
2. 選擇 CSV 檔案，編碼選 UTF-8
3. **重要：** 將所有欄位設為「文字」格式，避免 ID 和時間戳被自動轉換

#### 編輯注意事項

> ⚠️ **關鍵警告：** 試算表軟體可能會自動將大數字（如 Unix 毫秒時間戳 `1709481600000`）轉為科學記號或截斷精度。務必將相關欄位設為**純文字格式**。

**可安全編輯的操作：**

| 操作 | 對應表格 | 注意事項 |
|------|----------|----------|
| 修改帳本名稱 | `account.csv` 的 `title` 欄 | — |
| 修改交易備註 | `transactions.csv` 的 `note` 欄 | — |
| 修改交易分類 | `transactions.csv` 的 `category_id` 欄 | 確認 ID 存在於 `category.csv` |
| 修改交易金額 | `transactions.csv` 的 `from_amount` / `to_amount` 欄 | 金額 ×100 的整數 |
| 新增交易 | `transactions.csv` 新增行 | `_id` 必須唯一且大於所有現有 ID |
| 修改幣別資訊 | `currency.csv` | — |
| 刪除交易 | `transactions.csv` 刪除行 | 若有子交易需一併刪除 |

**新增交易的必填欄位：**

| 欄位 | 必填 | 範例值 | 說明 |
|------|:----:|--------|------|
| `_id` | ✓ | `1001` | 唯一正整數，建議從最大現有 ID+1 開始 |
| `from_account_id` | ✓ | `1` | 來源帳本 ID |
| `to_account_id` | ✓ | `0` | 一般交易填 `0`，轉帳填目標帳本 ID |
| `category_id` | ✓ | `5` | 分類 ID |
| `from_amount` | ✓ | `-15000` | 金額×100（負=支出，正=收入） |
| `to_amount` | ✓ | `0` | 轉帳時填目標金額，否則 `0` |
| `datetime` | ✓ | `1709481600000` | Unix 毫秒時間戳 |
| `is_template` | ✓ | `0` | `0`=普通交易 |
| `status` | ✓ | `UR` | 通常填 `UR` |
| `parent_id` |  | `0` | 分拆子交易填父 ID |

**Unix 毫秒時間戳轉換（Google Sheets 公式）：**

```
=（日期儲存格 - DATE(1970,1,1)）* 86400000
```

例如：2024-03-03 的時間戳

```
=(DATE(2024,3,3) - DATE(1970,1,1)) * 86400000
結果: 1709424000000
```

**反向轉換（時間戳→日期）：**

```
=DATE(1970,1,1) + 時間戳儲存格 / 86400000
```

### 5.5 將試算表轉回 .backup 格式

編輯完成後，將 CSV 儲存回來並用以下 Python 腳本轉回 `.backup` 格式：

```python
#!/usr/bin/env python3
"""csv_to_backup.py - 將 CSV 檔案轉回 Financisto .backup 格式"""

import csv
import gzip
import os
import sys

def read_header(input_dir):
    """讀取檔頭資訊"""
    header = {}
    header_path = os.path.join(input_dir, '_header.txt')
    with open(header_path, 'r', encoding='utf-8') as f:
        for line in f:
            line = line.strip()
            if ':' in line:
                key, _, value = line.partition(':')
                header[key] = value
    return header

def read_table_order(input_dir):
    """讀取表格順序"""
    order_path = os.path.join(input_dir, '_table_order.txt')
    with open(order_path, 'r', encoding='utf-8') as f:
        return [line.strip() for line in f if line.strip()]

def read_csv_table(filepath):
    """讀取單一 CSV 表格"""
    records = []
    with open(filepath, 'r', encoding='utf-8-sig') as f:
        reader = csv.DictReader(f)
        for row in reader:
            # 過濾空值
            record = {k: v for k, v in row.items() if v is not None and v != ''}
            if record:
                records.append(record)
    return records

def write_backup(header, table_order, tables, output_path, use_gzip=True):
    """寫入 .backup 檔案"""
    lines = []

    # 寫入 header
    for key in ['PACKAGE', 'VERSION_CODE', 'VERSION_NAME', 'DATABASE_VERSION']:
        if key in header:
            lines.append(f'{key}:{header[key]}')
    lines.append('#START')

    # 寫入每個表格
    for table_name in table_order:
        if table_name not in tables:
            continue
        for record in tables[table_name]:
            lines.append(f'$ENTITY:{table_name}')
            for key, value in record.items():
                if value is not None and value != '':
                    # 移除換行符
                    value = str(value).replace('\n', ' ')
                    lines.append(f'{key}:{value}')
            lines.append('$$')

    lines.append('#END')

    content = '\n'.join(lines)

    if use_gzip:
        with gzip.open(output_path, 'wt', encoding='utf-8') as f:
            f.write(content)
    else:
        with open(output_path, 'w', encoding='utf-8') as f:
            f.write(content)

    print(f'已寫入: {output_path} (GZip: {use_gzip})')

def main():
    if len(sys.argv) < 2:
        print('用法: python3 csv_to_backup.py <csv_dir> [output_file] [--no-gzip]')
        sys.exit(1)

    input_dir = sys.argv[1]
    output_file = sys.argv[2] if len(sys.argv) > 2 and not sys.argv[2].startswith('-') else 'restored.backup'
    use_gzip = '--no-gzip' not in sys.argv

    print(f'讀取 CSV 目錄: {input_dir}')

    header = read_header(input_dir)
    table_order = read_table_order(input_dir)

    tables = {}
    for table_name in table_order:
        csv_path = os.path.join(input_dir, f'{table_name}.csv')
        if os.path.exists(csv_path):
            records = read_csv_table(csv_path)
            tables[table_name] = records
            print(f'  讀取: {table_name} ({len(records)} 筆)')
        else:
            print(f'  跳過: {table_name} (CSV 不存在)')

    write_backup(header, table_order, tables, output_file, use_gzip)
    print('\n完成！可將檔案匯入 Financisto。')

if __name__ == '__main__':
    main()
```

### 5.6 重新壓縮與匯入

**壓縮為 GZip（若使用 `--no-gzip` 參數）：**

```bash
# Python 腳本已自動壓縮，若需手動壓縮：
python3 -c "
import gzip
with open('restored.backup', 'rb') as f:
    with gzip.open('restored.backup.gz', 'wb') as gz:
        gz.write(f.read())
"
# 將 restored.backup.gz 改名為 restored.backup
```

**匯入到 Financisto：**

1. 將 `.backup` 檔案複製到裝置的備份資料夾中
2. 開啟 Financisto → 選單 → 備份（Backup）
3. 選擇「還原備份」（Restore Backup）
4. 選取編輯後的 `.backup` 檔案
5. 確認還原

### 5.7 完整 Python 工具腳本

為方便使用，以下是一個整合的命令列工具：

```python
#!/usr/bin/env python3
"""
financisto_backup_tool.py - Financisto .backup 檔案的試算表編輯工具

用法:
    # 匯出為 CSV（給試算表編輯）
    python3 financisto_backup_tool.py export my_backup.backup csv_output/

    # 從 CSV 還原為 .backup
    python3 financisto_backup_tool.py import csv_output/ edited_backup.backup

    # 從 CSV 還原為未壓縮 .backup（方便檢查）
    python3 financisto_backup_tool.py import csv_output/ edited_backup.backup --no-gzip

    # 顯示 .backup 摘要
    python3 financisto_backup_tool.py info my_backup.backup
"""

import csv
import gzip
import os
import sys
from collections import OrderedDict


def read_backup_content(filepath):
    """讀取 .backup 檔案內容，自動偵測 GZip"""
    try:
        with gzip.open(filepath, 'rt', encoding='utf-8') as f:
            return f.read()
    except (gzip.BadGzipFile, OSError):
        with open(filepath, 'r', encoding='utf-8') as f:
            return f.read()


def parse_backup(content):
    """解析 .backup 內容"""
    header = OrderedDict()
    tables = OrderedDict()
    current_table = None
    current_record = None
    in_data = False

    for line in content.splitlines():
        if not in_data:
            if line == '#START':
                in_data = True
            elif ':' in line:
                key, _, value = line.partition(':')
                header[key] = value
            continue

        if line == '#END':
            break

        if line.startswith('$ENTITY:'):
            current_table = line[8:]
            current_record = OrderedDict()
        elif line == '$$':
            if current_table and current_record is not None:
                if current_table not in tables:
                    tables[current_table] = []
                tables[current_table].append(current_record)
            current_table = None
            current_record = None
        elif current_record is not None and ':' in line:
            key, _, value = line.partition(':')
            current_record[key] = value

    return header, tables


def export_csv(tables, output_dir):
    """匯出為 CSV"""
    os.makedirs(output_dir, exist_ok=True)

    for table_name, records in tables.items():
        if not records:
            continue

        all_columns = []
        seen = set()
        for record in records:
            for key in record.keys():
                if key not in seen:
                    all_columns.append(key)
                    seen.add(key)

        filepath = os.path.join(output_dir, f'{table_name}.csv')
        with open(filepath, 'w', newline='', encoding='utf-8-sig') as f:
            writer = csv.DictWriter(f, fieldnames=all_columns, extrasaction='ignore')
            writer.writeheader()
            for record in records:
                writer.writerow(record)

        print(f'  {table_name}.csv ({len(records)} 筆)')


def cmd_export(args):
    """匯出命令"""
    if len(args) < 2:
        print('用法: export <backup_file> <output_dir>')
        return

    backup_file, output_dir = args[0], args[1]

    print(f'讀取: {backup_file}')
    content = read_backup_content(backup_file)
    header, tables = parse_backup(content)

    print(f'\n檔頭:')
    for k, v in header.items():
        print(f'  {k}: {v}')

    print(f'\n匯出 {len(tables)} 個表格:')
    export_csv(tables, output_dir)

    # 儲存 metadata
    with open(os.path.join(output_dir, '_header.txt'), 'w', encoding='utf-8') as f:
        for k, v in header.items():
            f.write(f'{k}:{v}\n')

    with open(os.path.join(output_dir, '_table_order.txt'), 'w', encoding='utf-8') as f:
        for name in tables.keys():
            f.write(f'{name}\n')

    print(f'\n✅ 完成！CSV 檔案在 {output_dir}/ 目錄中。')


def cmd_import(args):
    """匯入命令"""
    if len(args) < 2:
        print('用法: import <csv_dir> <output_backup> [--no-gzip]')
        return

    input_dir, output_file = args[0], args[1]
    use_gzip = '--no-gzip' not in args

    # 讀取 metadata
    header = OrderedDict()
    with open(os.path.join(input_dir, '_header.txt'), 'r', encoding='utf-8') as f:
        for line in f:
            line = line.strip()
            if ':' in line:
                k, _, v = line.partition(':')
                header[k] = v

    with open(os.path.join(input_dir, '_table_order.txt'), 'r', encoding='utf-8') as f:
        table_order = [l.strip() for l in f if l.strip()]

    # 讀取 CSV
    tables = OrderedDict()
    for name in table_order:
        csv_path = os.path.join(input_dir, f'{name}.csv')
        if not os.path.exists(csv_path):
            continue
        records = []
        with open(csv_path, 'r', encoding='utf-8-sig') as f:
            for row in csv.DictReader(f):
                record = OrderedDict((k, v) for k, v in row.items() if v)
                if record:
                    records.append(record)
        tables[name] = records
        print(f'  {name}: {len(records)} 筆')

    # 寫入 .backup
    lines = []
    for key in ['PACKAGE', 'VERSION_CODE', 'VERSION_NAME', 'DATABASE_VERSION']:
        if key in header:
            lines.append(f'{key}:{header[key]}')
    lines.append('#START')

    for table_name in table_order:
        if table_name not in tables:
            continue
        for record in tables[table_name]:
            lines.append(f'$ENTITY:{table_name}')
            for k, v in record.items():
                if v:
                    lines.append(f'{k}:{str(v).replace(chr(10), " ")}')
            lines.append('$$')

    lines.append('#END')
    text = '\n'.join(lines)

    if use_gzip:
        with gzip.open(output_file, 'wt', encoding='utf-8') as f:
            f.write(text)
    else:
        with open(output_file, 'w', encoding='utf-8') as f:
            f.write(text)

    print(f'\n✅ 已產生: {output_file} (GZip: {use_gzip})')


def cmd_info(args):
    """顯示摘要"""
    if not args:
        print('用法: info <backup_file>')
        return

    content = read_backup_content(args[0])
    header, tables = parse_backup(content)

    print('=== Financisto Backup 摘要 ===\n')
    for k, v in header.items():
        print(f'  {k}: {v}')

    print(f'\n表格統計:')
    total = 0
    for name, records in tables.items():
        print(f'  {name:30s} {len(records):>6} 筆')
        total += len(records)
    print(f'  {"合計":30s} {total:>6} 筆')


def main():
    if len(sys.argv) < 2:
        print(__doc__)
        sys.exit(1)

    cmd = sys.argv[1]
    args = sys.argv[2:]

    if cmd == 'export':
        cmd_export(args)
    elif cmd == 'import':
        cmd_import(args)
    elif cmd == 'info':
        cmd_info(args)
    else:
        print(f'未知命令: {cmd}')
        print('可用命令: export, import, info')


if __name__ == '__main__':
    main()
```

---

## 6. 注意事項與常見問題

### ⚠️ 重要注意事項

1. **`_id` 必須唯一且為正整數**
   - 每個表格中的 `_id` 不能重複
   - 不可使用 `0` 或負數（`_id <= 0` 在還原時會被跳過）
   - 新增記錄時，建議使用比現有最大 `_id` 更大的值

2. **金額是整數（乘以 100）**
   - `150.00` 元 → 存為 `15000`
   - `-50.50` 元 → 存為 `-5050`
   - 支出為負數，收入為正數

3. **時間戳是 Unix 毫秒**
   - 2024-03-03 00:00:00 UTC → `1709424000000`
   - 不要使用秒級時間戳，必須乘以 1000

4. **外鍵關聯必須正確**
   - `transactions.from_account_id` 必須對應存在的 `account._id`
   - `transactions.category_id` 必須對應存在的 `category._id`
   - `account.currency_id` 必須對應存在的 `currency._id`

5. **`account.total_amount` 會自動重建**
   - 還原後 `IntegrityFix` 會重新計算所有帳本餘額
   - 所以備份中的 `total_amount` 值不需要手動維護正確
   - 但 `transactions.from_amount` / `to_amount` **必須正確**

6. **試算表中大數字精度問題**
   - Excel 和 Google Sheets 對超過 15 位的數字會失去精度
   - Unix 毫秒時間戳為 13 位，通常安全
   - 建議所有數字欄位都設為「純文字」格式

7. **換行符會被移除**
   - `.backup` 匯出時會將值中的 `\n` 替換為空格
   - 如果備註中需要換行，還原後會變成空格

8. **`sort_order` 的特殊行為**
   - 大多數表格的 `sort_order` 在匯出時**不會被寫入**（`account` 除外）
   - 還原時會自動按讀取順序分配 `sort_order`
   - 如果需要特定排序，可以調整 CSV 中的行順序

9. **分類的巢狀集合模型**
   - `category` 表使用 `left` 和 `right` 值來表示階層關係
   - 修改分類時要小心維護 `left`/`right` 的一致性
   - 建議不要在試算表中手動修改分類結構

### 常見編輯場景

#### 場景 1：批次修改交易分類

```
1. 匯出 .backup 為 CSV
2. 開啟 transactions.csv 和 category.csv
3. 在 category.csv 中找到目標分類的 _id
4. 在 transactions.csv 中批次修改 category_id
5. 轉回 .backup 並匯入
```

#### 場景 2：合併兩個帳本

```
1. 匯出 .backup 為 CSV
2. 在 transactions.csv 中，將帳本 B 的所有交易的
   from_account_id 改為帳本 A 的 _id
3. 在 account.csv 中刪除帳本 B 的列
4. 轉回 .backup 並匯入（餘額會自動重算）
```

#### 場景 3：批次新增交易

```
1. 匯出 .backup 為 CSV
2. 在 transactions.csv 中新增列
3. 確認 _id 唯一，from_account_id 正確
4. 金額乘以 100，支出為負數
5. datetime 填 Unix 毫秒時間戳
6. 轉回 .backup 並匯入
```

#### 場景 4：修改帳本幣別

```
1. 匯出 .backup 為 CSV
2. 確認 currency.csv 中有目標幣別
3. 在 account.csv 中修改目標帳本的 currency_id
4. 如需調整金額精度，同步修改相關交易的金額
5. 轉回 .backup 並匯入
```

---

> **免責聲明：** 手動編輯備份檔案有資料損壞的風險。操作前請務必保留原始 `.backup` 的副本。建議先在測試環境中驗證修改過的備份能正確還原。
