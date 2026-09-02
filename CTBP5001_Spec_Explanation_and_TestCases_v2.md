# CTBP5001 Data Extraction — 功能規格書解讀 + 測試用例 (Test Cases) — v2

> 對應文件：`Functional Specification – CTBP5001 Data Extraction v1.0 (working, 2026-07-20)`，全文 **27 頁**，另含 **4 張流程圖**（Overall Workflow / Application Flow / GIR 計數 / Correction 計數）。
>
> **v2 更新重點**：
> 1. 所有測試年度 **2024 → 2025**（RPT_YR、DocRefId、ReportingPeriod、Ad-hoc 參數皆已更新）。
> 2. 新增 **Scope & Direction（Inbound / Outbound）** 說明。
> 3. 依 **4 張流程圖** 新增 4 組測試（B18–B21，共 34 條）。
> 4. 補上圖 3 / 圖 4 揭露的 **代碼不一致**（OECD_10/11/12/13、GIR_101/102）到待釐清清單。

---

## 0. 背景與定位

CTBP5001 是香港稅務局 Pillar Two（BEPS 2.0 全球最低稅）資料交換系統中的一支 **批次程式**。職責：把資料庫內「待交換（to-be-exchanged）」的 GIR 記錄抽出，產生符合 OECD GIR XML Schema 的 XML 檔，再寫回資料庫；產出檔之後交由下游程式審批、簽章加密、傳輸。

### 術語表

| 術語 | 含義 |
|---|---|
| **GIR** | GloBE Information Return，全球資訊申報表 |
| **UPE / DFE** | Ultimate Parent Entity（最終母公司）/ Designated Filing Entity（指定申報實體）。只有這兩類實體提交的 GIR 才會被抽取 |
| **五個元素** | Filing Info、General Section、Summary、Jurisdiction Section、UTPR Attribution（皆可獨立更正） |
| **DocTypeIndic** | 文件類型：`OECD0`=重送、`OECD1`=新增、`OECD2`=更正、`OECD3`=刪除（SQL 寫 `OECD_0`～`OECD_3`；計數圖另見 `OECD_10`～`OECD_13`，見 A9） |
| **MessageTypeIndic** | 訊息類型：`GIR101`=新增記錄檔、`GIR102`=更正/申報實體更正檔（計數圖寫 `GIR_101`/`GIR_102`） |
| **FCE_TIN** | 申報成員實體稅務編號 |
| **REC_JDX_CTRY** | 接收方稅務管轄區代碼 |
| **RPT_YR** | 申報年度 |
| **HK** | 傳輸方國家（固定 HK） |
| **MessageRefId** | XML 檔唯一訊息參考編號 |
| **GLOBEBody** | GIR XML 內承載五元素的主體容器（見圖 2） |

### 0.1 Scope & Direction（Inbound / Outbound）★ 本次新增

**CTBP5001 是「純 Outbound（對外送出）」程式。** 依據：

1. MessageSpec `TransmittingCountry` 恆為 `HK`（HK 永遠是發送方）；
2. 交換夥伴判定用 SQL1 的 `OUT_DATE_STARTED / OUT_DATE_ENDED`——**outgoing 出向**有效期；
3. 整體流程（圖 1）為 HK 產檔 → 簽章加密 → 經 SFTP **傳出** OECD；
4. CTBP5001 範圍內**無 inbound 資料記錄處理**；唯一 inbound 是流程尾端收**狀態訊息（status message）**，由 **CTBP5004** 負責，屬邊界、非本程式。

**測試方向約定（本檔通用）**：
- 未特別標註者，**一律 Outbound**。
- 群組標題以 `[Outbound]` / `[Boundary]` 標示。
- 個別「收狀態回程」的用例，於 Test Scenario 開頭標 `[Inbound-Status]`。

> 若貴專案另有處理「HK 接收他國交換過來之 GIR」的 inbound 程式，該部分**不在 CTBP5001 規格內**，需另一份規格與另一套 inbound test case。

---

# Part A — 逐節內容解讀

## A1. Change History（第 1 頁）
版本紀錄。目前僅一筆：`2026-04-14 Initial version`（CRF/UQF 欄為「--」）。v1.0 初版工作稿。

## A2. Batch Job Properties（第 1 頁）
- **Batch Job ID**：`CTBP5001`
- **Frequency**：`Ad-hoc`（按需執行）
- **Ad-hoc Run**：`Supported`
- **Ad-hoc Parameters**：位置 #1，格式 `XX 9999` = **Tax Jurisdiction + 空格 + Fiscal Year**（例 `US 2025`）

## A3. Description（第 1–2 頁）
三條核心業務規則：

1. **抽取範圍**：只有 **UPE / DFE** 提交的 GIR 才抽。每個父元素依接收管轄區去查 **Jurisdiction Table**，同時滿足才抽：該管轄區在表內，且 **Start Date ≤ 申報年度起始日 ≤ Cessation Date**。
2. **更正合併**：五元素皆可更正。更正已接受的 GIR 不需重送整份，只需 Filing Info（帶 `OECD0`）＋被更正元素。抽取前若已更正，抽取採更正版。範例：Original File（Day 1，五元素）＋ Corrected File（Day 2，僅改 Jurisdiction Section）→ 四元素取自 Original、Jurisdiction Section 取自 Corrected。
3. **輸出檔類型**（皆須符合 OECD GIR XML Schema）：(i) Filing entity correction XML、(ii) New records XML、(iii) Correction and Deletion Records XML。不得含 `--`、`/*`、`&#`。

## A4. Overall Workflow of Data Extraction（第 2 頁，對應**圖 1**）★ 已據圖補全
三泳道 outbound 端到端流程 + 狀態回程：

