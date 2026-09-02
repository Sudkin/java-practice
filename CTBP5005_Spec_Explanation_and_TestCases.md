# CTBP5005 Receive Data Packet (from Pillar Two Partners) — 功能規格書解讀 + 測試用例 (Test Cases)

> 對應文件：`Functional Specification – CTBP5005 Receive Data Packet v1.0 (working, 2026-07-02)`，全文 **8 頁**，含 **3 張圖**（主流程、GIR 計數、Correction 計數；Claude 已成功讀取全部三張）。
> Part A = 逐節解讀（中文）；Part B = 英文 Test Cases（Test Case ID / Test Scenario / Test Data / Expected Result / Remark；Remark 為出處頁碼/圖）。
> **方向（Direction）**：**Inbound**——接收夥伴傳來的 **GIR 資料檔**，驗證後產生**回覆狀態訊息（GIRMessageStatus）**交給 CTBP5003 簽章回傳。此為 CTBP5001（Outbound 抽數）的**對向程式**。

---

## 0. 定位
CTBP5005 是資料交換的 **inbound 入口**：夥伴把 GIR 資料封包送到 CTS，本程式下載、去重、解密、驗證，據結果產生 Accepted/Rejected 的 **GIRMessageStatus** 狀態訊息，放入 `\data\in\status_out\pending\` 供 **CTBP5003** 簽章加密後回傳夥伴。下游＝CTBP5003（簽章加密狀態訊息）。

### 術語速查
| 詞 | 義 |
|---|---|
| **GIRMessageStatus** | HK 回覆夥伴的狀態訊息（Accepted/Rejected） |
| **ValidationResult** | 驗證結果：`Accepted` / `Rejected` |
| **Inbox 系列** | `\data\cts\Inbox\`（收件）、`Inbox_duplicated\`、`Inbox_recv\`、`Inbox_invalid\` |
| **UPDATED_RX_CNT** | 更新 GIR_CNT/AMEND_CNT 的 stored procedure（收件側；對照 CTBP5001 的 UPDATE_TX_CNT） |
| **DOC_TYPE_IND** | `OECD_1/OECD_11`=新增、`OECD_2/OECD_12`=更正、`OECD_3/OECD_13`=刪除、`OECD_0/OECD_10`=重送 |

---

# Part A — 逐節內容解讀

## A1. Change History（p.1）
單筆：`2026-04-10 Initial version`。v1.0 初版。

## A2. Batch Job Properties（p.1）
- **Batch Job ID**：`CTBP5005`
- **Frequency**：`Monday–Friday at 08:00, 12:00 and 16:00`（每工作日三次）
- **Ad-hoc Run**：`Supported`
- **Ad-hoc Parameters**：**（空白，無參數）**

## A3. Description（p.1）
本程式處理夥伴管轄區傳來的 GIR 資料檔，並在回覆的 **Status Message** 檔中提供驗證結果。

## A4. Program Flow（p.2–4，三張圖）★ Claude 已讀取全部三張

### 圖1 — 主流程（p.2）
1. Start
2. **List of GIR files** from CTS Inbox `\data\cts\Inbox\`
3. **Same filename already processed?** Yes → 移至 `\data\cts\Inbox_duplicated\` →（至「All file processed?」）；No →
4. **Unzip** the file in `\data\in\pending\{ZIP Filename}\` and perform **XML schema validation**
5. **Decrypt XML files success?** No → 移至 `\data\cts\Inbox_invalid\` → **Set ValidationResult='Rejected'**；Yes →
6. **XML Validation Error Found?** No → **Set ValidationResult='Accepted'**；Yes → **Set ValidationResult='Rejected'**
7. **Move ZIP file** into `\data\cts\Inbox_recv\`（已驗證者）
8. **Move unzipped folder** into `\data\in\processed\{ZIP Filename}\`
9. **Generate GIRMessageStatus Payload & Metadata XML** into `\data\in\status_out\pending\` **for CTBP5003**
10. **All file processed?** No → 迴圈回步驟 2；Yes → **Send notification email**
11. End

### 圖2 — GIR 計數邏輯（p.2–4）
`MessageSpec.MessageTypeIndic='GIR_101' AND FilingInfo.DocTypeIndic IN ('OECD_1','OECD_11')` → **GIR Count +1**；迴圈至所有 FilingInfo 處理完。（與 CTBP5001 一致）

### 圖3 — Correction 計數邏輯（p.2–4）
任一成立 → **Correction Count +1**：
- (a) `MessageTypeIndic='GIR_102'`；或
- (b) `GIR_101 AND FilingInfo.DocTypeIndic IN ('OECD_2','OECD_12','OECD_3','OECD_13')`；或
- (c) `GIR_101 AND FilingInfo.DocTypeIndic IN ('OECD_0','OECD_10') AND 任一段(GeneralSection/Summary/JurisdictionSection/UTPRAttribution).DocTypeIndic IN ('OECD_1','OECD_11')`。（與 CTBP5001 一致）

## A5. Program Logic（p.5–6）
1. 夥伴送 GIR 封包到 HK（可自 CTS server 下載）；先對**未加密**檔做前置檢查：**檔名是否已處理**——是則移 Duplicate 並停止。
2. 用 MessageSpec 內 **country code** 檢查是否來自交換夥伴，且**系統日期在該 incoming GIR 的 Start/End Date 內**；非夥伴則**不解密**。
3. 從 metadata（或 MessageRefId 的 SenderField）取 **data year**，檢查是否在 incoming GIR 的 **Start/End Year** 內。
4. 續**解密＋驗證**，並做 **threat scan**。
5. **Accepted** → 建立並寫入 message info + data file 記錄，產生 accept status message。
6. **Rejected** → 只寫入 message spec / data packet / transmission 記錄，**不需寫入 data file 記錄**；產生帶錯誤訊息的 rejected status message。
7. 依驗證結果產生 **status message XML**（須符合 OECD schema），並更新資料庫。
8. 無論 Accepted/Rejected，驗證完成即**寄 email 通知**指定使用者。
9. `GIR_MSG_SPEC.GIR_CNT` 與 `GIR_MSG_SPEC.AMEND_CNT` 由 stored procedure **`UPDATED_RX_CNT`**（輸入 MessageRefID）更新。
10. 掃描 Inbox 內 `.zip` 且檔名含 `GIR` 的檔。
11. 解密需**寄送方 public certificate**（於 keystore 檢查）。
12. 處理完寄 email 通知。

## A6. Database Records（p.6–7）
**For Incoming GIR File：**
| 表 | 動作 | 內容 |
|---|---|---|
| `CTS_DATA_PACKET` | Insert | `MSG_TYPE='GIR'`；`ZIP_FILENAME={Incoming GIR Filename}`；`MS_ID=GIR_MSG_SPEC.MS_ID`；`STS_ID=CTS_STS_MSG.STS_ID`；一組 `META_*`（/CTSSenderFileMetadata/*） |
| `CTS_TX_REC` | Insert | — |
| `GIR_*` | Insert | **僅** XSD 驗證成功的 XML 記錄才寫入 |

**For generated Status Message：**
| 表 | 動作 | 內容 |
|---|---|---|
| `CTS_DATA_PACKET` | Insert | `MSG_TYPE='GIRMessageStatus'`；`ZIP_FILENAME=NULL`；`MS_ID=GIR_MSG_SPEC.MS_ID`；`STS_ID=CTS_STS_MSG.STS_ID`；`META_*` |
| `CTS_STS_MSG` / `CTS_VALID_BY` | Insert | `CTS_STS_MSG.SGD_STS='P'`（待簽章）；`STS_MSG_REF_ID` 格式 = `'Status'+HK+<Reporting Year>+<Receiving Country>+'-'+<yyyyMMddHHmmss>` |
| `CTS_FILE_ERR` / `CTS_REC_ERR` / `CTS_ERR_FIELD` / `CTS_ERR_DOC_REF_ID` | Insert | **僅當** ValidationResult='Rejected' 時寫入 |

**Status Message 檔放置（p.7）**：Payload `\data\in\status_out\pending\<GIRMessageStatus MessageRefId>.xml`；Metadata `…metadata.xml`。

## A7. Email Template（p.7–8）
三種通知：
1. **Accepted**（p.7）：主旨「…containing no errors」；內容列 Send by / Message Ref ID / Message Type（GIR_101）/ Transmission ID / Result: ACCEPTED，提示可用 CTLQ5003。
2. **Rejected**（p.8）：主旨「…containing file error(s)」；內容多列 **Error Code**（例 50002）、Result: REJECTED。
3. **Fatal Error 未處理**（p.8；原文筆誤「Fata Error」）：主旨「unsuccessful processing…」；內容列 Country / Filename / Reason:{0}。

## A8. 待釐清處
1. **`UPDATED_RX_CNT` 命名**：本程式用 `UPDATED_RX_CNT`（過去式且 RX），CTBP5001 用 `UPDATE_TX_CNT`。命名不一致，建議統一/確認。
2. **OECD_1X 擴充碼**：計數圖用 `OECD_10/11/12/13`（與 CTBP5001、CTBP5004 一致）；正文未列完整值域，建議與 BA 定清。
3. **Rejected 不寫 data file 記錄**：step 6 明示 Rejected 不寫 `GIR_*`；測試須驗證此差異。
4. **「Fata Error」筆誤** → 應為「Fatal Error」。
5. **主流程圖 vs Program Logic 順序**：圖只畫「檔名去重→解壓驗證→解密→…」，Program Logic 另有「非夥伴不解密」「data year 在 Start/End Year 內」等檢查，圖未完整呈現；以 Program Logic 為準並補測。

---

# Part B — Test Cases (English)

> Format: **Test Case ID | Test Scenario | Test Data | Expected Result | Remark**. Direction = **Inbound** for all. "p.N" = page N; "Flow1"=main flow; "Flow2"=GIR count; "Flow3"=Correction count (all p.2–4).

## B1. Batch Properties & Trigger

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-05-001 | Scheduled runs three times daily | Weekday triggers at 08:00/12:00/16:00 | CTBP5005 runs on each configured time | p.1 |
| TC-05-002 | Ad-hoc run supported (no parameter) | Ad-hoc trigger, no parameter | Program runs and scans Inbox | p.1 |

## B2. Inbox Scan, Duplicate & Preliminary Checks

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-05-003 | Scan Inbox for GIR zips | `CH_GIR_20220317T144955000Z_XF2W2KlJOK_N77IKZ_jAFwb8gyxIoXBx.zip` in `\data\cts\Inbox\` | File picked up | p.5, Flow1 |
| TC-05-004 | Ignore non-matching files | `.zip` without "GIR" in name / non-zip | File NOT picked up | p.5, Flow1 |
| TC-05-005 | Duplicate filename moved and stopped | Filename already processed | Moved to `\data\cts\Inbox_duplicated\`; processing stops for it | p.5, p.2, Flow1 |
| TC-05-006 | Preliminary check on unencrypted file | An incoming file | Filename-processed check performed before decrypt | p.5 |

## B3. Partner & Date/Year Validation

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-05-007 | Sender is an exchange partner (in period) | MessageSpec country code = a partner; system date within incoming GIR Start/End Date | Processing continues (decrypt) | p.6 |
| TC-05-008 | Sender not a partner → no decrypt | MessageSpec country code not a partner | Data file NOT decrypted | p.6 |
| TC-05-009 | System date outside incoming GIR Start/End Date | Partner but system date out of period | Not processed as valid incoming (per period check) | p.6 |
| TC-05-010 | Data year within Start/End Year | Data year from metadata/SenderField within incoming GIR Start/End Year | Passes the year check | p.6 |
| TC-05-011 | Data year from MessageRefId SenderField | Year absent in metadata; present in MessageRefId SenderField | Year retrieved from SenderField and checked | p.6 |

## B4. Decrypt, Threat Scan & Validation Branch (Flow1)

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-05-012 | Public certificate present | Sending jurisdiction's public cert in keystore | Decryption proceeds | p.6, Flow1 |
| TC-05-013 | Public certificate missing | Cert absent from keystore | Decrypt fails; file → `\data\cts\Inbox_invalid\`; ValidationResult='Rejected' | p.6, p.2, Flow1 |
| TC-05-014 | Threat scan applied | Decrypted data file | Threat scan performed to ensure file is threat-free | p.6 |
| TC-05-015 | Unzip and XML schema validation | Valid zip | Unzipped into `\data\in\pending\{ZIP Filename}\` and XML schema validated | p.2, Flow1 |
| TC-05-016 | Decrypt success + no validation error → Accepted | Decrypt OK, XML valid | ValidationResult='Accepted' | p.2, Flow1 |
| TC-05-017 | Decrypt success + validation error → Rejected | Decrypt OK, XML invalid | ValidationResult='Rejected' | p.2, Flow1 |
| TC-05-018 | Validated file moved to Inbox_recv | Accepted or Rejected (validated) | ZIP moved to `\data\cts\Inbox_recv\` | p.2, Flow1 |
| TC-05-019 | Unzipped folder moved to processed | Any processed file | Unzipped folder moved to `\data\in\processed\{ZIP Filename}\` | p.2, Flow1 |

## B5. Status Message Generation (for CTBP5003)

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-05-020 | Generate GIRMessageStatus payload & metadata | A processed file | GIRMessageStatus payload & metadata XML generated into `\data\in\status_out\pending\` for CTBP5003 | p.2, p.7, Flow1 |
| TC-05-021 | Status message conforms to OECD schema | Generated status XML | File conforms to OECD-specified schema | p.6 |
| TC-05-022 | STS_MSG_REF_ID format | RPT_YR=2025, RX=GH, time=20260702103000 | STS_MSG_REF_ID = `StatusHK2025GH-20260702103000` | p.7 |
| TC-05-023 | Status message file naming/path | MessageRefId=StatusHK2025GH-20260702103000 | Payload `...\pending\StatusHK2025GH-20260702103000.xml`; Metadata `...metadata.xml` | p.7 |

## B6. Database Records — Incoming GIR File

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-05-024 | CTS_DATA_PACKET inserted for incoming GIR | An incoming GIR file | Insert: MSG_TYPE='GIR'; ZIP_FILENAME={incoming filename}; MS_ID=GIR_MSG_SPEC.MS_ID; STS_ID=CTS_STS_MSG.STS_ID | p.6–7 |
| TC-05-025 | Incoming metadata fields mapped | Metadata present | META_* mapped from /CTSSenderFileMetadata/* | p.6–7 |
| TC-05-026 | CTS_TX_REC inserted | Any incoming GIR | CTS_TX_REC record inserted | p.7 |
| TC-05-027 | GIR_* inserted only when XSD-valid | XML validated against XSD | GIR_* records inserted; invalid XML not saved | p.7 |
| TC-05-028 | Accepted inserts message info + data file records | ValidationResult='Accepted' | Message info and data file records created and inserted | p.6 |
| TC-05-029 | Rejected does not insert data file record | ValidationResult='Rejected' | Only message spec / data packet / transmission inserted; NO data file (GIR_*) record | p.6 (A8-3) |

## B7. Database Records — Generated Status Message

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-05-030 | CTS_DATA_PACKET inserted for status message | A generated status message | Insert: MSG_TYPE='GIRMessageStatus'; ZIP_FILENAME=NULL; MS_ID=GIR_MSG_SPEC.MS_ID; STS_ID=CTS_STS_MSG.STS_ID | p.7 |
| TC-05-031 | CTS_STS_MSG inserted pending signing | A generated status message | CTS_STS_MSG inserted with SGD_STS='P' (Pending for Signing & Encryption) | p.7 |
| TC-05-032 | CTS_VALID_BY inserted | A generated status message | CTS_VALID_BY record inserted | p.7 |
| TC-05-033 | Error tables inserted only when Rejected | ValidationResult='Rejected' | Insert into CTS_FILE_ERR, CTS_REC_ERR, CTS_ERR_FIELD, CTS_ERR_DOC_REF_ID | p.7 |
| TC-05-034 | Error tables NOT inserted when Accepted | ValidationResult='Accepted' | No insert into the error tables | p.7 |

## B8. GIR / Correction Count Logic (Flow2 / Flow3)

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-05-035 | GIR count — GIR_101 + OECD_1 | MessageTypeIndic='GIR_101', FilingInfo.DocTypeIndic='OECD_1' | GIR Count + 1 | p.2–4, Flow2 |
| TC-05-036 | GIR count — GIR_101 + OECD_11 | 'GIR_101', 'OECD_11' | GIR Count + 1 (extended code) | p.2–4, Flow2 (A8-2) |
| TC-05-037 | GIR count — GIR_101 + OECD_0 not counted | 'GIR_101', 'OECD_0' | NOT counted as GIR | p.2–4, Flow2 |
| TC-05-038 | Correction count — GIR_102 | MessageTypeIndic='GIR_102' | Correction Count + 1 | p.2–4, Flow3 |
| TC-05-039 | Correction count — GIR_101 + OECD_2/12/3/13 | 'GIR_101', FilingInfo.DocTypeIndic='OECD_2' | Correction Count + 1 | p.2–4, Flow3 |
| TC-05-040 | Correction count — GIR_101 + OECD_0/10 + section OECD_1/11 | 'GIR_101', FilingInfo='OECD_0', GeneralSection.DocTypeIndic='OECD_1' | Correction Count + 1 | p.2–4, Flow3 |
| TC-05-041 | Correction count — GIR_101 + OECD_0 + no section OECD_1/11 not counted | 'GIR_101', FilingInfo='OECD_0', no section in ('OECD_1','OECD_11') | NOT counted as correction | p.2–4, Flow3 |
| TC-05-042 | Counts persisted via UPDATED_RX_CNT | Processed MessageRefID | GIR_MSG_SPEC.GIR_CNT & AMEND_CNT updated by UPDATED_RX_CNT(MessageRefID) | p.6 (A8-1) |

## B9. Notification Email

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-05-043 | Accepted notification email | Result=ACCEPTED | Email "containing no errors" listing Send by/Message Ref ID/Type/Transmission ID/Result=ACCEPTED; mentions CTLQ5003 | p.7 |
| TC-05-044 | Rejected notification email with error code | Result=REJECTED, Error Code=50002 | Email "containing file error(s)" including Error Code and Result=REJECTED | p.8 |
| TC-05-045 | Fatal-error notification email | Fatal error processing a file | Email "unsuccessful processing" listing Country/Filename/Reason:{0} | p.8 |
| TC-05-046 | Email sent regardless of Accepted/Rejected | Validation completed (either) | Notification email sent to designated users | p.6, Flow1 |

## B10. Edge Cases

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-05-047 | Empty Inbox | No files in `\data\cts\Inbox\` | Program completes; no processing; no DB change | Flow1 |
| TC-05-048 | Mixed batch (valid + duplicate + undecryptable) | One valid, one duplicate-name, one no-cert | valid→Inbox_recv (Accepted/Rejected), duplicate→Inbox_duplicated, undecryptable→Inbox_invalid | p.2, Flow1 |
| TC-05-049 | Non-partner file is skipped without decrypt | File from non-partner country code | Not decrypted; no GIR_* inserted | p.6 |
| TC-05-050 | UPDATED_RX_CNT naming reconciliation | Any processed MessageRefID | Confirm stored proc name (`UPDATED_RX_CNT` vs `UPDATE_TX_CNT`) | p.6 (A8-1) |

---

### 統計
- 章節：Change History / Batch Job Properties / Description / Program Flow（三圖，已判讀）/ Program Logic (12 steps) / Database Records / Email Template 全數覆蓋。
- 測試用例：**50 條**，每條標註頁碼/圖。
- 待釐清：見 A8（UPDATED_RX_CNT 命名、OECD_1X 值域、Rejected 不寫 data file、圖與 Program Logic 落差）。
