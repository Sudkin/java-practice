# CS501 / CTBP5001 — Test Case 對照依據說明

> 本文件逐筆說明 120 個 Test Case 的「對應資料表 / View / 規格書位置」欄位是怎麼推導出來的，依據來源分成四種類型：
>
> - 🟢 **SQL明示**：規格書原文直接寫「can be found in SQL幾」，或直接寫出某個 View 名稱
> - 🔵 **表名欄位**：規格書寫了具體表名/欄位（Appendix 1 View 定義、Appendix 2 異動總覽、§4.7 DB 欄位對照表等），但原文沒有講「SQL幾號」
> - ⚪ **段落敘述**：規格書只有文字敘述（Description／Application Flow／流程圖標籤），沒有點名任何資料表或 View——這種 TC 測的是「邏輯規則」而不是「查詢」
> - 🟠 **規格模糊**：規格書完全沒交代這個情境，此 TC 的目的就是拿來測出規格漏洞，不是驗證規格已寫好的東西

> 對照的原始依據為 `CS501.docx`（完整規格書）；部分測試資料查詢方式另參照 `11111.docx`（環境操作手冊摘錄，含 SQL Developer 查詢範例）。

---

## 目錄

1. [Record Selection — To-be-exchanged Criteria](#1-record-selection--to-be-exchanged-criteria)
2. [Jurisdiction Table Date-Window Check](#2-jurisdiction-table-date-window-check)
3. [Correction-Before-Extraction Merge](#3-correction-before-extraction-merge)
4. [Program Modes](#4-program-modes)
5. [File Generation Order & Grouping](#5-file-generation-order--grouping)
6. [OECD0 Resend Option](#6-oecd0-resend-option)
7. [30,000-Record Cap & File Splitting](#7-30000-record-cap--file-splitting)
8. [MessageRefId & Sequence Number](#8-messagerefid--sequence-number)
9. [Filing Entity Correction XML (GIR102) — Section B](#9-filing-entity-correction-xml-gir102--section-b)
10. [New Records XML (GIR101) — Section C](#10-new-records-xml-gir101--section-c)
11. [Correction & Deletion Records XML (GIR102) — Section D](#11-correction--deletion-records-xml-gir102--section-d)
12. [Database Update — Section E](#12-database-update--section-e)
13. [XML Validation & Character Handling — Section F + Description](#13-xml-validation--character-handling--section-f--description)
14. [File Placement / Naming — Section F](#14-file-placement--naming--section-f)
15. [DocRefId Construction (Cross-Cutting)](#15-docrefid-construction-cross-cutting)
16. [Negative / Edge Cases from Spec Ambiguities](#16-negative--edge-cases-from-spec-ambiguities)
17. [Application Flow — Core Decision Logic (Img 2 / p.3)](#17-application-flow--core-decision-logic-img-2--p3)
18. [GIR Count Logic — GIR_MSG_SPEC.GIR_CNT (Img 3 / p.10)](#18-gir-count-logic--gir_msg_specgir_cnt-img-3--p10)
19. [Correction Count Logic — GIR_MSG_SPEC.AMEND_CNT (Img 4 / p.11)](#19-correction-count-logic--gir_msg_specamend_cnt-img-4--p11)
20. [End-to-End Outbound Workflow & Status Return (Img 1 / p.2)](#20-end-to-end-outbound-workflow--status-return-img-1--p2)

---

## 1. Record Selection — To-be-exchanged Criteria

### TC-SEL-001 — Extract only "Accepted" and active GIR records

- **類型**：🔵 表名欄位
- **對照結果**：§4.1 待交換條件第1項(Accepted+Active)；來源表 PTP_ENC.RTN（STATUS/IS_ACTIVE，見11111.docx SQL example 1）
- **為什麼**：規格 §4.1 條件1是「From a "Accepted" and active GIR data files」。規格本身沒點名資料表，但這個狀態欄位實際查詢方式在 11111.docx 的 SQL example 1 裡已經示範（`Ptp_ENC.RTN` 的 `STATUS` 欄位），所以對照到 RTN 表。

### TC-SEL-002 — Exclude records not in "Accepted" status

- **類型**：🔵 表名欄位
- **對照結果**：§4.1 待交換條件第1項；來源表 PTP_ENC.RTN.STATUS
- **為什麼**：同上，反向情境（Pending/Rejected），一樣是 RTN.STATUS 欄位，只是驗證不成立的分支。

### TC-SEL-003 — Exclude records with error or warning during filing

- **類型**：🔵 表名欄位
- **對照結果**：§4.1 待交換條件第2項(無Error/Warning)；來源表 PTP_ENC.NTFC_RECORD、NTFC_DETAIL（見11111.docx SQL example 2）
- **為什麼**：規格 §4.1 條件2是「No error or warning exist during filing」。規格沒寫是查哪張表，但 11111.docx 的 SQL example 2 直接示範了怎麼查 NTFC_RECORD／NTFC_DETAIL 來確認一筆申報是否有通知，所以對照到這兩張表。

### TC-SEL-004 — Exclude records already sent

- **類型**：🔵 表名欄位
- **對照結果**：§4.1 待交換條件第3項(尚未傳送)；來源表 CTS_TX_REC.TX_STS（見Appendix 2 CTBP5011段落，TX_STS='T'表已傳送成功）
- **為什麼**：規格 §4.1 條件3是「Not yet sent」。這個狀態在 Appendix 2「CTBP5011 Update Transmission Status」段落有定義：TX_STS='T' 代表已傳送成功，所以對照到 CTS_TX_REC.TX_STS。

### TC-SEL-005 — Include record whose target jurisdiction is within effective period

- **類型**：🟢 SQL明示
- **對照結果**：§4.1 待交換條件第4項；SQL1 View V_CTRY_PARTNER，來源表 PTP_TAX_JRDT.OUT_DATE_STARTED/OUT_DATE_ENDED（Appendix 1 SQL1）
- **為什麼**：規格 §4.1 條件4是「target tax jurisdiction should be within the effective period defined in PTP_TAX_JRDT」——這裡直接點名了 PTP_TAX_JRDT 這張表。而這個判斷邏輯被包成 SQL1／View V_CTRY_PARTNER（Appendix 1），所以兩者都寫。

### TC-SEL-006 — Exclude record whose jurisdiction is outside effective period

- **類型**：🟢 SQL明示
- **對照結果**：§4.1 待交換條件第4項；SQL1 View V_CTRY_PARTNER（Appendix 1 SQL1，IS_PARTNER='N'分支）
- **為什麼**：同上，只是驗證窗口外（IS_PARTNER='N'）的分支，一樣是 SQL1 V_CTRY_PARTNER。

### TC-SEL-007 — Exclude records awaiting a status message from a prior transmission

- **類型**：🟢 SQL明示
- **對照結果**：§4.1 待交換條件第5項；SQL2 View V_CTRY_TRANSMISSION，來源表 CTS_GSEC/JSEC/SMRY/UTPR_DOC_REF.TRANSMISSION_FLAG（Appendix 1 SQL2）
- **為什麼**：規格 §4.1 條件5「transmitted...whose corresponding status message has not yet been received will not be extracted」對應到 SQL2，Appendix 1 SQL2 的說明文字寫得很白：「Records that are still in the transmission stage and for which the status message has not yet been received」，逐字對上。View 名稱是 V_CTRY_TRANSMISSION。

### TC-SEL-008 — Group extracted files by TIN + recipient jurisdiction + reporting year

- **類型**：⚪ 段落敘述
- **對照結果**：§4.1 待交換條件第6項(依FilingCE TIN+接收國+年度分組)；分組依據見§4.1末段「records will be grouped by...」
- **為什麼**：規格 §4.1 末段只有文字「grouped by...FilingCE TIN, recipient jurisdiction country, and reporting year」，沒有指定任何表/View，因為分組是程式邏輯步驟，不是資料庫查詢。

### TC-SEL-009 — Exchange keys sourced from SQL7

- **類型**：🟢 SQL明示
- **對照結果**：§4.3 Step A「All the records...can be found in SQL7」；SQL7 View V_EXT（Appendix 1 SQL7）
- **為什麼**：規格 §4.3 Step A 原文：「All the records...that are ready for exchange can be found in SQL7」——直接寫「SQL7」三個字，對照到 Appendix 1 SQL7 定義的 View V_EXT。

---

## 2. Jurisdiction Table Date-Window Check

### TC-JDX-001 — Extract when reporting FY start is inside jurisdiction window

- **類型**：🟢 SQL明示
- **對照結果**：§1 Description 視窗判斷式(Start Date≤FY Start≤Cessation Date)；SQL1 View V_CTRY_PARTNER，來源表 PTP_TAX_JRDT（Appendix 1 SQL1）
- **為什麼**：規格 §1 Description 那段視窗判斷式（Start Date ≤ FY Start ≤ Cessation Date）的實作就是 SQL1。Appendix 1 SQL1 的 WHERE 條件 `TRUNC(OUT_DATE_STARTED)<=TRUNC(SYSDATE) AND (TRUNC(OUT_DATE_ENDED)>=TRUNC(SYSDATE) OR OUT_DATE_ENDED IS NULL)` 正是這個窗口邏輯，所以對照到 V_CTRY_PARTNER。

### TC-JDX-002 — Do not extract when FY start before jurisdiction start

- **類型**：🟢 SQL明示
- **對照結果**：§1 Description 視窗判斷式；PTP_TAX_JRDT.OUT_DATE_STARTED（Appendix 1 SQL1）
- **為什麼**：同一條 SQL1 WHERE 條件，這裡測的是 OUT_DATE_STARTED 這半邊條件不成立的情況。

### TC-JDX-003 — Do not extract when FY start after cessation

- **類型**：🟢 SQL明示
- **對照結果**：§1 Description 視窗判斷式；PTP_TAX_JRDT.OUT_DATE_ENDED（Appendix 1 SQL1）
- **為什麼**：同一條 SQL1 WHERE 條件，測的是 OUT_DATE_ENDED 這半邊條件不成立的情況。

### TC-JDX-004 — Skip jurisdiction absent from Jurisdiction Table

- **類型**：🟢 SQL明示
- **對照結果**：§1 Description「check the Jurisdiction Table to determine whether the jurisdiction is included」；SQL1 V_CTRY_PARTNER 以LEFT JOIN NVL2判斷不存在時IS_PARTNER='N'（Appendix 1 SQL1）
- **為什麼**：SQL1 的寫法是 `PTP_ISO_COUNTRY C LEFT JOIN SRC S`，用 LEFT JOIN + NVL2 讓「表內查不到的國家」自動被判定 IS_PARTNER='N'，而不是報錯或漏掉。這個「安全跳過」的行為是從 SQL1 的 JOIN 寫法本身推導出來的，不是規格書文字直接講的。

### TC-JDX-005 — Only UPE/DFE-filed GIRs subject to extraction

- **類型**：🔵 表名欄位
- **對照結果**：§1 Description「Only GIRs filed by...UPE and...DFE are subject to data extraction」；來源表 PTP_ENC.RTN.FE_ROLE（見11111.docx SQL example 1，fe_role='UPE'）
- **為什麼**：規格 §1 Description 第一句「Only GIRs filed by...UPE and...DFE are subject to data extraction」，這是文字敘述沒有指名表。但要準備測試資料，實際判斷 FE_ROLE 的欄位在 11111.docx SQL example 1 裡示範過（`fe_role='UPE'`），所以對照回 RTN.FE_ROLE。

### TC-JDX-006 — Multi-jurisdiction element evaluated per jurisdiction

- **類型**：🟢 SQL明示
- **對照結果**：§1 Description「For each receiving jurisdiction of an element...check the Jurisdiction Table」逐國判斷；SQL1 V_CTRY_PARTNER 對每一 CTRY_CODE 各自判斷（Appendix 1 SQL1）
- **為什麼**：規格 §1「For each receiving jurisdiction of an element, the system should check the Jurisdiction Table」——這句話講的就是對每個國家各自查一次 SQL1，所以多國情境還是對照 V_CTRY_PARTNER，只是換成驗證「逐筆查」而非「查一次套全部」。

---

## 3. Correction-Before-Extraction Merge

### TC-COR-001 — Merge corrected element with remaining originals

- **類型**：⚪ 段落敘述
- **對照結果**：§1 Description 第2段「For example, a filing entity submits...」完整範例情境
- **為什麼**：這整組是規格 §1 Description 第二段的範例情境直接搬過來的（「a filing entity submits a GIR data file containing all five elements...」），規格書本身就是用文字案例說明，沒有點名任何 SQL 或表，因為這是「合併邏輯」的概念說明，不是查詢。

### TC-COR-002 — Accept partial correction (FilingInfo OECD0 + element)

- **類型**：⚪ 段落敘述
- **對照結果**：§1 Description「It is only required to submit the Filing Info (with OECD0) along with the corrected element(s)」
- **為什麼**：同段規格文字「It is only required to submit the Filing Info (with OECD0) along with the corrected element(s)」，一樣是文字規則，沒有表名。

### TC-COR-003 — Multiple elements corrected across files

- **類型**：🔵 表名欄位
- **對照結果**：§1 Description 合併邏輯推演；對應 SQL4 V_EXT_*_GIR102（Appendix 1 SQL4）多元素合併
- **為什麼**：這裡雖然規格本身沒直接寫表名，但「多元素同時更正」這個情境最終會落到 SQL4（V_EXT_*_GIR102）的多個 View 去撈各自最新版本，所以我把它跟 SQL4 關聯起來，方便你知道實際驗證時該查哪裡。

---

## 4. Program Modes

### TC-MODE-001 — Auto mode (no parameter)

- **類型**：⚪ 段落敘述
- **對照結果**：§3「In Auto mode, the program will scan the database and retrieve all to-be-exchanged records」
- **為什麼**：規格 §3 原文「In Auto mode, the program will scan the database and retrieve all to-be-exchanged records」——純文字敘述模式定義，沒有點名表。

### TC-MODE-002 — Ad-hoc mode with valid parameter

- **類型**：⚪ 段落敘述
- **對照結果**：§3「In Ad-hoc mode, the program will read the input parameters and compile into a list of specified FilingCE TINs, Jurisdiction and reporting year」
- **為什麼**：同上，Ad-hoc 模式那句「read the input parameters and compile into a list」，一樣純文字。

### TC-MODE-003 — Ad-hoc parameter format handling (invalid)

- **類型**：🟠 規格模糊
- **對照結果**：§3 Ad-hoc模式參數格式，規格未明確定義錯誤格式行為（規格模糊點，需與開發/業主確認）
- **為什麼**：規格書只定義了「合法參數長什麼樣子」（範例格式），完全沒寫「參數格式錯誤時程式該怎麼反應」。這個 TC 的目的就是拿去逼出程式實際行為，不是驗證規格寫的東西，所以標「規格模糊點」。

### TC-MODE-004 — Ad-hoc selects correct subset only

- **類型**：⚪ 段落敘述
- **對照結果**：§3 Ad-hoc模式參數範圍
- **為什麼**：跟 TC-MODE-002 同一句規格文字，只是換了參數值來驗證精準度。

---

## 5. File Generation Order & Grouping

### TC-ORD-001 — File order within one group

- **類型**：⚪ 段落敘述
- **對照結果**：§4.2「files will be generated in the following order: Filing Entity Correction Files(GIR102)→New Records Files(GIR101)→Correction Records Files(GIR102)」
- **為什麼**：規格 §4.2 原文列出三種檔案的固定順序（Filing Entity Correction→New Records→Correction Records），這是文字條列規則，沒有對應到查詢或表。

### TC-ORD-002 — Grouping precedes file generation

- **類型**：⚪ 段落敘述
- **對照結果**：§4.2「The records will be grouped by reporting year, FilingCE TIN, and recipient country code first, and then files will be generated...」
- **為什麼**：規格 §4.2「The records will be grouped by reporting year, FilingCE TIN, and recipient country code first, and then files will be generated」，同樣是流程文字。

---

## 6. OECD0 Resend Option

### TC-RES-001 — OECD0 resend permitted when FilingInfo already sent

- **類型**：⚪ 段落敘述
- **對照結果**：§4.2「The resend option (OECD0) can only be used for the Filing Info element when the Filing Info element has already been sent...before」
- **為什麼**：規格 §4.2「The resend option (OECD0) can only be used for the Filing Info element when the Filing Info element has already been sent...before」，這句話直接定義了 OECD0 的使用條件，是文字規則本身。

### TC-RES-002 — OECD0 restricted to FilingInfo element

- **類型**：⚪ 段落敘述
- **對照結果**：§4.2 同上，OECD0限定僅適用於FilingInfo元素，不適用於其他Section
- **為什麼**：同一句規格文字的反向驗證——OECD0 不能用在其他 Section 上。

---

## 7. 30,000-Record Cap & File Splitting

### TC-CAP-001 — Single file when ≤ 30,000

- **類型**：⚪ 段落敘述
- **對照結果**：§4.2「each file should contain no more than 30,000 records across the sections...」
- **為什麼**：規格 §4.2「each file should contain no more than 30,000 records」，數字門檻直接寫在文字裡。

### TC-CAP-002 — Split when > 30,000

- **類型**：⚪ 段落敘述
- **對照結果**：§4.2「In this case, multiple files should be generated」
- **為什麼**：同段「In this case, multiple files should be generated」。

### TC-CAP-003 — FilingInfo option across split files

- **類型**：⚪ 段落敘述
- **對照結果**：§4.2「The option (OECD1) should be used in the first file...the option (OECD0) should be used for each subsequent file. The DocRefId used in OECD0 should be the same as in OECD1」
- **為什麼**：同段「The option (OECD1) should be used in the first file...OECD0 should be used for each subsequent file...DocRefId used in OECD0 should be the same as in OECD1」，這幾句話把切檔時 OECD1/OECD0 的用法講得很清楚。

### TC-CAP-004 — Boundary — exactly 30,000

- **類型**：🟠 規格模糊
- **對照結果**：§4.2 30,000筆邊界值；規格未明講=30,000時是否切檔（規格模糊點）
- **為什麼**：規格只講「不超過30,000」，沒講「剛好等於30,000算不算超過」。這是邊界值沒定義清楚的情況，屬於規格模糊點。

---

## 8. MessageRefId & Sequence Number

### TC-MRI-001 — Unique MessageRefId per file

- **類型**：🟢 SQL明示
- **對照結果**：§4.3 Step A「Each file should have a unique MessageRefId」；SQL7 V_EXT（Appendix 1 SQL7）
- **為什麼**：規格 §4.3 Step A「Each file should have a unique MessageRefId」，這句話緊接在「SQL7」那句後面，同一個 Step A 段落，所以一併對照 SQL7 V_EXT（因為 MessageRefId 是針對 SQL7 撈出的每個組合各自產生）。

### TC-MRI-002 — Sequence starts at 0001

- **類型**：⚪ 段落敘述
- **對照結果**：§4.3 Step A「The sequence number <9999> starts from 0001」
- **為什麼**：規格 §4.3「The sequence number <9999> starts from 0001」，純文字規則，沒有表名。

### TC-MRI-003 — Sequence resets per key combination

- **類型**：⚪ 段落敘述
- **對照結果**：§4.3 Step A「...and is reset for each combination of 'Tax Jurisdiction','Fiscal Year',and 'FilingCE TIN'」
- **為什麼**：同句「...and is reset for each combination of 'Tax Jurisdiction','Fiscal Year',and 'FilingCE TIN'」。

### TC-MRI-004 — Sequence increments within same combination

- **類型**：⚪ 段落敘述
- **對照結果**：§4.3 Step A 同一組合內序號遞增規則
- **為什麼**：同一段落規則的延伸驗證（序號遞增而非重置）。

---

## 9. Filing Entity Correction XML (GIR102) — Section B

### TC-FEC-001 — FilingInfo from latest OECD2 record

- **類型**：🟢 SQL明示
- **對照結果**：§4.4 Step B「Records can be found in SQL6. Find the latest OECD2 record of FilingInfo」；SQL6 View V_FILING_INFO_OECD2（Appendix 1 SQL6）
- **為什麼**：規格 §4.4 Step B 原文「Records can be found in SQL6. Find the latest OECD2 record of FilingInfo」——直接寫「SQL6」，對照到 Appendix 1 SQL6 定義的 View V_FILING_INFO_OECD2。

### TC-FEC-002 — No section elements present

- **類型**：⚪ 段落敘述
- **對照結果**：§4.4 Step B「No elements are present for 'GeneralSection','Summary','JurisdictionSection' and 'UTPRAttribution'」
- **為什麼**：規格 §4.4「No elements are present for 'GeneralSection'...」這句是純規則敘述，講的是「不該有什麼」，沒有對應表。

### TC-FEC-003 — MessageSpec values for GIR102

- **類型**：⚪ 段落敘述
- **對照結果**：§4.4 Step B MessageSpec表格：TransmittingCountry/ReceivingCountry/MessageType/MessageTypeIndic/DocRefId/ReportingPeriod
- **為什麼**：規格 §4.4 的 MessageSpec 表格（TransmittingCountry/ReceivingCountry/...）是規格書自己畫的欄位定義表，不是資料庫表，是「XML 產出格式」的規則表。

### TC-FEC-004 — FilingInfo OECD2 fields

- **類型**：🟢 SQL明示
- **對照結果**：§4.4 Step B FilingInfo element表格：DocTypeIndic='OECD2'、DocRefId、CorrDocRefId取自V_LAST_FILING_INFO
- **為什麼**：規格 §4.4 FilingInfo 表格裡寫「CorrDocRefID is retrieved from V_LAST_FILING_INFO」，直接點名這個 View（對應 SQL5，雖然規格本文沒寫「SQL5」三個字，但這裡直接寫出 View 名稱，等同明示）。

### TC-FEC-005 — Source of FilingInfo XML content

- **類型**：🟢 SQL明示
- **對照結果**：§4.4 Step B「The XML contents of 'FilingInfo' should be retrieved from 'FI_ID' in V_FILING_INFO_OECD2」（Appendix 1 SQL6）
- **為什麼**：規格 §4.4「The XML contents of 'FilingInfo' should be retrieved from 'FI_ID' in V_FILING_INFO_OECD2」，直接點名 View 名稱（對應 SQL6）。

---

## 10. New Records XML (GIR101) — Section C

### TC-NEW-001 — MessageSpec for GIR101

- **類型**：⚪ 段落敘述
- **對照結果**：§4.5 Step C MessageSpec element表格（MessageTypeIndic='GIR101'）
- **為什麼**：規格 §4.5 的 MessageSpec 表格跟 TC-FEC-003 一樣，是 XML 產出格式定義表，不是資料庫查詢。

### TC-NEW-002 — FilingInfo=OECD1 when no prior FilingInfo

- **類型**：🟢 SQL明示
- **對照結果**：§4.5 Step C FilingInfo「If corresponding record...not found in V_LAST_FILING_INFO: DocTypeIndic='OECD1'」
- **為什麼**：規格 §4.5「If corresponding record...not found in V_LAST_FILING_INFO: DocTypeIndic should be 'OECD1'」，直接點名 View。

### TC-NEW-003 — FilingInfo=OECD0 when prior FilingInfo exists

- **類型**：🟢 SQL明示
- **對照結果**：§4.5 Step C FilingInfo「If...found in V_LAST_FILING_INFO: DocTypeIndic='OECD0', DocRefId=V_LAST_FILING_INFO.FI_DOC_REF_ID」
- **為什麼**：同段「If...found in V_LAST_FILING_INFO: DocTypeIndic='OECD0', DocRefId=V_LAST_FILING_INFO.FI_DOC_REF_ID」，一樣直接點名。

### TC-NEW-004 — GeneralSection element (GIR101)

- **類型**：🟢 SQL明示
- **對照結果**：§4.5 Step C GeneralSection element；來源表 PTP_GEN_SEC by 'V_EXT_GSEC_GIR101.GS_ID'（Appendix 1 SQL3）
- **為什麼**：規格 §4.5「The XML contents should be retrieved from PTP_GEN_SEC by 'V_EXT_GSEC_GIR101.GS_ID'」——這句話點名了兩個東西：來源表 PTP_GEN_SEC，以及用來篩選的 View V_EXT_GSEC_GIR101（屬於 SQL3 的其中一個子 View）。

### TC-NEW-005 — JurisdictionSection element (GIR101)

- **類型**：🟢 SQL明示
- **對照結果**：§4.5 Step C JurisdictionSection element；來源表 PTP_JDX_SEC by 'V_EXT_JSEC_GIR101.JS_ID'（Appendix 1 SQL3）
- **為什麼**：同上邏輯換成 JurisdictionSection：「retrieved from PTP_JDX_SEC by 'V_EXT_JSEC_GIR101.JS_ID'」。

### TC-NEW-006 — Summary element (GIR101)

- **類型**：🟢 SQL明示
- **對照結果**：§4.5 Step C Summary element；來源表 PTP_SMRY by 'V_EXT_SMRY_GIR101.SMRY_ID'（Appendix 1 SQL3）
- **為什麼**：同上邏輯換成 Summary：「retrieved from PTP_SMRY by 'V_EXT_SMRY_GIR101.SMRY_ID'」。

### TC-NEW-007 — UTPRAttribution element (GIR101)

- **類型**：🟢 SQL明示
- **對照結果**：§4.5 Step C UTPRAttribution element；來源表 PTP_UTPR_ATTR by 'V_EXT_UTPR_GIR101.UPAT_ID'（Appendix 1 SQL3）
- **為什麼**：同上邏輯換成 UTPRAttribution：「retrieved from PTP_UTPR_ATTR by 'V_EXT_UTPR_GIR101.UPAT_ID'」。

### TC-NEW-008 — New records sourced from V_EXT_GIR101 (SQL3)

- **類型**：🟢 SQL明示
- **對照結果**：§4.5 Step C「Records can be found in SQL3」；SQL3 View群 V_EXT_GSEC/JSEC/SMRY/UTPR_GIR101（Appendix 1 SQL3）
- **為什麼**：規格 §4.5 Step C 開頭「Records can be found in SQL3」，直接寫「SQL3」，對照到 Appendix 1 SQL3 底下四個 View（GSEC/JSEC/SMRY/UTPR_GIR101）。

---

## 11. Correction & Deletion Records XML (GIR102) — Section D

### TC-CRC-001 — MessageSpec for GIR102 correction file

- **類型**：⚪ 段落敘述
- **對照結果**：§4.6 Step D MessageSpec element表格（MessageTypeIndic='GIR102'）
- **為什麼**：跟 TC-FEC-003／TC-NEW-001 同類，是 §4.6 的 MessageSpec 格式定義表，非資料庫查詢。

### TC-CRC-002 — FilingInfo=OECD1 when no prior FilingInfo

- **類型**：🟢 SQL明示
- **對照結果**：§4.6 Step D FilingInfo「If...not found in V_LAST_FILING_INFO: DocTypeIndic='OECD1'」
- **為什麼**：規格 §4.6 FilingInfo 判斷邏輯跟 TC-NEW-002 用的是同一句規則（「not found in V_LAST_FILING_INFO」），只是套用在 GIR102 情境。

### TC-CRC-003 — FilingInfo=OECD0 when prior FilingInfo exists

- **類型**：🟢 SQL明示
- **對照結果**：§4.6 Step D FilingInfo「If...found in V_LAST_FILING_INFO: DocTypeIndic='OECD0'」
- **為什麼**：同上，「found in V_LAST_FILING_INFO」分支。

### TC-CRC-004 — GeneralSection correction carries CorrDocRefId

- **類型**：🟢 SQL明示
- **對照結果**：§4.6 Step D GeneralSection含CorrDocRefId=<V_EXT_GSEC_GIR102.LAST_CTS_DOC_REF_ID>；來源表 PTP_GEN_SEC by 'V_EXT_GSEC_GIR102.GS_ID'（Appendix 1 SQL4）
- **為什麼**：規格 §4.6「retrieved from PTP_GEN_SEC by 'V_EXT_GSEC_GIR102.GS_ID'」＋「CorrDocRefId: <V_EXT_GSEC_GIR102.LAST_CTS_DOC_REF_ID>」，這裡點名了 SQL4 底下的 View，並多了 CorrDocRefId 這個更正專用欄位（New Records沒有這欄，這是 GIR102 特有的）。

### TC-CRC-005 — JurisdictionSection correction

- **類型**：🟢 SQL明示
- **對照結果**：§4.6 Step D JurisdictionSection含CorrDocRefId；來源表 PTP_JDX_SEC by 'V_EXT_JSEC_GIR102.JS_ID'（Appendix 1 SQL4）
- **為什麼**：同上換成 JurisdictionSection，對照 V_EXT_JSEC_GIR102。

### TC-CRC-006 — Summary correction

- **類型**：🟢 SQL明示
- **對照結果**：§4.6 Step D Summary含CorrDocRefId；來源表 PTP_SMRY by 'V_EXT_SMRY_GIR102.SMRY_ID'（Appendix 1 SQL4）
- **為什麼**：同上換成 Summary，對照 V_EXT_SMRY_GIR102。

### TC-CRC-007 — UTPRAttribution correction

- **類型**：🟢 SQL明示
- **對照結果**：§4.6 Step D UTPRAttribution含CorrDocRefId；來源表 PTP_UTPR_ATTR by 'V_EXT_UTPR_GIR102.UPAT_ID'（Appendix 1 SQL4）
- **為什麼**：同上換成 UTPRAttribution，對照 V_EXT_UTPR_GIR102。

### TC-CRC-008 — Deletion record (OECD3) extraction

- **類型**：🟢 SQL明示
- **對照結果**：§4.6 Step D「DocTypeIndic should be 'OECD2' or 'OECD3'」；SQL4 V_EXT_GSEC_GIR102 DELETE RECORDS分支（Appendix 1 SQL4，OECD_3 UNION段）
- **為什麼**：規格 §4.6「DocTypeIndic should be 'OECD2' or 'OECD3'」——OECD3 是刪除類型。回頭看 Appendix 1 SQL4 的定義，V_EXT_GSEC_GIR102 內部用 UNION 把「OECD_2 RECORDS」「OECD_3 RECORDS」「DELETE RECORDS」三段查詢接起來，第三段的註解就寫著「-- DATA EXTRACTION (DELETE RECORDS)」，直接對上。

---

## 12. Database Update — Section E

### TC-DBU-001 — GIR_MSG_SPEC insert values

- **類型**：🔵 表名欄位
- **對照結果**：§4.7 Step E「1. GIR_MSG_SPEC」欄位表：APRVD_STS='P'、SGD_STS=NULL、GIR_CNT/AMEND_CNT
- **為什麼**：規格 §4.7 Step E「1. GIR_MSG_SPEC」那張表格直接列出 APRVD_STS='P'、SGD_STS=NULL 等固定值，這是規格書自己畫的 DB 欄位對照表，不是查詢語法，所以類型標「表名欄位」而非「SQL明示」。

### TC-DBU-002 — GIR_CNT updated by stored procedure

- **類型**：🔵 表名欄位
- **對照結果**：§4.7 Step E「Logic to update the GIR Count in GIR_MSG_SPEC.GIR_CNT by stored procedure UPDATE_TX_CNT」(流程圖Img3)
- **為什麼**：規格 §4.7「Logic to update the GIR Count in GIR_MSG_SPEC.GIR_CNT by stored procedure UPDATE_TX_CNT」，並附了一張流程圖（Img3）。這裡欄位名稱寫得很清楚，但邏輯本身是流程圖畫的，不是 SQL View。

### TC-DBU-003 — AMEND_CNT updated by stored procedure

- **類型**：🔵 表名欄位
- **對照結果**：§4.7 Step E「Logic to update the Amendment Count in GIR_MSG_SPEC.AMEND_CNT by stored procedure UPDATE_TX_CNT」(流程圖Img4)
- **為什麼**：同上，AMEND_CNT 版本，對應流程圖 Img4。

### TC-DBU-004 — FILING_INFO field mapping

- **類型**：🔵 表名欄位
- **對照結果**：§4.7 Step E「2. FILING_INFO」欄位對照表（FILING_INFO_ID/CORR_DOC_REF_ID/DOC_REF_ID/DOC_TYPE_IND/FCE_TIN/PTP_FI_ID/MS_ID等）
- **為什麼**：規格 §4.7「2. FILING_INFO」欄位對照表，把 FILING_INFO_ID/DOC_REF_ID/DOC_TYPE_IND 等每一欄的值來源都列出來了，直接照抄表格內容。

### TC-DBU-005 — GEN_SEC field mapping

- **類型**：🔵 表名欄位
- **對照結果**：§4.7 Step E「3. GEN_SEC」欄位對照表（PTP_GS_ID取自V_EXT_GSEC_GIR101/102.GS_ID等）
- **為什麼**：規格 §4.7「3. GEN_SEC」欄位對照表，PTP_GS_ID 的值來源寫著「V_EXT_GSEC_GIR101.GS_ID or V_EXT_GSEC_GIR102.GS_ID」——這裡又出現了 Step C／D 用過的同一批 View，因為它們的資料最終要寫進這張表。

### TC-DBU-006 — JDX_SEC field mapping

- **類型**：🔵 表名欄位
- **對照結果**：§4.7 Step E「4. JDX_SEC」欄位對照表（PTP_JS_ID取自V_EXT_JSEC_GIR101/102.JS_ID等）
- **為什麼**：同上换成「4. JDX_SEC」欄位對照表。

### TC-DBU-007 — SMRY field mapping

- **類型**：🔵 表名欄位
- **對照結果**：§4.7 Step E「5. SMRY」欄位對照表（PTP_SMRY_ID取自V_EXT_SMRY_GIR101/102.SMRY_ID等）
- **為什麼**：同上换成「5. SMRY」欄位對照表。

### TC-DBU-008 — UTPR_ATTR field mapping

- **類型**：🔵 表名欄位
- **對照結果**：§4.7 Step E「6. UTPR_ATTR」欄位對照表（PTP_UPAT_ID取自V_EXT_UTPR_GIR101/102.UPAT_ID等）
- **為什麼**：同上换成「6. UTPR_ATTR」欄位對照表。

### TC-DBU-009 — Records inserted into GIR_* tables

- **類型**：⚪ 段落敘述
- **對照結果**：§1 Description「the records should be created and inserted into corresponding database tables」；§4.7 Step E「Insert the XML contents into database tables whose names start with 'GIR_*'」
- **為什麼**：規格 §1 Description 末段「the records should be created and inserted into corresponding database tables」+ §4.7「Insert the XML contents into database tables whose names start with 'GIR_*'」，這是整體規則敘述，不特定指哪張表。

### TC-DBU-010 — Common audit columns populated

- **類型**：🔵 表名欄位
- **對照結果**：§4.7 Step E 六張明細表共通欄位 LAST_UPD_AT/LAST_UPD_BY/CREATED_AT/CREATED_BY（各表格末列）
- **為什麼**：§4.7 六張明細表格的最後幾列都重複出現 LAST_UPD_AT/LAST_UPD_BY/CREATED_AT/CREATED_BY 這組共通稽核欄位，直接照抄。

---

## 13. XML Validation & Character Handling — Section F + Description

### TC-VAL-001 — Ampersand escaped

- **類型**：🔵 表名欄位
- **對照結果**：§4.8 Step F XML Entity Reference表：& → &amp;
- **為什麼**：規格 §4.8 Step F 有一張「XML Entity Reference」對照表，& 對應 &amp;，直接照抄表格內容（這張表不是資料庫表，是字元轉換規則表）。

### TC-VAL-002 — Less-than escaped

- **類型**：🔵 表名欄位
- **對照結果**：§4.8 Step F XML Entity Reference表：< → &lt;
- **為什麼**：同一張表，< 對應 &lt;。

### TC-VAL-003 — Greater-than escaped

- **類型**：🔵 表名欄位
- **對照結果**：§4.8 Step F XML Entity Reference表：> → &gt;
- **為什麼**：同一張表，> 對應 &gt;。

### TC-VAL-004 — Apostrophe escaped

- **類型**：🔵 表名欄位
- **對照結果**：§4.8 Step F XML Entity Reference表：' → &apos;
- **為什麼**：同一張表，' 對應 &apos;。

### TC-VAL-005 — Quote escaped

- **類型**：🔵 表名欄位
- **對照結果**：§4.8 Step F XML Entity Reference表：" → &quot;
- **為什麼**：同一張表，" 對應 &quot;。

### TC-VAL-006 — Reject unacceptable char — double dash

- **類型**：🔵 表名欄位
- **對照結果**：§1 Description 不可接受字元表：-- (Double dash)
- **為什麼**：規格 §1 Description 另外有一張「不可接受字元」表（與 Step F 的 Entity 表是不同表），-- 是其中一項。

### TC-VAL-007 — Reject unacceptable char — slash asterisk

- **類型**：🔵 表名欄位
- **對照結果**：§1 Description 不可接受字元表：/* (Slash asterisk)
- **為什麼**：同上不可接受字元表，/* 是其中一項。

### TC-VAL-008 — Reject unacceptable char — ampersand hash

- **類型**：🔵 表名欄位
- **對照結果**：§1 Description 不可接受字元表：&# (Ampersand hash)
- **為什麼**：同上不可接受字元表，&# 是其中一項。

### TC-VAL-009 — Output conforms to OECD GIR XML Schema

- **類型**：⚪ 段落敘述
- **對照結果**：§1 Description「These XML files must conform to the GIR XML Schema issued by OECD」
- **為什麼**：規格 §1「These XML files must conform to the GIR XML Schema issued by OECD」，這是外部規範引用（OECD Schema），規格書本身沒有把 Schema 內容列出來，只是要求要符合，所以是段落敘述而非表格。

### TC-VAL-010 — All three XML file types produced

- **類型**：⚪ 段落敘述
- **對照結果**：§1 Description「The following types of XML files should be generated: (i)(ii)(iii)」三種檔案類型
- **為什麼**：規格 §1「The following types of XML files should be generated: (i)(ii)(iii)」，三種檔案類型的條列文字。

### TC-VAL-011 — Escaping vs unacceptable-char interaction

- **類型**：🟠 規格模糊
- **對照結果**：§1 Description 不可接受字元表 + §4.8 Entity Reference表 交互情境（規格未明講&#是否會被XML轉換規則先轉成&amp;#，屬模糊點）
- **為什麼**：規格書有兩張字元規則表（Description 的「不可接受字元」表、Step F 的「Entity Reference」表），但沒有講清楚兩者的判斷順序——例如原始內容有 & 之後被轉成 &amp;，會不會反而在 &amp; 裡面造出一個 &# 的樣式而誤觸不可接受字元檢查？這是規格沒交代的交互情境，所以標規格模糊。

---

## 14. File Placement / Naming — Section F

### TC-FILE-001 — Payload file placement

- **類型**：⚪ 段落敘述
- **對照結果**：§4.8 Step F「Payload File \\data\\out\\pending\\<MessageRefId>.xml」
- **為什麼**：規格 §4.8「Payload File \\data\\out\\pending\\<MessageRefId>.xml」，路徑命名規則直接寫在文字裡，不是資料庫表。

### TC-FILE-002 — Metadata file placement

- **類型**：⚪ 段落敘述
- **對照結果**：§4.8 Step F「Metadata File \\data\\out\\pending\\<MessageRefId>.metadata.xml」
- **為什麼**：同上，Metadata File 的命名規則。

---

## 15. DocRefId Construction (Cross-Cutting)

### TC-DRI-001 — MessageSpec DocRefId

- **類型**：⚪ 段落敘述
- **對照結果**：§4.4/4.5/4.6 MessageSpec DocRefId公式：HK+RPT_YR+REC_JDX_CTRY+FCE_TIN+'-'+<yyyyMMdd>+'-'+<9999>
- **為什麼**：規格 §4.4/4.5/4.6 每個 Step 的 MessageSpec 表格都重複列出同一條 DocRefId 公式（HK+RPT_YR+...+<9999>），這是命名規則公式，不是資料庫查詢。

### TC-DRI-002 — FilingInfo DocRefId with '-F-'

- **類型**：⚪ 段落敘述
- **對照結果**：§4.4/4.5/4.6 FilingInfo DocRefId公式：HK+RPT_YR+REC_JDX_CTRY+FCE_TIN+'-F-'+<yyyyMMdd>
- **為什麼**：同樣道理，FilingInfo 的 DocRefId 公式（含 '-F-'）在三個 Step 都重複出現。

### TC-DRI-003 — GeneralSection DocRefId with '-G-'

- **類型**：🟢 SQL明示
- **對照結果**：§4.5/4.6 GeneralSection DocRefId公式：HK+RPT_YR+REC_JDX_CTRY+FCE_TIN+'-G-'+<yyyyMMdd>+'-'+<V_EXT_GSEC_GIR101/102.DOC_REF_ID>
- **為什麼**：GeneralSection 的 DocRefId 公式結尾用了 `<V_EXT_GSEC_GIR101.DOC_REF_ID>` 或 `<V_EXT_GSEC_GIR102.DOC_REF_ID>`，公式本身雖是命名規則，但取值來源直接點名了 View，所以標 SQL明示。

### TC-DRI-004 — Section suffix letters correct

- **類型**：⚪ 段落敘述
- **對照結果**：§4.5/4.6 四段落字尾：G(GeneralSection)/J(JurisdictionSection)/S(Summary)/U(UTPRAttribution)
- **為什麼**：四個 Section 各自的字尾代號（G/J/S/U）分散在 §4.5/4.6 四段表格裡，是命名規則的彙整比較，不是單一表格內容。

### TC-DRI-005 — ReportingPeriod construction

- **類型**：⚪ 段落敘述
- **對照結果**：§4.4/4.5/4.6 MessageSpec「ReportingPeriod: RPT_YR +'-12-31'」
- **為什麼**：ReportingPeriod 公式（RPT_YR+'-12-31'）在三個 MessageSpec 表格裡重複出現，同樣是命名規則。

---

## 16. Negative / Edge Cases from Spec Ambiguities

### TC-EDG-001 — OECD value spelling consistency (output vs comparison)

- **類型**：🟠 規格模糊
- **對照結果**：規格書中OECD值拼寫不一致：本文多處寫『OECD1/OECD2/OECD0』(§4.4-4.6)，Appendix 1 SQL View 定義卻用『OECD_1/OECD_2/OECD_0』(帶底線)；此為規格模糊點，需確認實際程式輸出格式
- **為什麼**：規格書內文（§4.4-4.6 表格）寫的是 OECD1/OECD2/OECD0（不帶底線），但 Appendix 1 SQL View 定義裡的 DOC_TYPE_IND 值卻是 'OECD_1'/'OECD_2'/'OECD_0'（帶底線）。兩處拼法不一致，規格書自己打架，所以這是規格模糊點，需要跟實際程式或資料庫核對哪個才是真的。

### TC-EDG-002 — View name typo does not break resolution

- **類型**：🟠 規格模糊
- **對照結果**：Appendix 1 SQL5 View 定義名稱為 V_LAST_FILING_INO（缺一個F），但內文§4.4-4.6多處引用寫作 V_LAST_FILING_INFO；此為規格書筆誤/命名不一致，需與實際DB核對
- **為什麼**：Appendix 1 SQL5 的 CREATE VIEW 語句寫的實際名稱是 V_LAST_FILING_INO（少一個F），但內文 §4.4/4.5/4.6 引用時全部寫作 V_LAST_FILING_INFO。這是規格書的命名筆誤，兩個名字實際上应该是同一個東西，但字面不一致，所以標規格模糊，測試時要驗證程式最終連的是哪個名字。

### TC-EDG-003 — No to-be-exchanged records found

- **類型**：🟢 SQL明示
- **對照結果**：§3「In Auto mode...retrieve all to-be-exchanged records」；SQL7 V_EXT查無資料之情境（Appendix 1 SQL7）
- **為什麼**：驗證「SQL7 查無資料」的情境，因為 SQL7（V_EXT）是待交換清單的權威來源，如果它是空的，Auto 模式該回傳空清單而不是報錯。

### TC-EDG-004 — Ad-hoc jurisdiction not an active partner

- **類型**：🟢 SQL明示
- **對照結果**：§3 Ad-hoc模式 + SQL1 V_CTRY_PARTNER.IS_PARTNER='N'（Appendix 1 SQL1）
- **為什麼**：跟 SQL1 V_CTRY_PARTNER 的 IS_PARTNER='N' 分支合併驗證，這裡的情境是「Ad-hoc 模式手動指定了一個不在夥伴清單內的國家」，考驗程式是否會因為是手動指定就繞過 SQL1 的檢查。

---

## 17. Application Flow — Core Decision Logic (Img 2 / p.3)

### TC-AF-001 — "Check input parameter?" → No branch

- **類型**：⚪ 段落敘述
- **對照結果**：§3「without input parameter (Auto mode)」流程圖(Application Flow, Img2)「Check input parameter?」No分支
- **為什麼**：對應規格書那張流程圖（Application Flow, Img2）裡「Check input parameter?」判斷框的 No 分支，圖片內容沒有轉成文字表格，只能對照到圖片本身的位置（p.3）。

### TC-AF-002 — "Check input parameter?" → Yes branch

- **類型**：⚪ 段落敘述
- **對照結果**：§3「with input parameter (Ad-hoc mode)」流程圖 Yes分支
- **為什麼**：同一個判斷框的 Yes 分支。

### TC-AF-003 — Grouping step

- **類型**：⚪ 段落敘述
- **對照結果**：§4.1末段「records will be grouped by...first」對應流程圖「分組」步驟
- **為什麼**：流程圖裡「分組」步驟其實文字版就在 §4.1 末段（跟 TC-SEL-008／TC-ORD-002 引用同一句話），這裡是驗證流程圖畫的順序跟文字規則一致。

### TC-AF-004 — MessageSpec generation granularity

- **類型**：🟠 規格模糊
- **對照結果**：流程圖 Img2 MessageSpec產生粒度標註（規格圖文字，需實測確認是否再細分至TIN，屬模糊點）
- **為什麼**：流程圖上寫「每個 Tax Jurisdiction 與 Fiscal Year 組合各產生一個 MessageSpec」，但沒有講清楚要不要再拆到 TIN 這一層，圖片文字本身留了空白，需要實測才能確認粒度。

### TC-AF-005 — Loop branch 1 — FilingInfo change → Filing Entity Correction

- **類型**：⚪ 段落敘述
- **對照結果**：流程圖 Img2 判斷分支1：FilingInfo異動→§4.4 Step B Filing Entity Correction
- **為什麼**：流程圖判斷分支1（FilingInfo異動）對應到文字版的 §4.4 Step B，兩者是同一件事的圖片版與文字版。

### TC-AF-006 — Loop branch 2 — new records → New Records

- **類型**：⚪ 段落敘述
- **對照結果**：流程圖 Img2 判斷分支2：新記錄→§4.5 Step C New Records
- **為什麼**：流程圖判斷分支2對應文字版 §4.5 Step C。

### TC-AF-007 — Loop branch 3 — correction records → Correction Records

- **類型**：⚪ 段落敘述
- **對照結果**：流程圖 Img2 判斷分支3：更正記錄→§4.6 Step D Correction Records
- **為什麼**：流程圖判斷分支3對應文字版 §4.6 Step D。

### TC-AF-008 — Branch precedence (FilingInfo checked first)

- **類型**：🟠 規格模糊
- **對照結果**：流程圖 Img2 三分支判斷順序（規格圖隱含FilingInfo優先，屬模糊點，需實測確認）
- **為什麼**：流程圖三個判斷框畫的順序「隱含」FilingInfo優先判斷，但沒有文字明確寫「優先順序」，是從圖的排列方式推測，所以標規格模糊，需要實測驗證。

### TC-AF-009 — Loop termination

- **類型**：⚪ 段落敘述
- **對照結果**：流程圖 Img2「End of records?」迴圈結束條件
- **為什麼**：流程圖裡「End of records?」是迴圈結束條件的判斷框，純粹是流程圖元素。

### TC-AF-010 — No branch matched

- **類型**：🟠 規格模糊
- **對照結果**：流程圖 Img2 三分支皆不符合時之跳過邏輯（規格圖隱含，非明文條列）
- **為什麼**：流程圖三個分支都不符合時該怎麼辦，圖上沒有畫第四條路徑，只能推測是跳過繼續下一筆，所以標規格模糊。

---

## 18. GIR Count Logic — GIR_MSG_SPEC.GIR_CNT (Img 3 / p.10)

### TC-GCNT-001 — GIR_101 + FilingInfo OECD_1 counted

- **類型**：🔵 表名欄位
- **對照結果**：§4.7 Step E GIR_CNT邏輯圖(Img3/p.10)：MessageTypeIndic='GIR_101' + FilingInfo.DocTypeIndic='OECD_1'
- **為什麼**：規格 §4.7 GIR_CNT 那張邏輯圖（Img3）畫的判斷式，圖片標題「MessageTypeIndic='GIR_101' + FilingInfo.DocTypeIndic='OECD_1'」直接對應這個 TC 的情境。

### TC-GCNT-002 — GIR_101 + FilingInfo OECD_11 counted (extended code)

- **類型**：🟠 規格模糊
- **對照結果**：§4.7 Step E GIR_CNT邏輯圖(Img3/p.10)：DocTypeIndic='OECD_11'擴充碼
- **為什麼**：OECD_11 是「擴充碼」，附錄裡沒有另外列出來，是否等同 OECD_1 計入，圖片本身沒有畫出這個分支，需要實測確認，所以歸類為規格模糊（雖然對照欄仍指向 Img3 的位置，但驗證目的本身帶有模糊性）。

### TC-GCNT-003 — GIR_101 + FilingInfo OECD_0 not counted

- **類型**：🔵 表名欄位
- **對照結果**：§4.7 Step E GIR_CNT邏輯圖(Img3/p.10)：DocTypeIndic='OECD_0'不計入
- **為什麼**：同一張 Img3 邏輯圖裡 OECD_0（續傳）不計入的分支。

### TC-GCNT-004 — GIR_102 not counted as GIR

- **類型**：🔵 表名欄位
- **對照結果**：§4.7 Step E GIR_CNT邏輯圖(Img3/p.10)：MessageTypeIndic='GIR_102'不計入
- **為什麼**：同一張圖裡 GIR_102 不算 GIR 的分支。

### TC-GCNT-005 — Multiple FilingInfo in one MessageSpec

- **類型**：🔵 表名欄位
- **對照結果**：§4.7 Step E GIR_CNT邏輯圖(Img3/p.10) 多筆FilingInfo混合類型加總
- **為什麼**：同一張圖的邏輯套用在多筆 FilingInfo 混合的情境，驗證加總正確性。

### TC-GCNT-006 — GIR_CNT persisted correctly

- **類型**：🔵 表名欄位
- **對照結果**：§4.7 Step E「GIR_MSG_SPEC.GIR_CNT」持久化欄位（GIR_MSG_SPEC表）
- **為什麼**：規格 §4.7「GIR_MSG_SPEC.GIR_CNT」是這個計算結果最終寫入的欄位，直接對照到 GIR_MSG_SPEC 表。

---

## 19. Correction Count Logic — GIR_MSG_SPEC.AMEND_CNT (Img 4 / p.11)

### TC-CCNT-001 — GIR_102 counted as correction

- **類型**：🔵 表名欄位
- **對照結果**：§4.7 Step E AMEND_CNT邏輯圖(Img4/p.11)：MessageTypeIndic='GIR_102'計為更正
- **為什麼**：規格 §4.7 AMEND_CNT 邏輯圖（Img4）裡 GIR_102 一律算更正的分支。

### TC-CCNT-002 — GIR_101 + FilingInfo OECD_2 counted

- **類型**：🔵 表名欄位
- **對照結果**：§4.7 Step E AMEND_CNT邏輯圖(Img4/p.11)：'GIR_101'+FilingInfo.DocTypeIndic='OECD_2'
- **為什麼**：同一張圖裡 FilingInfo.DocTypeIndic='OECD_2' 的分支。

### TC-CCNT-003 — GIR_101 + FilingInfo OECD_12 counted (extended)

- **類型**：🟠 規格模糊
- **對照結果**：§4.7 Step E AMEND_CNT邏輯圖(Img4/p.11)：DocTypeIndic='OECD_12'擴充碼
- **為什麼**：OECD_12 是擴充碼，跟 TC-GCNT-002 同樣道理，圖上沒畫出這個分支，需要實測確認。

### TC-CCNT-004 — GIR_101 + FilingInfo OECD_3 counted

- **類型**：🔵 表名欄位
- **對照結果**：§4.7 Step E AMEND_CNT邏輯圖(Img4/p.11)：DocTypeIndic='OECD_3'
- **為什麼**：同一張圖裡 OECD_3（刪除類型）的分支。

### TC-CCNT-005 — GIR_101 + FilingInfo OECD_13 counted (extended)

- **類型**：🟠 規格模糊
- **對照結果**：§4.7 Step E AMEND_CNT邏輯圖(Img4/p.11)：DocTypeIndic='OECD_13'擴充碼
- **為什麼**：OECD_13 同樣是擴充碼，需要實測確認。

### TC-CCNT-006 — GIR_101 + FilingInfo OECD_0/10 + a section OECD_1/11 counted

- **類型**：🔵 表名欄位
- **對照結果**：§4.7 Step E AMEND_CNT邏輯圖(Img4/p.11)：FilingInfo='OECD_0'但某Section='OECD_1/11'仍計入
- **為什麼**：同一張圖裡最複雜的分支：FilingInfo=OECD_0 但底下某個 Section 是 OECD_1/11 時仍要算一次更正。

### TC-CCNT-007 — GIR_101 + FilingInfo OECD_0 + no section OECD_1/11 not counted

- **類型**：🔵 表名欄位
- **對照結果**：§4.7 Step E AMEND_CNT邏輯圖(Img4/p.11)：FilingInfo='OECD_0'且四Section皆非'OECD_1/11'則不計入
- **為什麼**：同一張圖的反例分支：四個 Section 都不是 OECD_1/11 時不計入。

### TC-CCNT-008 — GIR_101 + FilingInfo OECD_1 not counted as correction

- **類型**：🔵 表名欄位
- **對照結果**：§4.7 Step E AMEND_CNT邏輯圖(Img4/p.11)：FilingInfo='OECD_1'不計為更正(與GIR_CNT互斥)
- **為什麼**：驗證 GIR_CNT 與 AMEND_CNT 兩個計數欄位互斥——OECD_1 只進 GIR_CNT（見 TC-GCNT-001），不會同時進 AMEND_CNT，這是把 Img3 跟 Img4 兩張圖放在一起比對得出的推論。

### TC-CCNT-009 — Section check order / single increment

- **類型**：🔵 表名欄位
- **對照結果**：§4.7 Step E AMEND_CNT邏輯圖(Img4/p.11)：四Section判斷順序(GeneralSection→Summary→JurisdictionSection→UTPRAttribution)命中即跳出、單次遞增
- **為什麼**：Img4 圖上對四個 Section 的檢查順序（GeneralSection→Summary→JurisdictionSection→UTPRAttribution）是圖片本身畫的判斷路徑，用來驗證「命中即跳出、不重複累加」。

### TC-CCNT-010 — AMEND_CNT persisted correctly

- **類型**：🔵 表名欄位
- **對照結果**：§4.7 Step E「GIR_MSG_SPEC.AMEND_CNT」持久化欄位（GIR_MSG_SPEC表）
- **為什麼**：規格 §4.7「GIR_MSG_SPEC.AMEND_CNT」是計算結果最終寫入的欄位，直接對照到 GIR_MSG_SPEC 表。

---

## 20. End-to-End Outbound Workflow & Status Return (Img 1 / p.2)

### TC-E2E-001 — Upstream sequence up to extraction

- **類型**：🔵 表名欄位
- **對照結果**：§2 Overall Workflow(Img1/p.2)：CTBP5010→CTBP5024→CTBP5001 上游順序；CTBP5010異動見Appendix 2 第1項
- **為什麼**：規格 §2 整體流程圖（Img1）畫出 CTBP5010→CTBP5024→CTBP5001 的順序；CTBP5010 這個程式對資料庫的實際異動寫在 Appendix 2 第1項（CTS_GSEC_DOC_REF 等表 TRANSMISSION_FLAG=NULL）。

### TC-E2E-002 — Post-extraction report generated

- **類型**：⚪ 段落敘述
- **對照結果**：§2 Overall Workflow(Img1/p.2)：CTBP5001→CTBP5025 報表產出
- **為什麼**：流程圖上 CTBP5001→CTBP5025 報表產出這段，Appendix 2 沒有專門列 CTBP5025 的資料庫異動（它是報表程式，不寫入 GIR_* 表），所以只能對照流程圖位置。

### TC-E2E-003 — Approval gate — Approve path

- **類型**：🔵 表名欄位
- **對照結果**：§2 Overall Workflow(Img1/p.2)：CTAP5002核准Approve路徑；資料庫異動見Appendix 2 第3項(APRVD_STS='A', SGD_STS='P')
- **為什麼**：CTAP5002 核准 Approve 路徑的資料庫異動明確寫在 Appendix 2 第3項：APRVD_STS='A'、SGD_STS='P'。

### TC-E2E-004 — Sign/encrypt + pre-transmit report

- **類型**：🔵 表名欄位
- **對照結果**：§2 Overall Workflow(Img1/p.2)：CTBP5002簽章加密→CTBP5026報表；資料庫異動見Appendix 2 第4項(SGD_STS='S', CTS_TX_REC.TX_STS='R')
- **為什麼**：CTBP5002 簽章加密的異動寫在 Appendix 2 第4項：GIR_MSG_SPEC.SGD_STS='S'，並新增 CTS_TX_REC（TX_STS='R'）。

### TC-E2E-005 — Outbound transmission & status update

- **類型**：🔵 表名欄位
- **對照結果**：§2 Overall Workflow(Img1/p.2)：SFTP傳輸→CTBP5011更新狀態；資料庫異動見Appendix 2 第5項(CTS_TX_REC.TX_STS/CTS_RTN_CODE/TX_DATE等)
- **為什麼**：CTBP5011 更新傳輸狀態的異動寫在 Appendix 2 第5項：CTS_TX_REC 的 TX_STS/CTS_RTN_CODE/TX_DATE 等欄位。

### TC-E2E-006 — Partner processing & status return

- **類型**：🔵 表名欄位
- **對照結果**：§2 Overall Workflow(Img1/p.2)：夥伴處理→CTBP5004接收狀態訊息；資料庫異動見Appendix 2 第6項(CTS_*_DOC_REF.TRANSMISSION_FLAG=NULL等)
- **為什麼**：CTBP5004 接收狀態訊息的異動寫在 Appendix 2 第6項：CTS_GSEC/JSEC/SMRY/UTPR_DOC_REF 的 TRANSMISSION_FLAG 改回 NULL，代表整個外傳流程走完一圈。

### TC-E2E-007 — Approval gate — Reject path

- **類型**：🔵 表名欄位
- **對照結果**：§2 Overall Workflow(Img1/p.2)：CTAP5002 Reject路徑；資料庫異動見Appendix 2 第3項(APRVD_STS='R', SGD_STS=NULL)
- **為什麼**：CTAP5002 Reject 路徑的資料庫異動同樣在 Appendix 2 第3項，只是走另一個分支：APRVD_STS='R'、SGD_STS=NULL。

---