- **泳道 1（產檔與審批）**：Start → **CTBP5010** Update Document References → 產報表 **CTBP5024**（Analysis of To-be-Exchanged Pillar Two Reports）→ **CTBP5001** Data Extraction → 產報表 **CTBP5025**（Pending Transmission to Pillar Two Partners）→ **CTAP5002** Approve Transmission → **Approve or Reject?**（Reject 迴圈返回；Approve 續下）
- **泳道 2（簽章與傳輸）**：**CTBP5002** Sign & Encrypt of Data Packet → 產報表 **CTBP5026** → **Transmit Data Packets to OECD via SFTP** → **CTBP5011** Update Transmission Status → Partners 下載並處理 → Partners 經 CTS 回傳 Status Message
- **泳道 3（收狀態）**：Download Status Message Packets from CTS via SFTP → **CTBP5004** Receive Status Message → End

> 定位：CTBP5001 位於泳道 1 第 3 步；其上游是 CTBP5010，下游是 CTAP5002/CTBP5002/…。

## A5. Application Flow（第 3 頁，對應**圖 2**）★ 已據圖補全
CTBP5001 核心決策樹：

1. **START** → **Check input parameter?**
   - **No** → No input parameter → **Retrieve ALL** FilingCE TINs that have to-be-exchanged records, 連同其 fiscal years 與 tax jurisdictions（Auto）
   - **Yes** → With input parameter 'Tax Jurisdiction and Fiscal Year' → 依參數 **find specified** FilingCE TINs（Ad-hoc）
2. **Group the records by FilingCE TINs, Fiscal Year and Tax Jurisdiction**
3. **For each XML file**：Generate an XML element **'MessageSpec' for each combination of 'Tax Jurisdiction' and 'Fiscal Year'**
4. **Loop for each record of 'FilingCE TIN' + Fiscal Year + Tax Jurisdiction**：
   - **Any changes on FilingInfo?** Yes → **Filing Entity Correction**：GLOBEBody with **'FilingInfo' only**，No GeneralSection / Summary / JurisdictionSection / UTPRAttribution
   - No → **Any new records?** Yes → **New Records**：GLOBEBody with 'FilingInfo' ＋ 四段的 **new** records
   - No → **Any correction records?** Yes → **Correction Records**：GLOBEBody with 'FilingInfo' ＋ 四段的 **correction** records
   - No → **End of records?** No 迴圈返回；Yes → **END**

## A6. Program Logic（第 4–14 頁）

### 兩種模式（第 4 頁）
- **Auto**（無參數）：掃全庫，取所有待交換記錄，彙整 TIN/Jurisdiction/year 清單。
- **Ad-hoc**（有參數）：僅針對指定 TIN/Jurisdiction/year。

### 待交換判定條件（第 4 頁）
須全部滿足：① 來自 Portal「Accepted」且 active；② 無 error/warning；③ 尚未送出；④ 目標管轄區在 `PTP_TAX_JRDT` 有效期內；⑤ 前訊息已傳、狀態訊息未收到者**不抽**；⑥ 按 FCE_TIN、接收管轄區、reporting year 分組。

### 分組與產檔順序（第 4 頁）
先按 **reporting year → FilingCE TIN → recipient country** 分組，再按固定順序產檔：**Filing Entity Correction（GIR102）→ New Records（GIR101）→ Correction Records（GIR102）**。

### OECD0 重送與 30,000 筆上限（第 4 頁）
- **OECD0**：僅當該 year/TIN/country 的 Filing Info **先前已送過**方可用。
- **30,000 上限**：四段合計超過須拆檔；**第一檔 FilingInfo 用 OECD1，之後每檔 OECD0，且 OECD0 的 DocRefId 與 OECD1 相同**。

### A. Retrieve To-be-exchanged records（第 4 頁）
- 來源 **SQL7**；每檔唯一 MessageRefId；序號 `<9999>` 由 **0001** 起，每個 (Tax Jurisdiction, Fiscal Year, FilingCE TIN) 組合**重置**。

### B. Filing entity correction XML（GIR102）（第 5 頁）
- 來源 **SQL6**，取最新 `OECD2` FilingInfo；**不含**四段元素。
- **MessageSpec**：TransmittingCountry=`HK`、ReceivingCountry=`REC_JDX_CTRY`、MessageType=`GIR`、MessageTypeIndic=`GIR102`、DocRefId=`HK+RPT_YR+REC_JDX_CTRY+FCE_TIN+'-'+<yyyyMMdd>+'-'+<9999>`、ReportingPeriod=`RPT_YR+'-12-31'`。
- **FilingInfo**：DocTypeIndic=`OECD2`、DocRefId=`…+'-F-'+<yyyyMMdd>`、CorrDocRefId=`V_LAST_FILING_INFO.FI_DOC_REF_ID`；內容取自 `V_FILING_INFO_OECD2.FI_ID`。

### C. New records XML（GIR101）（第 6–7 頁）
- 來源 **SQL3**；四段取自 `V_EXT_GIR101` 的 `GS_ID/JS_ID/SMRY_ID/UPAT_ID`。
- **MessageSpec**：MessageTypeIndic=`GIR101`。
- **FilingInfo 分支**：不存在於 `V_LAST_FILING_INFO`→`OECD1`（新 DocRefId `-F-`）；存在→`OECD0`（DocRefId 取自 `V_LAST_FILING_INFO.FI_DOC_REF_ID`）。
- **四段**：DocRefId 後綴 `-G-/-J-/-S-/-U-`，來源表 `PTP_GEN_SEC/PTP_JDX_SEC/PTP_SMRY/PTP_UTPR_ATTR`。

### D. Correction records XML（GIR102）（第 8–9 頁）
- 來源 **SQL4**（含 DELETE 分支，OECD3）；四段取自 `V_EXT_GIR102`。
- **MessageSpec**：MessageTypeIndic=`GIR102`；**FilingInfo 分支同 C**。
- **四段**：DocTypeIndic=`OECD2`/`OECD3`，且多 `CorrDocRefId`=各 view 的 `LAST_CTS_DOC_REF_ID`。

