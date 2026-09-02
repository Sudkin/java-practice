# CTBP5003 Sign and Encrypt Status Message — 功能規格書解讀 + 測試用例 (Test Cases)

> 對應文件：`Functional Specification – CTBP5003 Sign and Encrypt Status Message v1.0 (working, 2026-06-17)`，全文 **3 頁**，含 **1 張 Program Flow 圖**（雙泳道，Claude 已成功讀取圖內容）。
> Part A = 逐節解讀（中文）；Part B = 英文 Test Cases（Test Case ID / Test Scenario / Test Data / Expected Result / Remark；Remark 為出處頁碼/圖）。
> **方向（Direction）**：本程式為 **Outbound**（把 HK 產生的**狀態訊息**簽章加密後回傳給夥伴）。

---

## 0. 定位
CTBP5003 與 CTBP5002 是**同一支「簽章加密」機制的兩個變體**：
- CTBP5002 處理 **GIR 資料檔**（HK 主動交換的資料）；
- CTBP5003 處理 **狀態訊息（GIRMessageStatus）**——即 HK 收到夥伴傳來的資料後、由 **CTBP5005（Receive Data Packet）** 產生的「回覆狀態」，本程式把它簽章加密後經 SFTP 回傳夥伴。
上游＝CTBP5005（產生狀態訊息）；下游＝SFTP 傳輸 / CTBP5011。

### 術語速查
| 詞 | 義 |
|---|---|
| **GIRMessageStatus** | HK 回覆夥伴的狀態訊息類型 |
| **GIRMessageStatus MessageRefId** | 狀態訊息唯一參考編號（見 Text Format），亦為 Ad-hoc 參數 |
| **SGD_STS** | `P`=待簽章加密、`S`=已簽章 |
| **TX_STS** | `R`=Ready、`T`=Success、`N`=Pending、`F/U`=Failure、`P`=Pending for transmission（見 A9 衝突） |

---

# Part A — 逐節內容解讀

## A1. Change History（p.1）
單筆：`2026-04-09 Initial version`。v1.0 初版。

## A2. Batch Job Properties（p.1）
- **Batch Job ID**：`CTBP5003`
- **Frequency**：`Monday–Friday at 21:00`
- **Ad-hoc Run**：`Supported`
- **Ad-hoc Parameters**：位置 #1，格式 `Characters`，內容 = **GIRMessageStatus MessageRefId**

## A3. Description（p.1）
本程式將 **CTBP5005 – Receive Data Packet Received from Pillar Two Partners 產生** 的狀態訊息 XML 檔進行簽章與加密。

## A4. Program Flow（p.2，Program Flow 圖）★ Claude 已讀取圖內容
> **注意**：圖標題誤植為「CTBP5002 – Sign & Encrypt Status Message」，實為 **CTBP5003**。

雙泳道流程：

