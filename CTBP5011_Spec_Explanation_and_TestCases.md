# CTBP5011 Update Transmission Status — 功能規格書解讀 + 測試用例 (Test Cases)

> 對應文件：`Functional Specification – CTBP5011 Update Transmission Status v1.0 (working, 2026-06-17)`，全文 **3 頁**，含 **1 張 Program Flow 圖**（Claude 已成功讀取圖內容）。
> Part A = 逐節解讀（中文）；Part B = 英文 Test Cases（Test Case ID / Test Scenario / Test Data / Expected Result / Remark；Remark 為出處頁碼/圖）。
> **方向（Direction）**：**Outbound**——資料封包經 SFTP 傳往 OECD 後，更新傳輸狀態。被 CTBP5002 / CTBP5003 於傳輸後呼叫。

---

## 0. 定位
CTBP5011 在 CTBP5002（簽章加密資料）與 CTBP5003（簽章加密狀態訊息）**SFTP 傳輸之後**被呼叫，讀 SFTP transmission log 更新 `CTS_TX_REC`（傳輸記錄）。

### 術語速查
| 詞 | 義 |
|---|---|
| **CTS_TX_REC** | Transmission Record（傳輸記錄表） |
| **TX_STS** | 傳輸狀態：`R`=Ready、`T`=Success、`N`=Pending、`F/U`=Failure |
| **TX_ID / CTS_RTN_CODE** | Transmission ID / CTS 回傳碼 |
| **TX_DATE / TX_YR / TX_MTH** | 傳輸日期時間 / 年 / 月 |

---

# Part A — 逐節內容解讀

## A1. Change History（p.1）
單筆：`2026-04-09 Initial version`。v1.0 初版。

## A2. Batch Job Properties（p.1）
- **Batch Job ID**：`CTBP5011`
- **Frequency**：**（空白）**——無固定排程（由 CTBP5002/5003 傳輸後觸發）
- **Ad-hoc Run**：`Supported`
- **Ad-hoc Parameters**：**（空白，無參數）**

## A3. Description（p.1）
本程式在資料封包傳往 OECD server 後，**更新傳輸狀態**。

## A4. Program Flow（p.2，Program Flow 圖）★ Claude 已讀取圖內容
1. Start
2. 以 **SFTP transmission log** 為來源，更新 `CTS_TX_REC`（Transmission Record）
3. Update fields：`CTS_TX_REC.TX_STS`、`CTS_TX_REC.TX_DATE`、`CTS_TX_REC.TX_YR`、`CTS_TX_REC.TX_MTH`
4. **Running Environment = 'DEV'/'UAT'?**
   - **Yes** → **Generate**（模擬產生）Transmission ID & CTS Return Code，更新 `CTS_TX_REC.TX_ID`、`CTS_TX_REC.CTS_RTN_CODE`
   - **No**（正式環境）→ 透過 **RESTful API** 取得 Transmission ID & CTS Return Code，更新同兩欄
5. End

## A5. Program Logic（p.2–3）
GIR 檔傳輸後的 transmission log 範例（逗號分隔）：
`HK_GIR_20220831T143730554Z.zip,GIR,T,20220831143732`
以逗號拆解取得四項並對應欄位：
- **File Name** → `CTS_DATA_PACKET.ZIP_FILENAME`
- **File Type** → `GIR` / `GIRMessageStatus`
- **Transmission Status** → `CTS_TX_REC.TX_STS`
- **Transmission Date and Time (YYYYMMDDHHMMSS)** → `CTS_TX_REC.TX_DATE`、`CTS_TX_REC.TX_YR`、`CTS_TX_REC.TX_MTH`

## A6. Database Records（p.3）
| 動作 | 表 | 欄位 |
|---|---|---|
| Update | `CTS_TX_REC` | `TX_STS`、`CTS_RTN_CODE`、`TX_DATE`、`TX_YR`、`TX_MTH`（TX_STS：R=Ready/T=Success/N=Pending/F,U=Failure） |

## A7. 待釐清處
1. **Frequency 空白**：由 CTBP5002/5003 傳輸後觸發，建議確認觸發串接方式。
2. **DEV/UAT vs 正式**：DEV/UAT 為**模擬產生** TX_ID/CTS_RTN_CODE，正式環境經 RESTful API 取得；測試須分環境驗證。
3. **TX_ID vs TX_ID 欄名**：圖寫 `CTS_TX_REC.TX_ID`，A6 資料表未列 `TX_ID`（只列 CTS_RTN_CODE 等），建議確認 `TX_ID` 是否亦於 CTS_TX_REC 更新。