### E. Database Update（第 10–14 頁，計數對應**圖 3 / 圖 4**）
- 用 stored procedure **`UPDATE_TX_CNT`** 更新 `GIR_MSG_SPEC.GIR_CNT`（圖 3 邏輯）與 `GIR_MSG_SPEC.AMEND_CNT`（圖 4 邏輯）。
- 寫入 `GIR_*` 起始表：GIR_MSG_SPEC（`APRVD_STS='P'`、`SGD_STS=NULL`）、FILING_INFO、GEN_SEC、JDX_SEC、SMRY、UTPR_ATTR。六表共通：`LAST_UPD_AT/CREATED_AT=當前時間`、`LAST_UPD_BY/CREATED_BY=2`。

**圖 3（GIR 計數）條件**：`MessageTypeIndic='GIR_101' AND FilingInfo.DocTypeIndic IN ('OECD_1','OECD_11')` → **GIR Count +1**。
**圖 4（Correction 計數）任一成立 +1**：
- (a) `MessageTypeIndic='GIR_102'`；或
- (b) `GIR_101 AND FilingInfo.DocTypeIndic IN ('OECD_2','OECD_12','OECD_3','OECD_13')`；或
- (c) `GIR_101 AND FilingInfo.DocTypeIndic IN ('OECD_0','OECD_10') AND (GeneralSection 或 Summary 或 JurisdictionSection 或 UTPRAttribution).DocTypeIndic IN ('OECD_1','OECD_11')`。

### F. Validate XML File(s)（第 14 頁）
- 字元轉義：`&`→`&amp;`、`<`→`&lt;`、`>`→`&gt;`、`'`→`&apos;`、`"`→`&quot;`。
- 待 **CTAP5002** 審批的檔放置：Payload `\data\out\pending\<MessageRefId>.xml`、Metadata `\data\out\pending\<MessageRefId>.metadata.xml`。

## A7. Appendix 1（第 16–25 頁）— SQL Views

| 視圖 | 用途 | 頁 |
|---|---|---|
| **SQL1** `V_CTRY_PARTNER` | 依 `PTP_TAX_JRDT` outgoing 起訖日判定有效交換夥伴（`IS_PARTNER='Y'`） | 16 |
| **SQL2** `V_CTRY_TRANSMISSION` | 仍在傳輸、狀態未收到的記錄（`TRANSMISSION_FLAG='Y'`），用於排除 | 17 |
| **SQL3** `V_EXT_*_GIR101` | `OECD_1` 四段新增記錄 | 17–18 |
| **SQL4** `V_EXT_*_GIR102` | `OECD_2/OECD_3` 四段更正/刪除（含 UPDATE/DELETE 分支） | 18–23 |
| **SQL5** `V_LAST_FILING_INFO` | 最後一次 `ACCEPTED` 的 FilingInfo（按 RPT_YR/RX_CTRY/FCE_TIN 取最新） | 24 |
| **SQL6** `V_FILING_INFO_OECD2` | 自上次 `ACCEPTED` 後被更正的 FilingInfo | 24 |
| **SQL7** `V_EXT` | 待交換 FY/Jurisdiction/TIN（且對方為有效夥伴） | 25 |

## A8. Appendix 2（第 26–27 頁）— 跨程式狀態流轉

| 步驟／程式 | 動作 | 關鍵欄位 | 頁 |
|---|---|---|---|
| 1. CTBP5010 Update Document Reference | Create | 建 `*_DOC_REF`；`CTS_*_DOC_REF` 旗標=NULL | 26 |
| 2. **CTBP5001 Data Extraction** | Create | `GIR_MSG_SPEC.APRVD_STS='P'`；建五元素表 | 26 |
| 3. CTAP5002 Approve Transmission | Update | 批准 `APRVD_STS='A'`/`SGD_STS='P'`；拒絕 `APRVD_STS='R'`/`SGD_STS=NULL` | 27 |
| 4. CTBP5002 Sign & Encrypt | Update/Create | `SGD_STS='S'`；`CTS_TX_REC.TX_STS='R'`；`TRANSMISSION_FLAG='Y'` | 27 |
| 5. CTBP5011 Update Transmission Status | Update | `TX_STS`：T/N/U/F | 27 |
| 6. CTBP5004 Receive Status Message | Update | 收狀態後 `TRANSMISSION_FLAG=NULL`、清旗標 → 流程完成 | 27 |

## A9. 規格書可疑點 / 待釐清處

1. **★ 代碼系列不一致（最重要）**：正文/SQL 用 `OECD0–3`（`OECD_0–3`）與 `GIR101/102`；但**圖 3、圖 4** 用 `OECD_1/OECD_11`、`OECD_2/OECD_12`、`OECD_3/OECD_13`、`OECD_0/OECD_10` 及 `GIR_101/GIR_102`。多出的 `OECD_1X` 系列在正文完全未定義。**須確認**：權威代碼集為何？`OECD_1X` 實際會否出現？否則 GIR_CNT/AMEND_CNT 計數可能錯。
2. **★ MessageSpec 粒度**：圖 2 說「per combination of Tax Jurisdiction and Fiscal Year」，但分組/DocRefId 又含 FCE_TIN。**須確認**一個檔是否可含多個 TIN，或嚴格「一個 (TIN, FY, Jurisdiction) 一個 MessageSpec/檔」。
3. **視圖名 typo**：SQL5 建的視圖叫 `V_LAST_FILING_INO`（少 F），他處引用 `V_LAST_FILING_INFO`。
4. **欄位別名不一致**：`RX_CTRY`(SQL5) vs `REC_JDX_CTRY`；`TX_CTRY` vs `TransmittingCountry`。
5. **30,000 邊界**：「no more than 30,000」→「剛好 30,000」應歸單檔。
6. **轉義 vs 不可接受字元**：p.2 禁 `&#`、p.14 又要把 `&` 轉 `&amp;`，須確認處理次序不會產生/消除 `&#`。
7. **第 15 頁空白**：疑分頁殘留。
8. **Ad-hoc 參數格式校驗**：僅給 `XX 9999`，未述格式錯誤時行為。
9. **Direction**：確認 CTBP5001 為 outbound-only、inbound 資料程式另有其人（見 0.1）。