**泳道 1 — CTBP5003 Sign & Encrypt Status Message**
1. Start
2. Retrieve all **CTS_STS_MSG** records where **SGD_STS='P'**（待簽章加密）
3. Locate the related **GIRMessageStatus** payload & metadata XML files in **`\data\in\status_out\pending`**
4. **Perform sign and encrypt**
5. Move finished payload & metadata XML files to **`\data\in\status_out\processed`**
6. Move the generated ZIP file to **`\data\cts\Outbox\<receiving country>\`**
7. **Perform Database Update**：`CTS_STS_MSG.SGD_STS='S'`；`CTS_TX_REC.TX_STS='P'`（Pending for transmission）★見 A9

**泳道 2 — Transfer Data Packets to OECD via SFTP**
1. SFTP the files in `\data\cts\Outbox\<receiving country>` to OECD server，輸出 transmission log
2. **CTBP5011 Update Transmission Status**
3. Move transmitted files to **`\data\cts\Outbox_sent\<receiving country>`**
4. End

## A5. Text Format — GIRMessageStatus MessageRefId（p.2）
`'Status' + 'HK' + <Reporting Year> + <Receiving Country> + '-' + <yyyyMMddHHmmss>`
例（RPT_YR=2025、RX=US、時間 20260617103000）：`StatusHK2025US-20260617103000`。

## A6. File Format（p.2）
| 名稱 | 格式 |
|---|---|
| Payload XML File | `<GIRMessageStatus MessageRefId>.xml` |
| Metadata XML File | `<GIRMessageStatus MessageRefId>.metadata.xml` |

## A7. Folder Path（p.3）
| 路徑 | 說明 |
|---|---|
| `\data\in\status_out\pending\` | 待簽章加密的狀態訊息 payload/metadata |
| `\data\in\status_out\processed\` | 已簽章加密 |
| `\data\cts\Outbox\<ReceivingCountry>\` | 已加密、待傳輸 |
| `\data\cts\Outbox_sent\<ReceivingCountry>\` | 已傳輸 |

> 注意路徑在 **`\data\in\status_out\`** 下（因狀態訊息源於「收到夥伴資料」的處理），與 CTBP5002 的 `\data\out\` 不同。

## A8. Database Records（p.3）
| 表 | 動作 | 內容 |
|---|---|---|
| `CTS_STS_MSG` | Update | `SGD_STS='S'`（Signed） |
| `CTS_TX_REC` | Insert | `TX_STS='R'`（Ready for Transmission）★見 A9：圖寫 `P` |
| `CTS_DATA_PACKET` | Update | `CTS_DATA_PACKET.ZIP_FILENAME={Outgoing GIRMessageStatus Filename}` |

### CTBP5011 — Update Transmission Status（p.3）
DEV/UAT 手動產生 transmission log。範例：
`HK_GIRMessageStatus_20220831T143730554Z.zip,GIRMessageStatus,T,20220831143732`
以逗號拆解取得：**File Name**、**File Type（GIRMessageStatus）**、**Transmission Status（T/F,U/N）**、**Transmission Date and Time（YYYYMMDDHHMMSS）**。

## A9. 待釐清處
1. **★ TX_STS 值衝突**：**圖**寫 `CTS_TX_REC.TX_STS='P'`（Pending for transmission）且動作為「Database Update」；**內文表**寫 `TX_STS='R'`（Ready）且動作為「Insert」。二者對「值」與「動作」皆不一致（CTBP5002 兩處均為 `R`）。**須確認**狀態訊息的正確初始 TX_STS 與動作。
2. **圖標題筆誤**：Program Flow 圖標題寫「CTBP5002」，應為「CTBP5003」。
3. **`CTX_` 拼寫**：內文寫 `CTX_DATA_PACKET.ZIP_FILENAME`，應為 `CTS_DATA_PACKET`。
4. **泳道 2 標題筆誤**：`... vis SFTP`（`vis`→`via`）。

---

# Part B — Test Cases (English)

> Format: **Test Case ID | Test Scenario | Test Data | Expected Result | Remark**. Direction = **Outbound** for all. "p.N" = page N; "Flow" = Program Flow diagram (p.2).

## B1. Batch Properties & Trigger

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-03-001 | Scheduled run at configured time | Weekday 21:00 trigger | CTBP5003 runs on Mon–Fri 21:00 schedule | p.1 |
| TC-03-002 | Ad-hoc run with parameter | Param #1 = a valid GIRMessageStatus MessageRefId | Program runs for only that status message | p.1 |
| TC-03-003 | Ad-hoc run with non-existent MessageRefId | Param #1 = unknown MessageRefId | No matching record processed; handled gracefully | p.1 |

## B2. Record Selection

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-03-004 | Select status messages pending signing | CTS_STS_MSG with SGD_STS='P' | Record selected for signing & encryption | p.2, Flow |
| TC-03-005 | Exclude already-signed status messages | CTS_STS_MSG with SGD_STS='S' | Record NOT selected | p.2, Flow |
| TC-03-006 | No qualifying records | No CTS_STS_MSG with SGD_STS='P' | Program completes; nothing signed; no DB change | Flow |

## B3. MessageRefId Construction (Text Format)

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-03-007 | Build GIRMessageStatus MessageRefId | RPT_YR=2025, RX_CTRY=US, time=20260617103000 | MessageRefId = `StatusHK2025US-20260617103000` | p.2 |
| TC-03-008 | MessageRefId components order | Different RX_CTRY (e.g. JP), year 2025 | `Status`+`HK`+`2025`+`JP`+`-`+`<yyyyMMddHHmmss>` in exact order | p.2 |

## B4. Sign, Encrypt & File Movement

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-03-009 | Locate payload & metadata in pending folder | Selected MessageRefId with files in `\data\in\status_out\pending` | Both `<MessageRefId>.xml` and `.metadata.xml` located | p.2–3, Flow |
| TC-03-010 | Perform sign and encrypt | A located payload/metadata pair | Files signed and encrypted successfully | p.2, Flow |
| TC-03-011 | Move finished XML to processed folder | After sign & encrypt | Files moved to `\data\in\status_out\processed` | p.2–3, Flow |
| TC-03-012 | Move ZIP to Outbox by receiving country | ReceivingCountry='US' | ZIP moved to `\data\cts\Outbox\US\` | p.2–3, Flow |
| TC-03-013 | Missing payload/metadata in pending | Record selected but file absent | Handled as error; not moved to processed | p.2, Flow |

## B5. Database Records

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-03-014 | CTS_STS_MSG sign status updated | Status message signed | CTS_STS_MSG.SGD_STS='S' | p.3, Flow |
| TC-03-015 | CTS_TX_REC transmission record created | After packaging | CTS_TX_REC created with TX_STS — **confirm 'R' (text) vs 'P' (image)** | p.3 (A9-1) |
| TC-03-016 | CTS_DATA_PACKET ZIP filename updated | Signed ZIP produced | CTS_DATA_PACKET.ZIP_FILENAME = {outgoing GIRMessageStatus filename} | p.3 |

## B6. Transmission (SFTP + CTBP5011)

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-03-017 | SFTP transmit from Outbox to OECD | ZIP in `\data\cts\Outbox\US\` | Files SFTP-transmitted; transmission log output | p.2, Flow |
| TC-03-018 | CTBP5011 updates transmission status | After SFTP | CTBP5011 Update Transmission Status invoked | p.2–3, Flow |
| TC-03-019 | Move transmitted files to Outbox_sent | Transmission success | Files moved to `\data\cts\Outbox_sent\US\` | p.2, Flow |

## B7. Transmission Log Parsing (DEV/UAT)

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-03-020 | Parse comma-delimited status log | `HK_GIRMessageStatus_20220831T143730554Z.zip,GIRMessageStatus,T,20220831143732` | Split into: File Name, File Type='GIRMessageStatus', TX Status='T', DateTime='20220831143732' | p.3 |
| TC-03-021 | Interpret transmission status codes | Status = T / F / U / N | T=Success; F,U=Failure; N=Pending | p.3 |
| TC-03-022 | File type is GIRMessageStatus (not GIR) | Parsed status log line | File Type recognized as GIRMessageStatus | p.3 |

## B8. File & Folder Naming

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-03-023 | Payload file naming | MessageRefId=StatusHK2025US-20260617103000 | Payload = `StatusHK2025US-20260617103000.xml` | p.2 |
| TC-03-024 | Metadata file naming | Same MessageRefId | Metadata = `StatusHK2025US-20260617103000.metadata.xml` | p.2 |
| TC-03-025 | Status-out folder lifecycle | Same file at each stage | `\data\in\status_out\pending` → `\processed` → `\data\cts\Outbox\<country>` → `Outbox_sent\<country>` | p.2–3 |

## B9. Edge / Discrepancy Cases

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-03-026 | Sign/encrypt failure handling | Encryption fails (missing cert) | Not marked Signed; not moved to Outbox; error surfaced | p.2, Flow |
| TC-03-027 | TX_STS value/action reconciliation | CTS_TX_REC after signing | Determine authoritative value ('R' vs 'P') and action (Insert vs Update) | p.3, Flow (A9-1) |
| TC-03-028 | Diagram title mislabel does not affect logic | Status message run | Program behaves as CTBP5003 despite diagram title 'CTBP5002' | p.2 (A9-2) |

---

### 統計
- 章節：Change History / Batch Job Properties / Description / Program Flow（圖，已判讀）/ Text Format / File Format / Folder Path / Database Records / CTBP5011 全數覆蓋。
- 測試用例：**28 條**，每條標註頁碼/圖。
- 待釐清：見 A9（**A9-1 TX_STS 值/動作衝突** 為最高優先）。
