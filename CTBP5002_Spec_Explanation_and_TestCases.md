# CTBP5002 Sign and Encrypt Data File — 功能規格書解讀 + 測試用例 (Test Cases)

> 對應文件：`Functional Specification – CTBP5002 Sign and Encrypt Data File v1.0 (working, 2026-06-17)`，全文 **4 頁**，含 **1 張 Program Flow 圖**（雙泳道，Claude 已成功讀取圖內容）。
> Part A = 逐節解讀（中文，保留英文技術詞）；Part B = 英文 Test Cases（欄位：Test Case ID / Test Scenario / Test Data / Expected Result / Remark；Remark 為出處頁碼/圖）。
> **方向（Direction）**：本程式為 **Outbound**（把 HK 已核准的 GIR 資料檔簽章加密後傳往夥伴）。

---

## 0. 定位（在整條流程中的位置）
CTBP5002 是 CTBP5001 產出流程的**下游第 4 步**：把「CTBP5001 Data Extraction 產生、且 CTAP5002 已核准」的 GIR 資料 XML 檔進行**簽章與加密**，打包成 ZIP，放入 CTS Outbox，再由 SFTP 傳往 OECD，並由 CTBP5011 更新傳輸狀態。上游＝CTAP5002（核准）；下游＝SFTP 傳輸 / CTBP5011 / CTBP5004（收狀態）。

### 術語速查
| 詞 | 義 |
|---|---|
| **GIR MessageRefId** | GIR 訊息唯一參考編號（本程式 Payload/Metadata 檔名主體、也是 Ad-hoc 參數） |
| **SGD_STS** | Sign 狀態：`P`=待簽章加密、`S`=已簽章 |
| **APRVD_STS** | 核准狀態：`A`=已核准（由 CTAP5002 設定） |
| **TX_STS** | 傳輸狀態：`R`=Ready、`T`=Success、`N`=Pending、`F/U`=Failure |
| **TRANSMISSION_FLAG** | DOC_REF 表傳輸旗標，`Y`=已納入傳輸 |
| **CTS** | Common Transmission System（OECD 共同傳輸系統） |

---

# Part A — 逐節內容解讀

## A1. Change History（p.1）
單筆：`2026-04-09 Initial version`（CRF/UQF 欄「--」）。v1.0 初版。

## A2. Batch Job Properties（p.1）
- **Batch Job ID**：`CTBP5002`
- **Frequency**：`Monday–Friday at 21:00`（每工作日 21:00 排程）
- **Ad-hoc Run**：`Supported`
- **Ad-hoc Parameters**：位置 #1，格式 `Characters`，內容 = **GIR MessageRefId**

## A3. Description（p.1）
本程式將 **CTBP5001 Data Extraction 產生、且 CTAP5002 Approve Transmission of Data File 已核准** 的資料 XML 檔進行簽章與加密。

## A4. Program Flow（p.2，Program Flow 圖）★ Claude 已讀取圖內容
雙泳道流程：