---

# Part B — Test Cases (English)

> Format: **Test Case ID | Test Scenario | Test Data | Expected Result | Remark (source page)**
> **Direction convention:** every case is **OUTBOUND** unless its group is tagged `[Boundary]` or the row's scenario begins with `[Inbound-Status]`. "p.N" = page N; "Img N" = flowchart image N.
> **Year convention (v2):** all fiscal/reporting years use **2025**; `<yyyyMMdd>` in DocRefId is the extraction **run date** (example uses 20260720).

## B1. Record Selection — To-be-exchanged Criteria `[Outbound]`

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-SEL-001 | Extract only "Accepted" and active GIR records | GIR file, status="Accepted", IS_ACTIVE='Y', RPT_YR=2025 | Record included in to-be-exchanged list | p.4 |
| TC-SEL-002 | Exclude records not in "Accepted" status | GIR file, status=Pending/Rejected, RPT_YR=2025 | Record NOT included | p.4 |
| TC-SEL-003 | Exclude records with error or warning during filing | 2025 filing with an error or warning flag | Record NOT extracted | p.4 |
| TC-SEL-004 | Exclude records already sent | 2025 record already transmitted | Record NOT extracted (only not-yet-sent picked) | p.4 |
| TC-SEL-005 | Include record whose target jurisdiction is within effective period | REC_JDX_CTRY with OUT_DATE_STARTED ≤ SYSDATE and (OUT_DATE_ENDED ≥ SYSDATE or NULL) | IS_PARTNER='Y'; record extracted | p.4, p.16 (SQL1) |
| TC-SEL-006 | Exclude record whose jurisdiction is outside effective period | OUT_DATE_STARTED in future OR OUT_DATE_ENDED past | IS_PARTNER='N'; record NOT extracted | p.4, p.16 |
| TC-SEL-007 | Exclude records awaiting a status message from a prior transmission | Record transmitted before; CTS status not yet received (TRANSMISSION_FLAG='Y') | Record for that RPT_YR + REC_JDX_CTRY NOT extracted | p.4, p.17 (SQL2) |
| TC-SEL-008 | Group extracted files by TIN + recipient jurisdiction + reporting year | 2 different FCE_TINs, same jurisdiction, RPT_YR=2025 | Separate XML file per (TIN, jurisdiction, year) | p.4 |
| TC-SEL-009 | Exchange keys sourced from SQL7 | Records ready across several jurisdiction/TIN, RPT_YR=2025 | Key list derived from V_EXT (SQL7) | p.4, p.25 |

## B2. Jurisdiction Table Date-Window Check `[Outbound]`

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-JDX-001 | Extract when reporting FY start is inside jurisdiction window | Start Date ≤ 2025 FY Start ≤ Cessation Date | Element extracted to that jurisdiction | p.1 |
| TC-JDX-002 | Do not extract when FY start before jurisdiction start | 2025 FY Start < Jurisdiction Start Date | Element NOT extracted | p.1 |
| TC-JDX-003 | Do not extract when FY start after cessation | 2025 FY Start > Cessation Date | Element NOT extracted | p.1 |
| TC-JDX-004 | Skip jurisdiction absent from Jurisdiction Table | Receiving jurisdiction not in table | Element NOT extracted | p.1 |
| TC-JDX-005 | Only UPE/DFE-filed GIRs subject to extraction | GIR filed by neither UPE nor DFE | GIR NOT subject to extraction | p.1 |
| TC-JDX-006 | Multi-jurisdiction element evaluated per jurisdiction | Element to Jurisdiction A (in-window) + B (out-of-window), RPT_YR=2025 | Extracted for A only | p.1 |

## B3. Correction-Before-Extraction Merge `[Outbound]`

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-COR-001 | Merge corrected element with remaining originals | Original File (Day 1, 5 elements) + Corrected File (Day 2, Jurisdiction Section), RPT_YR=2025 | FilingInfo/GeneralSection/Summary/UTPRAttribution from Original; JurisdictionSection from Corrected | p.1 |
| TC-COR-002 | Accept partial correction (FilingInfo OECD0 + element) | Correction submission: FilingInfo(OECD0)+1 element, no full GIR | Partial correction accepted; extraction reflects it | p.1 |
| TC-COR-003 | Multiple elements corrected across files | Original + Corrected (Summary AND UTPRAttribution), RPT_YR=2025 | Corrected Summary & UTPRAttribution from Corrected; rest from Original | p.1 |

## B4. Program Modes `[Outbound]`

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-MODE-001 | Auto mode (no parameter) | Triggered with no parameter | Scans DB, retrieves ALL to-be-exchanged records; summarizes TIN/Jurisdiction/2025 | p.4, Img 2 |
| TC-MODE-002 | Ad-hoc mode with valid parameter | Parameter = "US 2025" (format "XX 9999") | Compiles list only for US, FY2025 | p.1, p.4, Img 2 |
| TC-MODE-003 | Ad-hoc parameter format handling (invalid) | Parameter = "USA2025" (bad format) | Treated as invalid / not processed — confirm behaviour | p.1 (A9-8) |
| TC-MODE-004 | Ad-hoc selects correct subset only | Parameter = "JP 2025"; DB also holds US/2025 records | Only JP/FY2025 compiled; US ignored | p.1, p.4 |

## B5. File Generation Order & Grouping `[Outbound]`

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-ORD-001 | File order within one group | 2025 group needing all three file types | Order: Filing Entity Correction (GIR102) → New Records (GIR101) → Correction Records (GIR102) | p.4 |
| TC-ORD-002 | Grouping precedes file generation | Mixed records across years/TINs/countries | Grouped by year → TIN → recipient country, then files generated | p.4, Img 2 |

