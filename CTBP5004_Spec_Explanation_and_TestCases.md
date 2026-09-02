# CTBP5004 Receive Status Message (from Pillar Two Partners) — 功能規格書解讀 + 測試用例 (Test Cases)

> 對應文件：`Functional Specification – CTBP5004 Receive Status Message v1.0 (working, 2026-06-17)`，全文 **10 頁**，含 **2 張 Program Flow 圖**（主流程 p.3、更新子流程 p.4；Claude 已成功讀取兩張圖內容）。
> Part A = 逐節解讀（中文）；Part B = 英文 Test Cases（Test Case ID / Test Scenario / Test Data / Expected Result / Remark；Remark 為出處頁碼/圖）。
> **方向（Direction）**：本程式為 **Inbound**（接收並處理夥伴回傳的**狀態訊息**，據以更新 HK 的送出記錄）。此即 CTBP5001 流程尾端的 inbound 回程。

---

## 0. 定位
CTBP5004 收取夥伴管轄區回傳的 **GIRMessageStatus** 狀態訊息（回覆 HK 先前送出的 GIR 是否 Accepted/Rejected），驗證通過後**更新對應的送出記錄**（DOC_REF、傳輸旗標、刪除旗標等），並發送通知 email。是整條交換鏈的**收尾**。