**泳道 1 — CTBP5002 Sign & Encrypt Data File**
1. Start
2. Retrieve all **GIR_MSG_SPEC** records where **SGD_STS='P'**（待簽章加密）**AND APRVD_STS='A'**（已核准）
3. Locate the related GIR payload & metadata XML files in **`\data\out\pending`**
4. **Perform sign and encrypt**
5. Move finished payload & metadata XML files to **`\data\out\processed`**
6. Move the generated ZIP file to **`\data\cts\Outbox\<receiving country>\`**
7. **Perform Database Insert**：`CTS_DATA_PACKET`、`CTS_TX_REC`
8. **Perform Database Update**：`GIR_MSG_SPEC.SGD_STS='S'`；`CTS_TX_REC.TX_STS='R'`；`CTS_GSEC_DOC_REF / CTS_JSEC_DOC_REF / CTS_SMRY_DOC_REF / CTS_UTPR_DOC_REF 的 TRANSMISSION_FLAG='Y'`（四表）

**泳道 2 — Transfer Data Packets to OECD via SFTP**
1. SFTP the files in `\data\cts\Outbox\<receiving country>` to OECD server，並輸出 transmission log
2. **CTBP5011 Update Transmission Status**
3. Move transmitted files to **`\data\cts\Outbox_sent\<receiving country>`**
4. End

## A5. File Format（p.2）
| 名稱 | 格式 |
|---|---|
| Payload XML File | `<GIR MessageRefId>.xml` |
| Metadata XML File | `<GIR MessageRefId>.metadata.xml` |

## A6. Folder Path（p.2–3）
| 路徑 | 說明 |
|---|---|
| `\data\out\pending\` | 待簽章加密的 GIR payload/metadata |
| `\data\out\processed\` | 已簽章加密 |
| `\data\cts\Outbox\<ReceivingCountry>\` | 已加密、待傳輸 |
| `\data\cts\Outbox_sent\<ReceivingCountry>\` | 已傳輸 |

## A7. Database Records — For Outgoing GIR File（p.3）
| 表 | 動作 | 內容 |
|---|---|---|
| `GIR_MSG_SPEC` | Update | `SGD_STS='S'`（Signed） |
| `CTS_TX_REC` | Create | `TX_STS='R'`（Ready；R=Ready/T=Success/N=Pending/F,U=Failure） |
| `CTS_DATA_PACKET` | Create | `MSG_TYPE='GIR'`；`MS_ID=GIR_MSG_SPEC.MS_ID`；`STS_ID=NULL`；`ZIP_FILENAME={Outgoing GIR Filename}`；一組 `META_*` 取自 `/CTSSenderFileMetadata/*`（TX_CTRY、RX_CTRY、COMM_TYPE、SNDR_FILE_ID、FILE_FMT、ENC_SCM、FILE_TS、TAX_YR、FILE_REV_IND、ORI_TX_ID、SNDR_EMAIL） |
| `CTS_GSEC_DOC_REF` / `CTS_JSEC_DOC_REF` / `CTS_SMRY_DOC_REF` / `CTS_UTPR_DOC_REF` | Update | `TRANSMISSION_FLAG='Y'` |

## A8. CTBP5011 — Update Transmission Status（p.3–4）
DEV/UAT 環境需**手動產生 transmission log** 模擬 SFTP 上傳；開發者寫程式輸出下列內容並 email 給自己。範例（逗號分隔）：
`HK_GIR_20220831T143730554Z.zip,GIR,T,20220831143732`
程式以逗號拆解，取得四項：**File Name**、**File Type（GIR）**、**Transmission Status（T=Success / F,U=Failure / N=Pending）**、**Transmission Date and Time（YYYYMMDDHHMMSS）**。

## A9. 待釐清處（圖 vs 內文不一致）
1. **CTS_*_DOC_REF 表清單筆誤**（p.3）：內文表把四表寫成 `CTS_GSEC_DOC_REF、CTS_JSEC_DOC_REF、CTS_GSEC_DOC_REF、CTS_GSEC_DOC_REF`（GSEC 重複 3 次）。**圖**正確列出 GSEC/JSEC/SMRY/UTPR。以圖為準，測試涵蓋四表。
2. **`CTX_` vs `CTS_` 拼寫**：圖中寫 `CTX_DATA_PACKET`/`CTX_TX_REC`（X），內文寫 `CTS_DATA_PACKET`/`CTS_TX_REC`（S）。應為 `CTS_`。
3. **泳道 2 標題筆誤**：`Transfer Data Packets to OECD vis SFTP`（`vis`→`via`）。

---

# Part B — Test Cases (English)

> Format: **Test Case ID | Test Scenario | Test Data | Expected Result | Remark**. Direction = **Outbound** for all. "p.N" = page N; "Flow" = Program Flow diagram (p.2).

## B1. Batch Properties & Trigger

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-02-001 | Scheduled run at configured time | Batch triggered on a weekday at 21:00 | CTBP5002 runs on Mon–Fri 21:00 schedule | p.1 |
| TC-02-002 | Ad-hoc run supported with parameter | Ad-hoc trigger, param #1 = a valid GIR MessageRefId | Program runs for only that GIR MessageRefId | p.1 |
| TC-02-003 | Ad-hoc run without a valid MessageRefId | Param #1 = non-existent MessageRefId | No matching record processed; handled gracefully | p.1 |

## B2. Record Selection

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-02-004 | Select records pending signing and approved | GIR_MSG_SPEC with SGD_STS='P' AND APRVD_STS='A' | Record selected for signing & encryption | p.3, Flow |
| TC-02-005 | Exclude records not yet approved | SGD_STS='P' AND APRVD_STS≠'A' (e.g. 'P' pending/'R' rejected) | Record NOT selected | p.3, Flow |
| TC-02-006 | Exclude already-signed records | SGD_STS='S' | Record NOT selected | p.3, Flow |
| TC-02-007 | No qualifying records | No record with SGD_STS='P' AND APRVD_STS='A' | Program completes; nothing signed; no DB change | Flow |

## B3. Sign, Encrypt & File Movement

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-02-008 | Locate payload & metadata in pending folder | Selected MessageRefId with files in `\data\out\pending` | Both `<MessageRefId>.xml` and `<MessageRefId>.metadata.xml` located | p.2, Flow |
| TC-02-009 | Perform sign and encrypt | A located payload/metadata pair | Files are signed and encrypted successfully | p.2, Flow |
| TC-02-010 | Move finished XML to processed folder | After sign & encrypt | Payload & metadata moved to `\data\out\processed` | p.2, Flow |
| TC-02-011 | Move generated ZIP to Outbox by receiving country | ReceivingCountry='US' | ZIP moved to `\data\cts\Outbox\US\` | p.2–3, Flow |
| TC-02-012 | Missing payload/metadata in pending folder | Record selected but file absent in pending | Handled as error; not moved to processed | p.2, Flow |

## B4. Database Records (Outgoing GIR File)

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-02-013 | GIR_MSG_SPEC sign status updated | Record signed | GIR_MSG_SPEC.SGD_STS='S' | p.3, Flow |
| TC-02-014 | CTS_TX_REC created ready for transmission | After packaging | CTS_TX_REC created with TX_STS='R' | p.3, Flow |
| TC-02-015 | CTS_DATA_PACKET created with GIR type & metadata | Signed ZIP for a GIR | CTS_DATA_PACKET created: MSG_TYPE='GIR', MS_ID=GIR_MSG_SPEC.MS_ID, STS_ID=NULL, ZIP_FILENAME={outgoing filename} | p.3 |
| TC-02-016 | CTS_DATA_PACKET metadata fields mapped | Metadata XML present | All META_* mapped from /CTSSenderFileMetadata/* (TX_CTRY, RX_CTRY, COMM_TYPE, SNDR_FILE_ID, FILE_FMT, ENC_SCM, FILE_TS, TAX_YR, FILE_REV_IND, ORI_TX_ID, SNDR_EMAIL) | p.3 |
| TC-02-017 | All four DOC_REF transmission flags set | Signed GIR covering the four sections | TRANSMISSION_FLAG='Y' set in CTS_GSEC_DOC_REF, CTS_JSEC_DOC_REF, CTS_SMRY_DOC_REF, CTS_UTPR_DOC_REF | p.3, Flow (A9-1) |

## B5. Transmission (SFTP + CTBP5011)

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-02-018 | SFTP transmit from Outbox to OECD | ZIP in `\data\cts\Outbox\US\` | Files SFTP-transmitted to OECD server; transmission log output | p.2, Flow |
| TC-02-019 | CTBP5011 updates transmission status | After SFTP | CTBP5011 Update Transmission Status invoked | p.2, Flow |
| TC-02-020 | Move transmitted files to Outbox_sent | Transmission success | Files moved to `\data\cts\Outbox_sent\US\` | p.2, Flow |

## B6. Transmission Log Parsing (DEV/UAT)

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-02-021 | Parse comma-delimited transmission log | `HK_GIR_20220831T143730554Z.zip,GIR,T,20220831143732` | Split by comma into: File Name, File Type='GIR', TX Status='T', DateTime='20220831143732' | p.3–4 |
| TC-02-022 | Interpret transmission status codes | Status field = T / F / U / N | T=Success; F,U=Failure; N=Pending | p.4 |
| TC-02-023 | Multiple log lines processed | 3 log lines in the file | Each line parsed independently into its 4 fields | p.3–4 |

## B7. File & Folder Naming

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-02-024 | Payload file naming | MessageRefId=MR001 | Payload file = `MR001.xml` | p.2 |
| TC-02-025 | Metadata file naming | MessageRefId=MR001 | Metadata file = `MR001.metadata.xml` | p.2 |
| TC-02-026 | Folder path per lifecycle stage | Same file at each stage | pending → processed → Outbox\<country> → Outbox_sent\<country> in order | p.2–3 |

## B8. Edge / Discrepancy Cases

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-02-027 | Sign/encrypt failure handling | Encryption fails (e.g. missing certificate) | Record not marked Signed; not moved to Outbox; error surfaced | p.2, Flow |
| TC-02-028 | Transmission failure status | Log status = F/U for a file | TX_STS reflects failure; file not moved to Outbox_sent | p.4, Flow |
| TC-02-029 | Table-name typo verification (GSEC repeated) | Signed GIR covering four sections | Confirm all four DOC_REF tables updated, not GSEC three times — align text with image | p.3 (A9-1) |
| TC-02-030 | CTX_ vs CTS_ naming | DB insert step | Inserts target CTS_DATA_PACKET / CTS_TX_REC (not CTX_) — confirm naming | p.3 (A9-2) |

---

### 統計
- 章節：Change History / Batch Job Properties / Description / Program Flow（圖，已判讀）/ File Format / Folder Path / Database Records / CTBP5011 全數覆蓋。
- 測試用例：**30 條**，每條標註頁碼/圖。
- 待釐清：見 A9（圖與內文的表名、拼寫不一致）。