---

# Part B — Test Cases (English)

> Format: **Test Case ID | Test Scenario | Test Data | Expected Result | Remark**. Direction = **Outbound** for all. "p.N" = page N; "Flow" = Program Flow diagram (p.2).

## B1. Batch Properties & Trigger

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-11-001 | Ad-hoc run supported (no parameter) | Ad-hoc trigger, no parameter | Program runs and updates transmission status | p.1 |
| TC-11-002 | Triggered after SFTP transmission | CTBP5002/5003 completes SFTP | CTBP5011 runs to update CTS_TX_REC (confirm trigger) | p.1 (A7-1) |

## B2. Transmission Log Parsing

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-11-003 | Parse comma-delimited GIR log line | `HK_GIR_20220831T143730554Z.zip,GIR,T,20220831143732` | Split into File Name, File Type='GIR', TX Status='T', DateTime='20220831143732' | p.2–3 |
| TC-11-004 | Map File Name to ZIP_FILENAME | Parsed file name | Mapped to CTS_DATA_PACKET.ZIP_FILENAME | p.3 |
| TC-11-005 | Map status to TX_STS | Parsed status='T' | CTS_TX_REC.TX_STS='T' | p.3 |
| TC-11-006 | Map datetime to TX_DATE/TX_YR/TX_MTH | Parsed datetime='20220831143732' | CTS_TX_REC.TX_DATE/TX_YR/TX_MTH set from the timestamp | p.3 |
| TC-11-007 | Recognize GIRMessageStatus file type | A GIRMessageStatus log line | File Type recognized as GIRMessageStatus | p.3 |
| TC-11-008 | Multiple log lines processed | 3 log lines | Each line parsed and its CTS_TX_REC updated | p.2–3 |

## B3. Field Update (CTS_TX_REC)

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-11-009 | Update transmission status fields | A matching CTS_TX_REC row | TX_STS, TX_DATE, TX_YR, TX_MTH updated | p.2, p.3, Flow |
| TC-11-010 | Status success | Log status='T' | CTS_TX_REC.TX_STS='T' (Success) | p.3, Flow |
| TC-11-011 | Status failure | Log status='F' or 'U' | CTS_TX_REC.TX_STS reflects failure (F/U) | p.3 |
| TC-11-012 | Status pending | Log status='N' | CTS_TX_REC.TX_STS='N' (Pending) | p.3 |

## B4. Environment Branch (DEV/UAT vs Production)

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-11-013 | DEV/UAT generates TX_ID & return code | Running Environment='DEV' or 'UAT' | Transmission ID & CTS Return Code generated (simulated); CTS_TX_REC.TX_ID & CTS_RTN_CODE updated | p.2, Flow |
| TC-11-014 | Production fetches via RESTful API | Running Environment not DEV/UAT | TX_ID & CTS_RTN_CODE obtained via RESTful API; CTS_TX_REC.TX_ID & CTS_RTN_CODE updated | p.2, Flow |
| TC-11-015 | Both branches update the same two fields | Either environment | CTS_TX_REC.TX_ID and CTS_TX_REC.CTS_RTN_CODE are populated regardless of branch | p.2, Flow (A7-3) |
| TC-11-016 | RESTful API failure handling (production) | API call fails | Handled gracefully (confirm behaviour with BA) | p.2, Flow |

## B5. Edge Cases

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-11-017 | Empty transmission log | No log lines | Program completes; no CTS_TX_REC updated | p.2–3 |
| TC-11-018 | Log line for unknown ZIP filename | File name not matching any CTS_TX_REC | Handled gracefully; no incorrect update | p.3 |
| TC-11-019 | Malformed log line (wrong field count) | Line with missing comma fields | Handled as error; not applied blindly | p.2–3 |

---

### 統計
- 章節：Change History / Batch Job Properties / Description / Program Flow（圖，已判讀）/ Program Logic / Database Records 全數覆蓋。
- 測試用例：**19 條**，每條標註頁碼/圖。
- 待釐清：見 A7（觸發方式、DEV/UAT vs API、TX_ID 欄位）。