## B6. OECD0 Resend Option `[Outbound]`

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-RES-001 | OECD0 resend permitted when FilingInfo already sent | FilingInfo previously sent for same 2025/TIN/country | OECD0 allowed for FilingInfo | p.4 |
| TC-RES-002 | OECD0 restricted to FilingInfo element | Attempt OECD0 on GeneralSection/Summary/etc. | OECD0 used ONLY for FilingInfo | p.4 |

## B7. 30,000-Record Cap & File Splitting `[Outbound]`

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-CAP-001 | Single file when ≤ 30,000 | 25,000 records across 4 sections, RPT_YR=2025 | One XML file | p.4 |
| TC-CAP-002 | Split when > 30,000 | 65,000 records across 4 sections | Multiple files, each ≤ 30,000 | p.4 |
| TC-CAP-003 | FilingInfo option across split files | Run split into 3 files | 1st file FilingInfo=OECD1; subsequent=OECD0; OECD0 DocRefId = OECD1 DocRefId | p.4 |
| TC-CAP-004 | Boundary — exactly 30,000 | 30,000 records | Single file (no split) — confirm boundary | p.4 (A9-5) |

## B8. MessageRefId & Sequence Number `[Outbound]`

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-MRI-001 | Unique MessageRefId per file | Multiple files in one run | Each file has a distinct MessageRefId | p.4 |
| TC-MRI-002 | Sequence starts at 0001 | First file for (US, 2025, T123) | Sequence <9999> = 0001 | p.4 |
| TC-MRI-003 | Sequence resets per key combination | New (Jurisdiction, 2025, TIN) combination | Sequence resets to 0001 | p.4 |
| TC-MRI-004 | Sequence increments within same combination | 3 files for (US, 2025, T123) | Sequence = 0001, 0002, 0003 | p.4 |

## B9. Filing Entity Correction XML (GIR102) — Section B `[Outbound]`

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-FEC-001 | FilingInfo from latest OECD2 record | SQL6 returns latest OECD2 FilingInfo, RPT_YR=2025 | FilingInfo generated from that record | p.5, p.24 |
| TC-FEC-002 | No section elements present | A filing-entity-correction file | No GeneralSection/Summary/JurisdictionSection/UTPRAttribution | p.5, Img 2 |
| TC-FEC-003 | MessageSpec values for GIR102 | RPT_YR=2025, REC_JDX_CTRY=US, FCE_TIN=T123, run 20260720, seq 0001 | TransmittingCountry=HK; ReceivingCountry=US; MessageType=GIR; MessageTypeIndic=GIR102; DocRefId=HK2025UST123-20260720-0001; ReportingPeriod=2025-12-31 | p.5 |
| TC-FEC-004 | FilingInfo OECD2 fields | Same group | DocTypeIndic=OECD2; DocRefId=HK2025UST123-F-20260720; CorrDocRefId=V_LAST_FILING_INFO.FI_DOC_REF_ID | p.5 |
| TC-FEC-005 | Source of FilingInfo XML content | FI_ID in V_FILING_INFO_OECD2 | Content retrieved from FI_ID in V_FILING_INFO_OECD2 | p.5 |

## B10. New Records XML (GIR101) — Section C `[Outbound]`

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-NEW-001 | MessageSpec for GIR101 | RPT_YR=2025, US, T123 | MessageTypeIndic=GIR101; other fields per spec (2025-12-31 etc.) | p.6 |
| TC-NEW-002 | FilingInfo=OECD1 when no prior FilingInfo | (2025, T123, US) NOT in V_LAST_FILING_INFO | DocTypeIndic=OECD1; new DocRefId `-F-` | p.6 |
| TC-NEW-003 | FilingInfo=OECD0 when prior FilingInfo exists | (2025, T123, US) found in V_LAST_FILING_INFO | DocTypeIndic=OECD0; DocRefId from V_LAST_FILING_INFO.FI_DOC_REF_ID | p.6 |
| TC-NEW-004 | GeneralSection element (GIR101) | GS_ID in V_EXT_GSEC_GIR101 | XML from PTP_GEN_SEC; DocRefId uses '-G-' | p.6 |
| TC-NEW-005 | JurisdictionSection element (GIR101) | JS_ID in V_EXT_JSEC_GIR101 | XML from PTP_JDX_SEC; DocRefId uses '-J-' | p.7 |
| TC-NEW-006 | Summary element (GIR101) | SMRY_ID in V_EXT_SMRY_GIR101 | XML from PTP_SMRY; DocRefId uses '-S-' | p.7 |
| TC-NEW-007 | UTPRAttribution element (GIR101) | UPAT_ID in V_EXT_UTPR_GIR101 | XML from PTP_UTPR_ATTR; DocRefId uses '-U-' | p.7 |
| TC-NEW-008 | New records sourced from V_EXT_GIR101 (SQL3) | Records classified GIR101 (OECD_1), RPT_YR=2025 | Four-section content via GS_ID/JS_ID/SMRY_ID/UPAT_ID in V_EXT_GIR101 | p.6, p.17 |