### 術語速查
| 詞 | 義 |
|---|---|
| **GIRMessageStatus** | 夥伴回傳的狀態訊息（Accepted/Rejected） |
| **OriginalMessageRefID / CTSTransmissionID** | 用來回找原始 GIR 送出記錄的鍵（CTSTransmissionID 比對**不分大小寫**） |
| **ValidationResult** | 狀態訊息驗證結果：`Accepted` / `Rejected` |
| **DOC_TYPE_IND** | 文件類型：`OECD_1/OECD_11`=新增、`OECD_3/OECD_13`=刪除（本程式 SQL 明確使用此擴充系列） |
| **DELETE_FLAG / TRANSMISSION_FLAG** | DOC_REF 表的刪除旗標 / 傳輸旗標 |
| **Inbox 系列資料夾** | `\data\cts\Inbox\`（收件）、`Inbox_duplicated\`、`Inbox_recv\`、`Inbox_invalid\` |

---

# Part A — 逐節內容解讀

## A1. Change History（p.1）
單筆：`2026-04-10 Initial version`。v1.0 初版。

## A2. Batch Job Properties（p.1）
- **Batch Job ID**：`CTBP5004`
- **Frequency**：`Monday–Friday at 08:00, 12:00 and 16:00`（每工作日三次）
- **Ad-hoc Run**：`Supported`
- **Ad-hoc Parameters**：**（空白，無參數）**

## A3. Description（p.1）
處理夥伴管轄區回傳的狀態訊息；若狀態訊息通過所有驗證，則更新資料庫中對應的**送出（outgoing）記錄**。

## A4. Program Flow（p.3 圖1「主流程」、p.4 圖2「更新子流程」）★ Claude 已讀取兩張圖

### 圖1 — 主流程（p.3）
1. Start
2. **List of GIRMessageStatus files** from CTS Inbox `\data\cts\Inbox\`
3. **Same filename already processed?** Yes → 移至 `\data\cts\Inbox_duplicated\` →（至「All file processed?」）；No →
4. **Status Message already received?** Yes → **Override the previous result?**（Yes → 續解壓驗證；No → 視為不處理 →「All file processed?」）；No →
5. **Unzip** the file in `\data\in\pending\{ZIP Filename}\` and perform validation
6. **Decrypt XML files success?** No → 移至 `\data\cts\Inbox_invalid\`；Yes →
7. **XML validation error found?** Yes → 移至 `\data\cts\Inbox_invalid\`；No → 移至 `\data\cts\Inbox_recv\`
8. **Move unzipped folder** into `\data\in\processed\`
9. **All file processed?** No → 迴圈回步驟 2；Yes → **Send notification email** about the processing result
10. End

### 圖2 — Update Document References & Transmission Status（p.4）
1. Start
2. **ValidationResult = 'Accepted'?**
   - **Yes** → Update `LAST_FCE_TIN`、`LAST_DOC_REF_ID`、`LAST_CTS_DOC_REF_ID`（四表：CTS_GSEC/JSEC/SMRY/UTPR_DOC_REF）→ 續判 DELETE_FLAG
     - `DOC_TYPE_IND='OECD_3'?` Yes → `DELETE_FLAG='Y'`
     - 否則 `DOC_TYPE_IND='OECD_1'?` Yes → `DELETE_FLAG=NULL`
     - （SQL 另含 `OECD_13→'Y'`、`OECD_11→NULL`、其餘保留原值，見下）
   - **No**（即 Rejected）→ 直接跳至設定 TRANSMISSION_FLAG
3. **Set TRANSMISSION_FLAG=NULL**（四表：CTS_GSEC/JSEC/SMRY/UTPR_DOC_REF）
4. End

## A5. Program Logic（p.5）
1. 掃描 Inbox 內副檔名 `.zip` 且檔名含 `GIRMessageStatus` 的檔（例 `CH_GIRMessageStatus_20220317T144955000Z_XF2W2KlJOK_N77IKZ_jAFwb8gyxIoXBx.zip`）。
2. 解密需**寄送方管轄區的 public certificate**；程式於 keystore 檢查該憑證。
3. 以狀態訊息內 `\GIRStatusMessage\OriginalMessage\OriginalMessageRefID\` 或 `\GIRStatusMessage\OriginalMessage\FileMetaData\CTSTransmissionID\` 為鍵，回找原始 GIR 訊息；**CTSTransmissionID 比對不分大小寫**。
4. 可處理**先前已 Accepted/Rejected** 的狀態訊息並覆寫前次結果（此情境 by request）。
5. **Update Document References and Transmission Status**（`{0}`=狀態訊息的 Original Message Reference ID）：
   - **a. Accepted**（p.5–7）：對四表 `CTS_GSEC/JSEC/SMRY/UTPR_DOC_REF` 執行 UPDATE：
     - `LAST_FCE_TIN`、`LAST_DOC_REF_ID`、`LAST_CTS_DOC_REF_ID` ← 對應 `V_CTS_*` 視圖；
     - `DELETE_FLAG` ← `CASE WHEN DOC_TYPE_IND IN ('OECD_3','OECD_13') THEN 'Y' WHEN IN ('OECD_1','OECD_11') THEN NULL ELSE 保留原值 END`；
     - `TRANSMISSION_FLAG=NULL`、`LAST_UPD_AT=SYSDATE`、`LAST_UPD_BY=2`；
     - 條件：`WHERE EXISTS (... V.TRANSMISSION_FLAG='Y' AND V.MSG_REF_ID={0})`。
   - **b. Rejected**（p.8）：對四表僅設 `TRANSMISSION_FLAG=NULL`、`LAST_UPD_AT=SYSDATE`、`LAST_UPD_BY=2`；條件同上。

## A6. Database Records — For Incoming GIRMessageStatus File（p.8–9）
| 表 | 動作 | 內容 |
|---|---|---|
| `CTS_GSEC/JSEC/SMRY/UTPR_DOC_REF` | Update | 依驗證結果更新 `LAST_FCE_TIN`、`LAST_DOC_REF_ID`、`LAST_CTS_DOC_REF_ID`、`DELETE_FLAG`、`TRANSMISSION_FLAG` |
| `CTS_DATA_PACKET` | Insert | `MSG_TYPE='GIRMessageStatus'`；`ZIP_FILENAME={Incoming filename}`；`MS_ID=GIR_MSG_SPEC.MS_ID`（依 OriginalMessageRefID 或 CTSTransmissionID 回找原始 GIR）；`STS_ID=CTS_STS_MSG.STS_ID`；一組 `META_*` 取自 `/CTSSenderFileMetadata/*` |
| `CTS_TX_REC` / `CTS_STS_MSG` / `CTS_VALID_BY` | Insert | 建立傳輸/狀態/驗證記錄 |
| `CTS_FILE_ERR` / `CTS_REC_ERR` / `CTS_ERR_FIELD` / `CTS_ERR_DOC_REF_ID` | Insert | **僅當** ValidationResult='Rejected' 時寫入 |
| `CTS_DATA_PACKET`（原始 GIR 記錄） | Update | `STS_ID=CTS_STS_MSG.STS_ID`，`WHERE DP_ID={original GIR record}` |

## A7. Email Template（p.9–10）
四種通知：
1. **Accepted**（p.9）：主旨「…containing no errors」；內容列 Send by / Original Message Ref ID / Type（GIR_101）/ Transmission Time / Transmission ID / Result: ACCEPTED，並提示可用 CTLQ5003 查閱。
2. **Rejected**（p.9）：主旨「…containing file error(s)」；內容多列 **Error Code**（例 50002）、Result: REJECTED。
3. **Fatal Error 未處理**（p.10；原文筆誤「Fata Error」）：主旨「unsuccessful processing…」；內容列 Country / Filename / Reason:{0}。
4. **Already Accepted/Rejected Data File**（p.10）：內容列本次結果與**前一次狀態訊息**的 Message Ref ID 及 Result（例 本次 ACCEPTED、前次 REJECTED）。

## A8. 待釐清處
1. **DOC_TYPE_IND 擴充系列**：SQL 明確使用 `OECD_1/OECD_11`、`OECD_3/OECD_13`（與 CTBP5001 計數圖一致），但圖2 只畫 `OECD_1`、`OECD_3`（簡化）。以 **SQL 為準**；仍建議與 BA 確認 `_1X` 系列來源與完整值域。
2. **「Fata Error」筆誤**（p.10）→ 應為「Fatal Error」。
3. **Email 範例日期格式不一**：Accepted 用 `2026/03/25 22:40:02`、Rejected 用 `2022-06-10`；疑為樣本資料，非規格要求。
4. **CTSTransmissionID 大小寫**：規格明示**不分大小寫**比對；`OriginalMessageRefID` 是否亦不分大小寫未明示，建議確認。
5. **Override 情境**：步驟4 註明「by request」；正式行為（是否預設允許覆寫）建議確認。

---

# Part B — Test Cases (English)

> Format: **Test Case ID | Test Scenario | Test Data | Expected Result | Remark**. Direction = **Inbound** for all. "p.N" = page N; "Flow1"=main flow (p.3); "Flow2"=update sub-flow (p.4).

## B1. Batch Properties & Trigger

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-04-001 | Scheduled runs three times daily | Weekday triggers at 08:00/12:00/16:00 | CTBP5004 runs on each configured time | p.1 |
| TC-04-002 | Ad-hoc run supported (no parameter) | Ad-hoc trigger, no parameter | Program runs and scans Inbox as usual | p.1 |

## B2. Inbox Scan & File Identification

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-04-003 | Scan Inbox for status message zips | `CH_GIRMessageStatus_20220317T144955000Z_XF2W2KlJOK_N77IKZ_jAFwb8gyxIoXBx.zip` in `\data\cts\Inbox\` | File picked up for processing | p.5, Flow1 |
| TC-04-004 | Ignore non-matching files | A `.zip` without "GIRMessageStatus" in name, or a non-zip file | File NOT picked up | p.5, Flow1 |
| TC-04-005 | Process multiple files in one run | Several qualifying zips in Inbox | Each file processed; loop continues until all processed | Flow1 |

## B3. Duplicate & Already-Received Handling

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-04-006 | Same filename already processed | A file whose filename was processed before | Moved to `\data\cts\Inbox_duplicated\`; not reprocessed | p.3, Flow1 |
| TC-04-007 | Status message already received — override | Status message already received; override = Yes | Proceeds to unzip & validation (previous result overridden) | p.5, Flow1 |
| TC-04-008 | Status message already received — no override | Already received; override = No | Not reprocessed; notification "Already Accepted/Rejected" applies | p.3, p.10, Flow1 |
| TC-04-009 | Override a previously Rejected result with Accepted | Prior status = REJECTED; new = ACCEPTED; override=Yes | New ACCEPTED result applied; original record updated accordingly | p.5, p.10 |

## B4. Decryption & Validation

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-04-010 | Public certificate present in keystore | Sending jurisdiction's public cert available | Decryption proceeds | p.5, Flow1 |
| TC-04-011 | Public certificate missing | Sending jurisdiction's public cert absent from keystore | Decrypt fails; file moved to `\data\cts\Inbox_invalid\` | p.5, p.3, Flow1 |
| TC-04-012 | Unzip to pending folder | Valid zip | Unzipped into `\data\in\pending\{ZIP Filename}\` | p.3, Flow1 |
| TC-04-013 | Decrypt success then no validation error | Decrypt OK, XML valid | File moved to `\data\cts\Inbox_recv\` | p.3, Flow1 |
| TC-04-014 | Decrypt success but validation error found | Decrypt OK, XML invalid | File moved to `\data\cts\Inbox_invalid\` | p.3, Flow1 |
| TC-04-015 | Move unzipped folder after processing | Any processed file | Unzipped folder moved into `\data\in\processed\` | p.3, Flow1 |

## B5. Original GIR Record Retrieval

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-04-016 | Retrieve original by OriginalMessageRefID | Status msg with `\GIRStatusMessage\OriginalMessage\OriginalMessageRefID` | Original GIR record located by that key | p.5 |
| TC-04-017 | Retrieve original by CTSTransmissionID | Status msg with `\FileMetaData\CTSTransmissionID` | Original GIR record located by that key | p.5 |
| TC-04-018 | CTSTransmissionID matching is case-insensitive | CTSTransmissionID differing only in letter case | Still matches the original record | p.5 |
| TC-04-019 | Original record not found | Neither key matches any record | Handled as fatal/unprocessed (see email template 3) | p.5, p.10 |

## B6. Update Logic — ValidationResult = Accepted (Flow2 / SQL)

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-04-020 | Accepted updates LAST_* fields in four tables | ValidationResult='Accepted', {0}=OriginalMessageRefID | CTS_GSEC/JSEC/SMRY/UTPR_DOC_REF get LAST_FCE_TIN, LAST_DOC_REF_ID, LAST_CTS_DOC_REF_ID from V_CTS_* | p.4, p.5–7, Flow2 |
| TC-04-021 | Accepted + DOC_TYPE_IND OECD_3 sets delete flag | Accepted; DOC_TYPE_IND='OECD_3' | DELETE_FLAG='Y' | p.5–7, p.4, Flow2 |
| TC-04-022 | Accepted + DOC_TYPE_IND OECD_13 sets delete flag | Accepted; DOC_TYPE_IND='OECD_13' | DELETE_FLAG='Y' | p.5–7 (A8-1) |
| TC-04-023 | Accepted + DOC_TYPE_IND OECD_1 clears delete flag | Accepted; DOC_TYPE_IND='OECD_1' | DELETE_FLAG=NULL | p.5–7, p.4, Flow2 |
| TC-04-024 | Accepted + DOC_TYPE_IND OECD_11 clears delete flag | Accepted; DOC_TYPE_IND='OECD_11' | DELETE_FLAG=NULL | p.5–7 (A8-1) |
| TC-04-025 | Accepted + other DOC_TYPE_IND keeps existing delete flag | Accepted; DOC_TYPE_IND not in the above sets | DELETE_FLAG unchanged (ELSE branch keeps prior value) | p.5–7 |
| TC-04-026 | Accepted clears transmission flag | Accepted record | TRANSMISSION_FLAG=NULL in the four tables | p.4, p.5–7, Flow2 |
| TC-04-027 | Accepted update guarded by TRANSMISSION_FLAG='Y' and MSG_REF_ID | Row with TRANSMISSION_FLAG='Y' and MSG_REF_ID={0} | Only such rows updated (WHERE EXISTS condition) | p.5–7 |
| TC-04-028 | Accepted audit columns set | Accepted update | LAST_UPD_AT=SYSDATE; LAST_UPD_BY=2 | p.5–7 |

## B7. Update Logic — ValidationResult = Rejected (Flow2 / SQL)

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-04-029 | Rejected clears transmission flag only | ValidationResult='Rejected', {0}=OriginalMessageRefID | Four tables get TRANSMISSION_FLAG=NULL (LAST_* and DELETE_FLAG NOT changed) | p.8, p.4, Flow2 |
| TC-04-030 | Rejected audit columns set | Rejected update | LAST_UPD_AT=SYSDATE; LAST_UPD_BY=2 | p.8 |
| TC-04-031 | Rejected update guarded by TRANSMISSION_FLAG='Y' and MSG_REF_ID | Row with TRANSMISSION_FLAG='Y' and MSG_REF_ID={0} | Only such rows updated | p.8 |

## B8. Database Records (Incoming GIRMessageStatus File)

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-04-032 | CTS_DATA_PACKET inserted for status message | Incoming status message | Insert: MSG_TYPE='GIRMessageStatus'; ZIP_FILENAME={incoming filename}; MS_ID from original GIR; STS_ID=CTS_STS_MSG.STS_ID | p.9 |
| TC-04-033 | CTS_DATA_PACKET metadata fields mapped | Metadata present | META_* mapped from /CTSSenderFileMetadata/* | p.9 |
| TC-04-034 | CTS_TX_REC / CTS_STS_MSG / CTS_VALID_BY inserted | Any processed status message | Records inserted into the three tables | p.9 |
| TC-04-035 | Error tables inserted only when Rejected | ValidationResult='Rejected' | Insert into CTS_FILE_ERR, CTS_REC_ERR, CTS_ERR_FIELD, CTS_ERR_DOC_REF_ID | p.9 |
| TC-04-036 | Error tables NOT inserted when Accepted | ValidationResult='Accepted' | No insert into the error tables | p.9 |
| TC-04-037 | Original GIR record linked to status | Original found via OriginalMessageRefID/CTSTransmissionID | CTS_DATA_PACKET.STS_ID=CTS_STS_MSG.STS_ID WHERE DP_ID={original GIR record} | p.9 |

## B9. Notification Email

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-04-038 | Accepted notification email | A status message with Result=ACCEPTED | Email "containing no errors" listing Send by/Original Msg Ref ID/Type/Transmission Time/Transmission ID/Result=ACCEPTED; mentions CTLQ5003 | p.9 |
| TC-04-039 | Rejected notification email with error code | Result=REJECTED, Error Code=50002 | Email "containing file error(s)" including Error Code and Result=REJECTED | p.9–10 |
| TC-04-040 | Fatal-error notification email | Processing fails fatally for a file | Email "unsuccessful processing" listing Country/Filename/Reason:{0} | p.10 |
| TC-04-041 | Already-accepted/rejected notification email | Status msg for an already-decided data file (no override) | Email lists current Result and previous status message's Ref ID and Result | p.10 |
| TC-04-042 | Email sent only after all files processed | End of run | Notification email(s) sent about the processing result | p.3, Flow1 |

## B10. Edge / Discrepancy Cases

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-04-043 | Empty Inbox | No files in `\data\cts\Inbox\` | Program completes; no processing; no DB change | Flow1 |
| TC-04-044 | Mixed batch (valid + duplicate + invalid) | Inbox with one valid, one duplicate-name, one undecryptable | Valid→Inbox_recv; duplicate→Inbox_duplicated; undecryptable→Inbox_invalid; all folders in \data\in\processed | p.3, Flow1 |
| TC-04-045 | DOC_TYPE_IND extended-code coverage | Accepted rows with OECD_1/OECD_11/OECD_3/OECD_13 mixed | DELETE_FLAG resolved per CASE for each code — confirm `_1X` series validity | p.5–7 (A8-1) |
| TC-04-046 | Rejected does not overwrite LAST_*/DELETE_FLAG | Rejected after a prior Accepted set LAST_* | LAST_* and DELETE_FLAG retain prior values; only TRANSMISSION_FLAG cleared | p.8 |
| TC-04-047 | OriginalMessageRefID case sensitivity | OriginalMessageRefID differing in case | Confirm whether match is case-insensitive (spec explicit only for CTSTransmissionID) | p.5 (A8-4) |
| TC-04-048 | Override behaviour is by-request | Already-decided status; override flag toggled | Confirm default override behaviour with BA | p.5 (A8-5) |

---

### 統計
- 章節：Change History / Batch Job Properties / Description / Program Flow（兩圖，已判讀）/ Program Logic（Accepted+Rejected SQL）/ Database Records / Email Template 全數覆蓋。
- 測試用例：**48 條**，每條標註頁碼/圖。
- 待釐清：見 A8（DOC_TYPE_IND 擴充系列、CTSTransmissionID/OriginalMessageRefID 大小寫、override 行為）。
