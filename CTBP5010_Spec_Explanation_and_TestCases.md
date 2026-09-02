# CTBP5010 Update Document References — 功能規格書解讀 + 測試用例 (Test Cases)

> 對應文件：`Functional Specification – CTBP5010 Update Document References v1.0 (working, 2026-06-17)`，全文 **3 頁**，含 **1 張 Program Flow 圖**（EMF 向量圖，Claude 已轉檔並成功讀取圖內容）。
> Part A = 逐節解讀（中文）；Part B = 英文 Test Cases（Test Case ID / Test Scenario / Test Data / Expected Result / Remark；Remark 為出處頁碼/圖）。
> **方向（Direction）**：**Outbound 前置（staging）**——把 Portal 中「準備交換」的最新 GIR 記錄複製到交換用的 DOC_REF 表，是整條 outbound 流程的**第一步**（在 CTBP5001 Data Extraction 之前）。

---

## 0. 定位
CTBP5010 是 CTBP5001 整體流程圖（Overall Workflow）的**第 1 步**：把 Pillar Two Portal 中「ready for exchange」的最新 GloBE 資訊申報（GIR）記錄複製進本系統的 `*_DOC_REF` / `CTS_*_DOC_REF` 表，供後續 CTBP5001 抽數使用。下游＝CTBP5024 報表 / CTBP5001。

### 術語速查
| 詞 | 義 |
|---|---|
| **四段 DOC_REF** | `GSEC_DOC_REF`(General Section)、`JSEC_DOC_REF`(Jurisdiction Section)、`SMRY_DOC_REF`(Summary)、`UTPR_DOC_REF`(UTPR Attribution) |
| **CTS_*_DOC_REF** | 交換傳輸用的對應 DOC_REF 表（GSEC/JSEC/SMRY/UTPR） |
| **Pillar Two Portal** | GIR 申報來源系統 |

---

# Part A — 逐節內容解讀

## A1. Change History（p.1）
單筆：`2026-04-09 Initial version`。v1.0 初版。

## A2. Batch Job Properties（p.1）
- **Batch Job ID**：`CTBP5010`
- **Frequency**：**（空白）**——無固定排程頻率
- **Ad-hoc Run**：`Supported`
- **Ad-hoc Parameters**：**（空白，無參數）**

## A3. Description（p.1）
本程式將 Pillar Two Portal 中**準備交換（ready for exchange）**的最新 GloBE 資訊申報記錄複製過來。

## A4. Program Flow（p.2，Program Flow 圖）★ Claude 已讀取圖內容
順序呼叫四支程序，各自從 Pillar Two Portal 複製對應段落記錄：
1. Start
2. Call procedure **`GSEC_DOC_REF`** — copy **General Section** records from Pillar Two Portal
3. Call procedure **`JSEC_DOC_REF`** — copy **Jurisdiction Section** records
4. Call procedure **`SMRY_DOC_REF`** — copy **Summary** records
5. Call procedure **`UTPR_DOC_REF`** — copy **UTPR Attribution** records
6. End

## A5. Program Logic — Database Records（p.2–3）
| 動作 | 表 | 內容 |
|---|---|---|
| Create | `CTS_GSEC_DOC_REF` / `CTS_JSEC_DOC_REF` / `CTS_SMRY_DOC_REF` / `CTS_UTPR_DOC_REF` | `TRANSMISSION_FLAG=NULL`、`LAST_FCE_TIN=NULL`、`LAST_DOC_REF_ID=NULL`、`LAST_CTS_DOC_REF_ID=NULL` |
| Create | `GSEC_DOC_REF` / `JSEC_DOC_REF` / `SMRY_DOC_REF` / `UTPR_DOC_REF` | （由上述四支程序自 Portal 複製建立） |

## A6. 待釐清處
1. **Frequency 空白**：未定排程；實務上應由上游觸發或人工 ad-hoc。建議確認觸發方式。
2. **兩組 Create 的關係**：內文表列了兩個 Create（`CTS_*_DOC_REF` 旗標全 NULL、以及 `*_DOC_REF`）。建議確認先建 `*_DOC_REF` 再建對應 `CTS_*_DOC_REF`（或反之）的順序與關聯鍵。

---

# Part B — Test Cases (English)

> Format: **Test Case ID | Test Scenario | Test Data | Expected Result | Remark**. Direction = **Outbound-prep** for all. "p.N" = page N; "Flow" = Program Flow diagram (p.2).

## B1. Batch Properties & Trigger

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-10-001 | Ad-hoc run supported (no parameter) | Ad-hoc trigger, no parameter | Program runs and copies ready-for-exchange records | p.1 |
| TC-10-002 | No fixed schedule configured | Frequency field is blank | Program is triggered on demand (confirm trigger mechanism) | p.1 (A6-1) |

## B2. Copy Records from Pillar Two Portal (Program Flow)

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-10-003 | Copy only ready-for-exchange records | Portal has GIR records flagged ready for exchange + some not ready | Only ready-for-exchange records are copied | p.1, Flow |
| TC-10-004 | GeneralSection procedure runs | Portal General Section records ready | GSEC_DOC_REF procedure copies General Section records | p.2, Flow |
| TC-10-005 | JurisdictionSection procedure runs | Portal Jurisdiction Section records ready | JSEC_DOC_REF procedure copies Jurisdiction Section records | p.2, Flow |
| TC-10-006 | Summary procedure runs | Portal Summary records ready | SMRY_DOC_REF procedure copies Summary records | p.2, Flow |
| TC-10-007 | UTPRAttribution procedure runs | Portal UTPR Attribution records ready | UTPR_DOC_REF procedure copies UTPR Attribution records | p.2, Flow |
| TC-10-008 | All four procedures run in sequence | All four sections have ready records | GSEC → JSEC → SMRY → UTPR executed in order | p.2, Flow |

## B3. Database Records

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-10-009 | CTS_*_DOC_REF created with NULL flags | New records copied | CTS_GSEC/JSEC/SMRY/UTPR_DOC_REF created with TRANSMISSION_FLAG=NULL, LAST_FCE_TIN=NULL, LAST_DOC_REF_ID=NULL, LAST_CTS_DOC_REF_ID=NULL | p.2–3 |
| TC-10-010 | Section DOC_REF tables created | New records copied | GSEC_DOC_REF / JSEC_DOC_REF / SMRY_DOC_REF / UTPR_DOC_REF records created | p.2–3 |
| TC-10-011 | Four CTS_*_DOC_REF tables all covered | Records across all four sections | All four CTS_*_DOC_REF tables receive records (not a subset) | p.2–3 |

## B4. Edge Cases

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-10-012 | No ready-for-exchange records in Portal | Portal has no ready records | Program completes; no DOC_REF records created | p.1, Flow |
| TC-10-013 | Only some sections have records | Only General Section ready | Only GSEC_DOC_REF / CTS_GSEC_DOC_REF populated; others empty | p.2, Flow |
| TC-10-014 | Re-run does not duplicate already-copied records | Records already copied earlier | Confirm idempotency / no duplicate copy (verify with BA) | p.1 (A6-2) |

---

### 統計
- 章節：Change History / Batch Job Properties / Description / Program Flow（圖，已判讀）/ Program Logic (Database Records) 全數覆蓋。
- 測試用例：**14 條**，每條標註頁碼/圖。
- 待釐清：見 A6（觸發頻率、兩組 Create 順序/關聯）。