## B11. Correction & Deletion Records XML (GIR102) — Section D `[Outbound]`

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-CRC-001 | MessageSpec for GIR102 correction file | RPT_YR=2025 group | MessageTypeIndic=GIR102; fields per spec | p.8 |
| TC-CRC-002 | FilingInfo=OECD1 when no prior FilingInfo | (2025, T123, US) NOT in V_LAST_FILING_INFO | DocTypeIndic=OECD1; new DocRefId | p.8 |
| TC-CRC-003 | FilingInfo=OECD0 when prior FilingInfo exists | (2025, T123, US) found in V_LAST_FILING_INFO | DocTypeIndic=OECD0; DocRefId from V_LAST_FILING_INFO | p.8 |
| TC-CRC-004 | GeneralSection correction carries CorrDocRefId | GS_ID in V_EXT_GSEC_GIR102 | DocTypeIndic=OECD2/OECD3; DocRefId '-G-'; CorrDocRefId=LAST_CTS_DOC_REF_ID | p.8 |
| TC-CRC-005 | JurisdictionSection correction | JS_ID in V_EXT_JSEC_GIR102 | DocTypeIndic=OECD2/OECD3; DocRefId '-J-'; CorrDocRefId set | p.9 |
| TC-CRC-006 | Summary correction | SMRY_ID in V_EXT_SMRY_GIR102 | DocTypeIndic=OECD2/OECD3; DocRefId '-S-'; CorrDocRefId set | p.9 |
| TC-CRC-007 | UTPRAttribution correction | UPAT_ID in V_EXT_UTPR_GIR102 | DocTypeIndic=OECD2/OECD3; DocRefId '-U-'; CorrDocRefId set | p.9 |
| TC-CRC-008 | Deletion record (OECD3) extraction | Record in V_EXT_*_GIR102 DELETE branch (OECD_3), RPT_YR=2025 | Extracted into correction & deletion file, DocTypeIndic=OECD3 | p.8–9, p.19 |

## B12. Database Update — Section E `[Outbound]`

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-DBU-001 | GIR_MSG_SPEC insert values | A 2025 GIR message generated | APRVD_STS='P'; SGD_STS=NULL; GIR_CNT/AMEND_CNT by UPDATE_TX_CNT | p.12 |
| TC-DBU-002 | GIR_CNT updated by stored procedure | GIR101 records inserted | GIR_MSG_SPEC.GIR_CNT updated via UPDATE_TX_CNT (see B19) | p.10, p.12 |
| TC-DBU-003 | AMEND_CNT updated by stored procedure | GIR102 records inserted | GIR_MSG_SPEC.AMEND_CNT updated via UPDATE_TX_CNT (see B20) | p.11, p.12 |
| TC-DBU-004 | FILING_INFO field mapping | FilingInfo element generated | FILING_INFO_ID=Oracle Seq; DOC_REF_ID/CORR_DOC_REF_ID/DOC_TYPE_IND from /FilingInfo/DocSpec/*; FCE_TIN=V_EXT.FCE_TIN; MS_ID=GIR_MSG_SPEC.MS_ID | p.12 |
| TC-DBU-005 | GEN_SEC field mapping | GeneralSection inserted | ID=Oracle Seq; PTP_GS_ID/PTP_FI_ID from V_EXT_GSEC_GIR101/102; DocSpec mapped; FILING_INFO_ID=FILING_INFO.FILING_INFO_ID | p.13 |
| TC-DBU-006 | JDX_SEC field mapping | JurisdictionSection inserted | PTP_JS_ID/PTP_FI_ID from V_EXT_JSEC_GIR101/102; DocSpec mapped | p.13 |
| TC-DBU-007 | SMRY field mapping | Summary inserted | PTP_SMRY_ID/PTP_FI_ID from V_EXT_SMRY_GIR101/102; DocSpec mapped | p.14 |
| TC-DBU-008 | UTPR_ATTR field mapping | UTPRAttribution inserted | PTP_UPAT_ID/PTP_FI_ID from V_EXT_UTPR_GIR101/102; DocSpec mapped | p.14 |
| TC-DBU-009 | Records inserted into GIR_* tables | An XML file generated | Inserted into GIR_* tables + FILING_INFO/GEN_SEC/JDX_SEC/SMRY/UTPR_ATTR | p.12 |
| TC-DBU-010 | Common audit columns populated | Any insert into the six tables | LAST_UPD_AT/CREATED_AT=current datetime; LAST_UPD_BY/CREATED_BY=2 | p.12–14 |

## B13. XML Validation & Character Handling — Section F + Description `[Outbound]`

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-VAL-001 | Ampersand escaped | Content containing `&` | `&` → `&amp;` | p.14 |
| TC-VAL-002 | Less-than escaped | Content containing `<` | `<` → `&lt;` | p.14 |
| TC-VAL-003 | Greater-than escaped | Content containing `>` | `>` → `&gt;` | p.14 |
| TC-VAL-004 | Apostrophe escaped | Content containing `'` | `'` → `&apos;` | p.14 |
| TC-VAL-005 | Quote escaped | Content containing `"` | `"` → `&quot;` | p.14 |
| TC-VAL-006 | Reject unacceptable char — double dash | Payload would contain `--` | Generated file does not contain `--` | p.2 |
| TC-VAL-007 | Reject unacceptable char — slash asterisk | Payload would contain `/*` | Generated file does not contain `/*` | p.2 |
| TC-VAL-008 | Reject unacceptable char — ampersand hash | Payload would contain `&#` | Generated file does not contain `&#` | p.2 |
| TC-VAL-009 | Output conforms to OECD GIR XML Schema | A generated XML file | File validates against OECD GIR XML Schema | p.2 |
| TC-VAL-010 | All three XML file types produced | 2025 run requiring all types | (i) Filing entity correction, (ii) New records, (iii) Correction & Deletion generated | p.2 |
| TC-VAL-011 | Escaping vs unacceptable-char interaction | Content with `&` and a would-be `&#` | Escaping order does not (re)introduce `&#`; passes threat scan — confirm order | p.2, p.14 (A9-6) |

## B14. File Placement / Naming — Section F `[Outbound]`

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-FILE-001 | Payload file placement | Payload for MessageRefId=MR001, pending CTAP5002 | Placed at `\data\out\pending\MR001.xml` | p.14 |
| TC-FILE-002 | Metadata file placement | Metadata for MessageRefId=MR001 | Placed at `\data\out\pending\MR001.metadata.xml` | p.14 |

## B15. DocRefId Construction (Cross-Cutting) `[Outbound]`

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-DRI-001 | MessageSpec DocRefId | RPT_YR=2025, REC_JDX_CTRY=US, FCE_TIN=T123, date=20260720, seq=0001 | `HK2025UST123-20260720-0001` | p.5–9 |
| TC-DRI-002 | FilingInfo DocRefId with '-F-' | Same inputs | `HK2025UST123-F-20260720` | p.5–9 |
| TC-DRI-003 | GeneralSection DocRefId with '-G-' | + V_EXT_GSEC_GIR101.DOC_REF_ID=7 | `HK2025UST123-G-20260720-7` | p.6, p.8 |
| TC-DRI-004 | Section suffix letters correct | Four sections | `-G-`/`-J-`/`-S-`/`-U-` for GeneralSection/JurisdictionSection/Summary/UTPRAttribution | p.6–9 |
| TC-DRI-005 | ReportingPeriod construction | RPT_YR=2025 | ReportingPeriod = `2025-12-31` | p.5–9 |

## B16. Downstream Status Flow (Integration Boundary) — Appendix 2 `[Boundary]`

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-STS-001 | After extraction, GIR_MSG_SPEC pending approval | CTBP5001 completes (2025 run) | APRVD_STS='P' | p.26 |
| TC-STS-002 | Approval sets Approved + pending sign | CTAP5002 approves | APRVD_STS='A'; SGD_STS='P' | p.27 |
| TC-STS-003 | Rejection sets Rejected | CTAP5002 rejects | APRVD_STS='R'; SGD_STS=NULL | p.27 |
| TC-STS-004 | Sign & encrypt marks ready for transmission | CTBP5002 signs & encrypts | SGD_STS='S'; CTS_TX_REC.TX_STS='R'; TRANSMISSION_FLAG='Y' | p.27 |
| TC-STS-005 | Transmission status update values | CTBP5011 updates status | TX_STS ∈ T/N/U/F | p.27 |
| TC-STS-006 | [Inbound-Status] Cycle completes on status message received | CTBP5004 receives status message | TRANSMISSION_FLAG=NULL; DOC_REF flags cleared; extraction process complete | p.27 |

## B17. Negative / Edge Cases from Spec Ambiguities `[Outbound]`

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-EDG-001 | OECD value spelling consistency (output vs comparison) | Element stored DOC_TYPE_IND='OECD_1', RPT_YR=2025 | Emitted DocTypeIndic consistent & schema-valid; comparison aligns — confirm 'OECD1' vs 'OECD_1' vs 'OECD_11' | p.5–9 vs Img 3/4 (A9-1) |
| TC-EDG-002 | View name typo does not break resolution | Reference to last accepted FilingInfo | Resolves correctly (fix `V_LAST_FILING_INO` → `V_LAST_FILING_INFO`) | p.24 (A9-3) |
| TC-EDG-003 | No to-be-exchanged records found | DB has zero qualifying records (Auto, 2025) | Completes gracefully; no file; no DB insert | p.4 |
| TC-EDG-004 | Ad-hoc jurisdiction not an active partner | Parameter jurisdiction with IS_PARTNER='N' | No records extracted for that jurisdiction | p.4, p.16 |

---

## B18. Application Flow — Core Decision Logic (Img 2 / p.3) `[Outbound]` ★ 新增

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-AF-001 | "Check input parameter?" → No branch | Program run with no parameter | Retrieve ALL FilingCE TINs with to-be-exchanged records + their fiscal years & tax jurisdictions | Img 2, p.3 |
| TC-AF-002 | "Check input parameter?" → Yes branch | Parameter "US 2025" (Tax Jurisdiction + Fiscal Year) | Find only the specified FilingCE TINs with to-be-exchanged records for US/2025 | Img 2, p.3 |
| TC-AF-003 | Grouping step | Mixed 2025 records across TIN/FY/jurisdiction | Records grouped by FilingCE TIN, Fiscal Year, Tax Jurisdiction before generation | Img 2, p.3 |
| TC-AF-004 | MessageSpec generation granularity | Two files for US/2025 and JP/2025 | One 'MessageSpec' element generated per combination of Tax Jurisdiction & Fiscal Year (per XML file) — confirm granularity vs TIN | Img 2, p.3 (A9-2) |
| TC-AF-005 | Loop branch 1 — FilingInfo change → Filing Entity Correction | Record with a change on FilingInfo, RPT_YR=2025 | GLOBEBody with 'FilingInfo' ONLY; no GeneralSection/Summary/JurisdictionSection/UTPRAttribution | Img 2, p.3 |
| TC-AF-006 | Loop branch 2 — new records → New Records | Record with no FilingInfo change but new section data | GLOBEBody with 'FilingInfo' + new records of the four sections | Img 2, p.3 |
| TC-AF-007 | Loop branch 3 — correction records → Correction Records | Record with no FilingInfo change, no new, but correction data | GLOBEBody with 'FilingInfo' + correction records of the four sections | Img 2, p.3 |
| TC-AF-008 | Branch precedence (FilingInfo checked first) | Record that has both a FilingInfo change and section changes | Routed to Filing Entity Correction branch (FilingInfo evaluated before new/correction) — confirm intended precedence | Img 2, p.3 |
| TC-AF-009 | Loop termination | "End of records?" reached | No → continue loop to next record; Yes → END | Img 2, p.3 |
| TC-AF-010 | No branch matched | Record with no FilingInfo change, no new, no correction | Record added to no GLOBEBody; loop proceeds to next record | Img 2, p.3 |

## B19. GIR Count Logic — GIR_MSG_SPEC.GIR_CNT (Img 3 / p.10) `[Outbound]` ★ 新增

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-GCNT-001 | GIR_101 + FilingInfo OECD_1 counted | MessageTypeIndic='GIR_101', FilingInfo.DocTypeIndic='OECD_1' | GIR Count + 1 | Img 3, p.10 |
| TC-GCNT-002 | GIR_101 + FilingInfo OECD_11 counted (extended code) | MessageTypeIndic='GIR_101', FilingInfo.DocTypeIndic='OECD_11' | GIR Count + 1 — confirm OECD_11 validity | Img 3, p.10 (A9-1) |
| TC-GCNT-003 | GIR_101 + FilingInfo OECD_0 not counted | MessageTypeIndic='GIR_101', FilingInfo.DocTypeIndic='OECD_0' | NOT counted as GIR | Img 3, p.10 |
| TC-GCNT-004 | GIR_102 not counted as GIR | MessageTypeIndic='GIR_102', any FilingInfo | NOT counted as GIR | Img 3, p.10 |
| TC-GCNT-005 | Multiple FilingInfo in one MessageSpec | 3 FilingInfo: OECD_1, OECD_11, OECD_0 | GIR Count incremented for the OECD_1 and OECD_11 only (=2); loop runs until all processed | Img 3, p.10 |
| TC-GCNT-006 | GIR_CNT persisted correctly | A file with N qualifying FilingInfo | GIR_MSG_SPEC.GIR_CNT (via UPDATE_TX_CNT) = N | Img 3, p.10; p.12 |

## B20. Correction Count Logic — GIR_MSG_SPEC.AMEND_CNT (Img 4 / p.11) `[Outbound]` ★ 新增

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-CCNT-001 | GIR_102 counted as correction | MessageTypeIndic='GIR_102' | Correction Count + 1 | Img 4, p.11 |
| TC-CCNT-002 | GIR_101 + FilingInfo OECD_2 counted | 'GIR_101', FilingInfo.DocTypeIndic='OECD_2' | Correction Count + 1 | Img 4, p.11 |
| TC-CCNT-003 | GIR_101 + FilingInfo OECD_12 counted (extended) | 'GIR_101', FilingInfo.DocTypeIndic='OECD_12' | Correction Count + 1 — confirm OECD_12 validity | Img 4, p.11 (A9-1) |
| TC-CCNT-004 | GIR_101 + FilingInfo OECD_3 counted | 'GIR_101', FilingInfo.DocTypeIndic='OECD_3' | Correction Count + 1 | Img 4, p.11 |
| TC-CCNT-005 | GIR_101 + FilingInfo OECD_13 counted (extended) | 'GIR_101', FilingInfo.DocTypeIndic='OECD_13' | Correction Count + 1 — confirm OECD_13 validity | Img 4, p.11 (A9-1) |
| TC-CCNT-006 | GIR_101 + FilingInfo OECD_0/10 + a section OECD_1/11 counted | 'GIR_101', FilingInfo='OECD_0', GeneralSection.DocTypeIndic='OECD_1' | Correction Count + 1 | Img 4, p.11 |
| TC-CCNT-007 | GIR_101 + FilingInfo OECD_0 + no section OECD_1/11 not counted | 'GIR_101', FilingInfo='OECD_0', all four sections DocTypeIndic NOT in ('OECD_1','OECD_11') | NOT counted as correction | Img 4, p.11 |
| TC-CCNT-008 | GIR_101 + FilingInfo OECD_1 not counted as correction | 'GIR_101', FilingInfo='OECD_1' | NOT counted as correction (counts as GIR per Img 3) | Img 4, p.11; Img 3 |
| TC-CCNT-009 | Section check order / single increment | 'GIR_101', FilingInfo='OECD_0', Summary & UTPRAttribution both OECD_1 | Correction Count incremented once for this FilingInfo (order: GeneralSection→Summary→JurisdictionSection→UTPRAttribution; first match wins) | Img 4, p.11 |
| TC-CCNT-010 | AMEND_CNT persisted correctly | A file with M qualifying FilingInfo | GIR_MSG_SPEC.AMEND_CNT (via UPDATE_TX_CNT) = M | Img 4, p.11; p.12 |

## B21. End-to-End Outbound Workflow & Status Return (Img 1 / p.2) `[Boundary]` ★ 新增

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-E2E-001 | Upstream sequence up to extraction | 2025 run | CTBP5010 Update Document References → CTBP5024 report → CTBP5001 Data Extraction execute in order | Img 1, p.2 |
| TC-E2E-002 | Post-extraction report generated | After CTBP5001 | CTBP5025 report (To-be-Exchanged GIR Reports Pending Transmission) generated | Img 1, p.2 |
| TC-E2E-003 | Approval gate — Approve path | CTAP5002 review, decision=Approve | Proceeds to CTBP5002 Sign & Encrypt | Img 1, p.2 |
| TC-E2E-004 | Sign/encrypt + pre-transmit report | CTBP5002 signs & encrypts | CTBP5026 report generated before transmission | Img 1, p.2 |
| TC-E2E-005 | Outbound transmission & status update | Approved, signed packet | Data Packets transmitted to OECD via SFTP; CTBP5011 Update Transmission Status | Img 1, p.2 |
| TC-E2E-006 | Partner processing & status return | Packet delivered to partners | Pillar Two Partners download & process; return a Status Message via CTS | Img 1, p.2 |
| TC-E2E-007 | [Inbound-Status] Status return leg to End | Status message available on CTS | Download Status Message Packets from CTS via SFTP; CTBP5004 Receive Status Message → workflow reaches End | Img 1, p.2 |
| TC-E2E-008 | Approval gate — Reject path | CTAP5002 review, decision=Reject | Not signed/transmitted; flow loops back; records remain pending | Img 1, p.2 |

---

### 統計（v2）
- 說明章節：全 27 頁章節 + **4 張流程圖** 全數覆蓋（圖 1/2 已據圖補全內容；圖 3/4 計數邏輯已還原）。
- 測試用例：**共 21 組、約 127 條**；每條標註來源頁碼或圖號。
- 方向：核心 CTBP5001 為 **Outbound**；`[Boundary]` 群組含唯一的 `[Inbound-Status]` 回程（CTBP5004）。
- 待釐清：見 A9（**A9-1 代碼系列不一致** 為最高優先）。
