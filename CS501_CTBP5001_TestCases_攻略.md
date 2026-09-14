# CS501｜CTBP5001 資料萃取 — 測試案例執行攻略

> 本文件對應 `CS501_CTBP5001_TestCases_Merged.xlsx` 裡的 120 筆測試案例，逐筆說明**目的**（為什麼要測、驗證規格書哪個規則）與**執行流程**（怎麼準備資料、跑什麼、檢查什麼），依規格書章節分成 20 組。執行時建議搭配該 xlsx 的 Test Data／Expected Result 欄位對照，本文件不重複列出完整測試資料，只聚焦「為何測」與「怎麼測」。

## 目錄

| # | 分類 | 案例數 | 對應規格 |
|---|---|---|---|
| 1 | [Record Selection — 待交換記錄篩選條件](#1-record-selection--待交換記錄篩選條件) | 9 | §4.1 |
| 2 | [Jurisdiction Table Date-Window Check](#2-jurisdiction-table-date-window-check) | 6 | §1、SQL1 |
| 3 | [Correction-Before-Extraction Merge](#3-correction-before-extraction-merge) | 3 | §1 範例情境 |
| 4 | [Program Modes](#4-program-modes) | 4 | §3、§4.1 |
| 5 | [File Generation Order & Grouping](#5-file-generation-order--grouping) | 2 | §4.2 |
| 6 | [OECD0 Resend Option](#6-oecd0-resend-option) | 2 | §4.2 |
| 7 | [30,000-Record Cap & File Splitting](#7-30000-record-cap--file-splitting) | 4 | §4.2 |
| 8 | [MessageRefId & Sequence Number](#8-messagerefid--sequence-number) | 4 | §4.3、SQL7 |
| 9 | [Filing Entity Correction XML (GIR102) — Step B](#9-filing-entity-correction-xml-gir102--step-b) | 5 | §4.4 |
| 10 | [New Records XML (GIR101) — Step C](#10-new-records-xml-gir101--step-c) | 8 | §4.5 |
| 11 | [Correction & Deletion Records XML (GIR102) — Step D](#11-correction--deletion-records-xml-gir102--step-d) | 8 | §4.6 |
| 12 | [Database Update — Step E](#12-database-update--step-e) | 10 | §4.7 |
| 13 | [XML Validation & Character Handling — Step F](#13-xml-validation--character-handling--step-f) | 11 | §4.8、§1 |
| 14 | [File Placement / Naming — Step F](#14-file-placement--naming--step-f) | 2 | §4.8 |
| 15 | [DocRefId Construction（跨步驟共用公式）](#15-docrefid-construction跨步驟共用公式) | 5 | §4.4–4.6 |
| 16 | [Negative / Edge Cases from Spec Ambiguities](#16-negative--edge-cases-from-spec-ambiguities) | 4 | 全文 |
| 17 | [Application Flow — Core Decision Logic](#17-application-flow--core-decision-logic) | 10 | §3 |
| 18 | [GIR Count Logic（GIR_CNT）](#18-gir-count-logicgir_cnt) | 6 | §4.7 Img3 |
| 19 | [Correction Count Logic（AMEND_CNT）](#19-correction-count-logicamend_cnt) | 10 | §4.7 Img4 |
| 20 | [End-to-End Outbound Workflow & Status Return](#20-end-to-end-outbound-workflow--status-return) | 7 | §2 |

**合計 120 筆。**

---

## 1. Record Selection — 待交換記錄篩選條件

這組驗證「什麼樣的記錄才算待交換」六項條件全部有被正確套用，是整個萃取程式的第一道關卡——漏放或錯放都會導致資料外洩，或該傳的資料被漏傳。

### TC-SEL-001 — 只萃取 Accepted 且 Active 的記錄
- **目的**：確保只有已受理（Accepted）且有效（Active）狀態的 GIR 記錄會被視為待交換，防止把尚未受理或已作廢的資料誤傳出境。
- **流程**：
  1. 準備一筆 `status='Accepted'`、`IS_ACTIVE='Y'`、`RPT_YR=2025` 的 GIR 記錄（可用 `PTP_ENC.RTN` 查詢核對來源狀態）。
  2. 執行 CTBP5001（Auto／Ad-hoc 皆可）。
  3. 查詢 SQL7（V_EXT）或檢視輸出 XML，確認該筆記錄有出現在待交換清單中。
  4. 檢核：記錄成功納入，無遺漏。

### TC-SEL-002 — 排除非 Accepted 狀態的記錄
- **目的**：驗證 Pending 或 Rejected 狀態的記錄不會被誤判為可交換，避免傳送未經核准的資料。
- **流程**：
  1. 準備一筆 `status=Pending/Rejected`、`RPT_YR=2025` 的記錄。
  2. 執行程式。
  3. 確認 SQL7 結果與輸出 XML 皆不含此筆。
  4. 檢核：正確排除。

### TC-SEL-003 — 排除存在 Error／Warning 的記錄
- **目的**：驗證申報過程中曾出現通知（NTFC）的記錄不會流入萃取結果，避免有問題的資料外傳。
- **流程**：
  1. 用 `ptp_enc.ntfc_record` / `ntfc_detail` 確認一筆 2025 申報存在 Error 或 Warning 通知。
  2. 執行程式。
  3. 檢查該筆是否被排除於待交換清單與輸出 XML 之外。
  4. 檢核：有通知記錄者完全不出現在產出檔案。

### TC-SEL-004 — 排除已傳送（Sent）的記錄
- **目的**：防止已傳送成功（`CTS_TX_REC.TX_STS='T'`）的記錄被重複萃取、重複傳送。
- **流程**：
  1. 準備一筆已標記傳輸完成的 2025 記錄。
  2. 執行程式。
  3. 確認該記錄未再次出現於本次萃取結果。
  4. 檢核：只有「已萃取但尚未傳送」者才會被抓取。

### TC-SEL-005 — 接收國在生效期間內 → 納入
- **目的**：驗證接收國生效日期窗口判斷正確——在窗口內才算合法交換夥伴。
- **流程**：
  1. 設定 `PTP_TAX_JRDT.OUT_DATE_STARTED ≤ 今天 ≤ OUT_DATE_ENDED`（或 ENDED 為 NULL）。
  2. 執行 SQL1（V_CTRY_PARTNER）或整支程式。
  3. 確認 `IS_PARTNER='Y'`，對應記錄被萃取。
  4. 檢核：判斷與萃取結果皆正確。

### TC-SEL-006 — 接收國不在生效期間內 → 排除
- **目的**：驗證窗口外（尚未生效／已失效）的接收國不會被誤判為交換夥伴。
- **流程**：
  1. 設定 `OUT_DATE_STARTED` 為未來日期，或 `OUT_DATE_ENDED` 為過去日期。
  2. 執行 SQL1。
  3. 確認 `IS_PARTNER='N'`，記錄未被萃取。
  4. 檢核：正確排除。

### TC-SEL-007 — 排除等待狀態回覆中的記錄
- **目的**：避免同一筆記錄在等待 OECD 夥伴回覆狀態訊息期間被重複萃取、重複傳送。
- **流程**：
  1. 準備一組已透過 CTBP5002 傳輸（`TRANSMISSION_FLAG='Y'`）但尚未收到 CTBP5004 狀態回覆的 `RPT_YR+REC_JDX_CTRY` 組合。
  2. 執行 SQL2（V_CTRY_TRANSMISSION）確認該組合列在「傳輸中」清單。
  3. 執行程式，確認本次未再次萃取。
  4. 檢核：避免重複傳送。

### TC-SEL-008 — 依 TIN＋接收國＋年度分組產檔
- **目的**：驗證同一批待交換資料依「FilingCE TIN＋接收國＋年度」正確分組、各自產生獨立檔案。
- **流程**：
  1. 準備 2 個不同 FCE_TIN、相同接收國、相同 `RPT_YR=2025` 的待交換記錄。
  2. 執行程式。
  3. 確認產出兩個獨立 XML 檔案（各自 MessageRefId），內容不混雜。
  4. 檢核：分組正確，無跨 TIN 混檔。

### TC-SEL-009 — 待交換清單來源為 SQL7
- **目的**：驗證整個「待交換 key 清單」的權威來源就是 SQL7（V_EXT），確保後續所有步驟都以此為準。
- **流程**：
  1. 準備跨多個 jurisdiction／TIN 的待交換記錄，`RPT_YR=2025`。
  2. 直接執行 SQL7 取得清單。
  3. 比對 CTBP5001 實際處理的組合是否與 SQL7 結果一致。
  4. 檢核：兩者一致，無多也無少。

---

## 2. Jurisdiction Table Date-Window Check

這組聚焦規格書 Description 段落最核心的判斷式：`Start Date ≤ Reporting FY Start Date ≤ Cessation Date`，確保每個 Parent Element（GeneralSection／Summary／JurisdictionSection／UTPRAttribution）針對每個接收國各自正確判斷是否交換。

### TC-JDX-001 — FY 起始日在窗口內 → 交換
- **目的**：驗證窗口判斷式在「命中」情境下確實觸發交換。
- **流程**：
  1. 設定 Jurisdiction Table 的 `Start Date ≤ 2025 FY Start ≤ Cessation Date`。
  2. 執行程式。
  3. 確認該元素被交換到該 jurisdiction。
  4. 檢核：條件成立即交換。

### TC-JDX-002 — FY 起始日早於生效日 → 不交換
- **目的**：驗證窗口判斷式在「還沒生效」情境下正確擋下。
- **流程**：
  1. 設定 `FY Start < Jurisdiction Start Date`。
  2. 執行程式。
  3. 確認元素未被交換。
  4. 檢核：正確排除。

### TC-JDX-003 — FY 起始日晚於終止日 → 不交換
- **目的**：驗證窗口判斷式在「已經失效」情境下正確擋下。
- **流程**：
  1. 設定 `FY Start > Cessation Date`。
  2. 執行程式。
  3. 確認元素未被交換。
  4. 檢核：正確排除。

### TC-JDX-004 — 接收國不在 Jurisdiction Table → 跳過
- **目的**：驗證接收國根本不在表內時，不會被誤判為可交換。
- **流程**：
  1. 準備一個不存在於 `PTP_TAX_JRDT` 的接收國。
  2. 執行 SQL1，確認不在結果內／`IS_PARTNER='N'`。
  3. 執行程式，確認元素未被交換。
  4. 檢核：安全跳過，不報錯。

### TC-JDX-005 — 只有 UPE／DFE 申報的 GIR 才萃取
- **目的**：驗證只有最終母公司（UPE）或指定申報實體（DFE）提交的 GIR 才進入萃取範圍，一般 Constituent Entity 申報不該被交換。
- **流程**：
  1. 用 `FE_ROLE` 篩選語法準備一筆 `FE_ROLE` 非 UPE 也非 DFE 的申報。
  2. 執行程式。
  3. 確認該 GIR 完全未被納入萃取範圍。
  4. 檢核：正確排除。

### TC-JDX-006 — 多接收國元素逐國判斷
- **目的**：驗證同一元素要交換給多個接收國時，程式會逐一國家分別判斷視窗，而不是套用單一結果到全部國家。
- **流程**：
  1. 準備一個元素同時要送往 Jurisdiction A（窗口內）與 B（窗口外），`RPT_YR=2025`。
  2. 執行程式。
  3. 確認只有 A 收到該元素，B 沒有。
  4. 檢核：逐國判斷正確。

---

## 3. Correction-Before-Extraction Merge

對應規格書「原始檔案 vs 更正檔案」範例——如果 GIR 在被萃取前已有部分元素被更正，萃取時要抓「各元素各自最新版本」，而不是整份都用舊版或都用新版。

### TC-COR-001 — 合併更正元素與其餘原始元素
- **目的**：完整重現規格書範例，驗證「更正過的元素取更正版、未更正的元素取原版」這個混合邏輯。
- **流程**：
  1. Day1 送出含五個元素的 Original File；Day2 只送 JurisdictionSection 的 Corrected File，`RPT_YR=2025`，兩者皆 Accepted。
  2. 執行程式。
  3. 檢查輸出：FilingInfo／GeneralSection／Summary／UTPRAttribution 應來自 Original，JurisdictionSection 應來自 Corrected。
  4. 檢核：四個舊、一個新，符合規格範例。

### TC-COR-002 — 接受局部更正（FilingInfo OECD0＋單一元素）
- **目的**：驗證更正申報不需重送整份 GIR，只送 FilingInfo(OECD0) ＋被更正的元素即可被接受並反映在萃取結果。
- **流程**：
  1. 提交一筆只含 FilingInfo(OECD0)＋1 個元素的更正申報（非完整 5 元素）。
  2. 執行程式。
  3. 確認該更正被接受並正確反映在萃取結果中。
  4. 檢核：部分更正生效。

### TC-COR-003 — 一次更正多個元素
- **目的**：驗證合併邏輯在「一次更正多個元素」時仍逐一元素正確取最新版本，不漏也不錯拿。
- **流程**：
  1. 準備 Original ＋ 同時更正 Summary 與 UTPRAttribution 的 Corrected File，`RPT_YR=2025`。
  2. 執行程式。
  3. 確認 Summary、UTPRAttribution 取自 Corrected，其餘取自 Original。
  4. 檢核：多元素合併正確。

---

## 4. Program Modes

驗證程式的兩種執行模式（無參數 Auto／帶參數 Ad-hoc）各自的篩選範圍正確，且對不合法參數有合理防呆。

### TC-MODE-001 — Auto 模式（不帶參數）
- **目的**：確認不帶參數執行時，程式掃描全庫、抓出「所有」符合條件的組合，不遺漏。
- **流程**：
  1. 不帶任何參數執行 CTBP5001。
  2. 資料庫中預先放好分屬不同 TIN／年度／國家的待交換記錄。
  3. 檢查彙總清單是否涵蓋所有符合條件的組合。
  4. 檢核：無遺漏。

### TC-MODE-002 — Ad-hoc 模式，合法參數
- **目的**：確認帶入「國家＋年度」參數時，只處理指定組合，不誤觸其他資料。
- **流程**：
  1. 帶入參數 `"US 2025"` 執行。
  2. 資料庫同時存在其他國家／年度資料。
  3. 確認只有 US／2025 被處理。
  4. 檢核：範圍精準。

### TC-MODE-003 — Ad-hoc 參數格式錯誤（規格模糊點）
- **目的**：規格書沒明講錯誤格式參數（如 `"USA2025"`）的行為，此案例用來逼出程式實際防呆設計。
- **流程**：
  1. 帶入格式錯誤的參數執行。
  2. 觀察回應：有無清楚錯誤訊息、是否誤處理成其他組合、是否當機。
  3. 記錄實際行為，與開發／業主確認是否符合預期。
  4. 檢核：至少不能靜默誤處理或崩潰。

### TC-MODE-004 — Ad-hoc 只選出正確子集合
- **目的**：確認資料庫混雜多國資料時，Ad-hoc 模式仍只精準抓出指定國家。
- **流程**：
  1. 帶入參數 `"JP 2025"`，資料庫同時有 US/2025 記錄。
  2. 執行程式。
  3. 確認只彙整出 JP/2025，US 記錄完全未被觸及。
  4. 檢核：無誤觸其他國家。

---

## 5. File Generation Order & Grouping

驗證檔案「先分組、再依固定順序產生」的規則，順序錯了會造成夥伴國收到資料時邏輯對不上（例如更正檔比新增檔先到，但要更正的東西還不存在）。

### TC-ORD-001 — 同一組合內的檔案產生順序
- **目的**：驗證同一組合下三種檔案固定依 Filing Entity Correction → New Records → Correction Records 順序產生，不可顛倒。
- **流程**：
  1. 準備一個同時需要三種檔案的 2025 組合（FilingInfo 異動＋新記錄＋既有記錄更正）。
  2. 執行程式。
  3. 檢查三個檔案的產生時間戳記／序號順序。
  4. 檢核：順序正確不顛倒。

### TC-ORD-002 — 分組先於產檔
- **目的**：驗證程式先把記錄依「年度→TIN→接收國」分好組才開始產檔，而非邊掃描邊亂序產檔。
- **流程**：
  1. 準備跨多年度／多 TIN／多國的混合記錄。
  2. 執行程式。
  3. 確認產出檔案清楚對應各自分組，同組合內檔案集中產生。
  4. 檢核：分組先於產檔。

---

## 6. OECD0 Resend Option

驗證 OECD0（沿用既有 DocRefId 的續傳選項）只能用在 FilingInfo 上，且只有該 FilingInfo 之前已送過時才能用。

### TC-RES-001 — FilingInfo 先前已送過 → 允許 OECD0
- **目的**：確認「FilingInfo 先前已成功傳送」時，本次可合法使用 OECD0 沿用舊 DocRefId。
- **流程**：
  1. 準備一筆 FilingInfo 先前已對同一 2025／TIN／國家送過的記錄。
  2. 執行程式產生新一輪檔案。
  3. 確認 `FilingInfo.DocTypeIndic=OECD0`，DocRefId 沿用舊值。
  4. 檢核：允許續傳。

### TC-RES-002 — OECD0 限定用於 FilingInfo
- **目的**：確認 OECD0 不會被誤用在 GeneralSection／Summary 等其他元素——規格明講 OECD0 專屬 FilingInfo。
- **流程**：
  1. 檢查四個 Section 的 DocTypeIndic 邏輯。
  2. 確認產出結果中這四類元素只會出現 OECD1／OECD2／OECD3，不會出現 OECD0。
  3. 檢核：OECD0 僅限 FilingInfo。

---

## 7. 30,000-Record Cap & File Splitting

驗證單檔上限 30,000 筆（四個 Section 合計）的控管，以及分割多檔時 FilingInfo 選項規則的正確性。

### TC-CAP-001 — 未超過上限 → 單檔
- **目的**：確認在上限以內時，程式不會多此一舉切成多檔。
- **流程**：
  1. 準備 25,000 筆（四個 Section 合計）2025 記錄。
  2. 執行程式。
  3. 確認只產出 1 個 XML 檔案。
  4. 檢核：不誤切。

### TC-CAP-002 — 超過上限 → 自動分割
- **目的**：確認超過上限時程式自動切割成多檔案，每檔皆不超過上限。
- **流程**：
  1. 準備 65,000 筆記錄。
  2. 執行程式。
  3. 確認產出多個檔案，逐一加總每檔四個 Section 筆數皆 ≤30,000。
  4. 檢核：正確分割。

### TC-CAP-003 — 分割檔案的 FilingInfo 選項規則
- **目的**：驗證分割成多檔時，第一檔用 OECD1、後續改用 OECD0，且 OECD0 的 DocRefId 須與第一檔 OECD1 完全一致（讓夥伴國知道是同一批延續）。
- **流程**：
  1. 沿用會切成 3 檔的情境執行程式。
  2. 檢查第 1 檔 `FilingInfo.DocTypeIndic=OECD1`，記下其 DocRefId。
  3. 檢查第 2、3 檔 `DocTypeIndic=OECD0`，且 DocRefId 與第 1 檔完全相同。
  4. 檢核：後續檔案正確沿用同一 DocRefId。

### TC-CAP-004 — 邊界值：剛好 30,000 筆（規格模糊點）
- **目的**：規格只寫「不得超過 30,000」，未明講「剛好等於」是否算超過，此為邊界模糊點，需實測並與業主確認。
- **流程**：
  1. 準備剛好 30,000 筆記錄。
  2. 執行程式。
  3. 確認是否產出單一檔案（不切割）。
  4. 檢核：記錄實際邊界行為，若與預期不符需回報釐清。

---

## 8. MessageRefId & Sequence Number

驗證每個檔案的 MessageRefId 唯一性，以及序號 `<9999>` 在同一組合內遞增、跨組合歸零的規則。

### TC-MRI-001 — 每個檔案的 MessageRefId 唯一
- **目的**：確認同一次執行若產出多個檔案，MessageRefId 都不重複。
- **流程**：
  1. 準備會產出多個檔案的情境並執行。
  2. 蒐集所有產出檔案的 MessageRefId。
  3. 確認全部唯一無重複。
  4. 檢核：無重複。

### TC-MRI-002 — 序號從 0001 起始
- **目的**：確認一個全新組合（國家／年度／TIN）第一個檔案的序號固定從 0001 開始。
- **流程**：
  1. 針對全新的 (US, 2025, T123) 組合執行程式產生第一個檔案。
  2. 檢查 DocRefId 末尾的 `<9999>` 部分。
  3. 檢核：等於 0001。

### TC-MRI-003 — 序號依組合各自歸零
- **目的**：確認換到不同組合時序號重新從 0001 起算，不沿用其他組合的序號。
- **流程**：
  1. 針對全新的 (Jurisdiction, 2025, TIN) 組合執行程式。
  2. 檢查序號是否為 0001，不受其他組合影響。
  3. 檢核：獨立歸零。

### TC-MRI-004 — 同一組合內序號依序遞增
- **目的**：確認同一組合下產生多個檔案時，序號依序遞增。
- **流程**：
  1. 對 (US, 2025, T123) 組合執行程式產生 3 個檔案。
  2. 檢查三個檔案序號依序為 0001、0002、0003。
  3. 檢核：正確遞增不跳號不重複。

---

## 9. Filing Entity Correction XML (GIR102) — Step B

驗證 Step B 產生的 Filing Entity Correction 檔案（只更正 FilingInfo，不含四個 Section）欄位規則與資料來源正確。

### TC-FEC-001 — FilingInfo 取自最新 OECD2 記錄
- **目的**：確認 FilingInfo 內容抓的是 SQL6 找出的「最新一筆 OECD2 更正記錄」，而非舊版或其他版本。
- **流程**：
  1. 準備 SQL6（V_FILING_INFO_OECD2）會回傳的最新 OECD2 FilingInfo，`RPT_YR=2025`。
  2. 執行程式（Step B）。
  3. 比對輸出 FilingInfo XML 內容是否等於該筆 SQL6 記錄。
  4. 檢核：內容一致。

### TC-FEC-002 — 不含其他四個 Section 元素
- **目的**：確認 Filing Entity Correction 檔案的 `GLOBEBody` 只有 FilingInfo，不能混入其他四個 Section。
- **流程**：
  1. 產生一份 Filing Entity Correction 檔案。
  2. 檢查 `GLOBEBody` 節點內容。
  3. 檢核：只有 FilingInfo 一個子節點。

### TC-FEC-003 — MessageSpec 欄位正確性
- **目的**：逐欄位驗證 MessageSpec（TransmittingCountry/ReceivingCountry/MessageType/MessageTypeIndic/DocRefId/ReportingPeriod）組值公式正確。
- **流程**：
  1. 代入 `RPT_YR=2025、REC_JDX_CTRY=US、FCE_TIN=T123、日期20260720、序號0001`。
  2. 執行程式。
  3. 比對輸出：`TransmittingCountry=HK、ReceivingCountry=US、MessageType=GIR、MessageTypeIndic=GIR102、DocRefId=HK2025UST123-20260720-0001、ReportingPeriod=2025-12-31`。
  4. 檢核：逐欄位完全吻合。

### TC-FEC-004 — FilingInfo OECD2 欄位
- **目的**：驗證 FilingInfo 元素的 `DocTypeIndic`／`DocRefId`／`CorrDocRefId` 組值正確。
- **流程**：
  1. 沿用同組測試資料。
  2. 檢查輸出：`DocTypeIndic=OECD2`、`DocRefId=HK2025UST123-F-20260720`、`CorrDocRefId=V_LAST_FILING_INFO.FI_DOC_REF_ID`。
  3. 檢核：三個欄位皆正確。

### TC-FEC-005 — FilingInfo XML 內容來源
- **目的**：驗證 FilingInfo 的實際 XML 內容是從 `V_FILING_INFO_OECD2.FI_ID` 對應來源正確帶入，不只是欄位值對。
- **流程**：
  1. 找出 `V_FILING_INFO_OECD2` 中一筆 `FI_ID`。
  2. 執行程式產出對應 XML。
  3. 比對 XML 內容欄位與該 FI_ID 來源資料一致。
  4. 檢核：內容溯源正確。

---

## 10. New Records XML (GIR101) — Step C

驗證 Step C 產生新記錄檔案時，MessageSpec、FilingInfo（首次 vs 已有）、以及四個 Section 元素各自的資料來源與 DocRefId 組值規則。

### TC-NEW-001 — MessageSpec for GIR101
- **目的**：驗證 New Records 檔案的 MessageSpec 欄位（含 `MessageTypeIndic=GIR101`）組值正確。
- **流程**：
  1. 代入 `RPT_YR=2025、US、T123`。
  2. 執行程式（Step C）。
  3. 比對輸出是否符合 §4.5 規則（DocRefId/ReportingPeriod 公式與 GIR102 相同，但 MessageTypeIndic=GIR101）。
  4. 檢核：欄位正確。

### TC-NEW-002 — 無先前 FilingInfo → OECD1
- **目的**：驗證該 TIN 首次對此組合傳送時，FilingInfo 正確標記為 OECD1 並產生全新 DocRefId。
- **流程**：
  1. 確保 (2025,T123,US) 不存在於 `V_LAST_FILING_INFO`。
  2. 執行程式。
  3. 檢查 `DocTypeIndic=OECD1`，DocRefId 含 "-F-"＋全新日期。
  4. 檢核：正確視為首次。

### TC-NEW-003 — 已有先前 FilingInfo → OECD0
- **目的**：驗證該組合已傳送過時，改用 OECD0 沿用舊 DocRefId，不重複產生新 FilingInfo。
- **流程**：
  1. 確保 (2025,T123,US) 存在於 `V_LAST_FILING_INFO`。
  2. 執行程式。
  3. 檢查 `DocTypeIndic=OECD0`，`DocRefId=V_LAST_FILING_INFO.FI_DOC_REF_ID`。
  4. 檢核：正確沿用。

### TC-NEW-004 — GeneralSection 元素（GIR101）
- **目的**：驗證 GeneralSection 的 XML 內容來源與 DocRefId 後綴 "-G-" 組值規則。
- **流程**：
  1. 取一筆 `V_EXT_GSEC_GIR101.GS_ID`。
  2. 執行程式。
  3. 比對 XML 內容是否來自 `PTP_GEN_SEC` 該 GS_ID，DocRefId 含 "-G-"。
  4. 檢核：來源與格式皆正確。

### TC-NEW-005 — JurisdictionSection 元素（GIR101）
- **目的**：同 TC-NEW-004 邏輯，換成 JurisdictionSection，後綴應為 "-J-"。
- **流程**：
  1. 取一筆 `V_EXT_JSEC_GIR101.JS_ID`。
  2. 執行程式。
  3. 比對內容來自 `PTP_JDX_SEC`，DocRefId 含 "-J-"。
  4. 檢核：正確。

### TC-NEW-006 — Summary 元素（GIR101）
- **目的**：同上邏輯，換成 Summary，後綴 "-S-"。
- **流程**：
  1. 取一筆 `V_EXT_SMRY_GIR101.SMRY_ID`。
  2. 執行程式。
  3. 比對內容來自 `PTP_SMRY`，DocRefId 含 "-S-"。
  4. 檢核：正確。

### TC-NEW-007 — UTPRAttribution 元素（GIR101）
- **目的**：同上邏輯，換成 UTPRAttribution，後綴 "-U-"。
- **流程**：
  1. 取一筆 `V_EXT_UTPR_GIR101.UPAT_ID`。
  2. 執行程式。
  3. 比對內容來自 `PTP_UTPR_ATTR`，DocRefId 含 "-U-"。
  4. 檢核：正確。

### TC-NEW-008 — 新記錄來源為 V_EXT_GIR101（SQL3）
- **目的**：驗證整個 New Records 檔案的資料來源確實是 SQL3 這組 View（OECD_1 分類），而非誤用其他 View。
- **流程**：
  1. 準備符合 SQL3 分類（OECD_1）的 2025 記錄。
  2. 執行程式。
  3. 確認輸出四個 Section 內容分別對應 SQL3 各 View 的 GS_ID/JS_ID/SMRY_ID/UPAT_ID。
  4. 檢核：來源正確無誤用。

---

## 11. Correction & Deletion Records XML (GIR102) — Step D

驗證 Step D 產生更正／刪除記錄檔案時的 MessageSpec、FilingInfo、四個 Section（含 CorrDocRefId）以及刪除記錄（OECD3）的正確性。

### TC-CRC-001 — MessageSpec for GIR102 更正檔
- **目的**：驗證更正檔案的 MessageSpec 欄位規則（`MessageTypeIndic=GIR102`）正確。
- **流程**：
  1. 準備 `RPT_YR=2025` 群組資料。
  2. 執行程式（Step D）。
  3. 比對欄位符合 §4.6 規則。
  4. 檢核：正確。

### TC-CRC-002 — 無先前 FilingInfo → OECD1
- **目的**：同 New Records 邏輯，驗證更正檔案裡首次傳送的 FilingInfo 也用 OECD1。
- **流程**：
  1. 確保無 `V_LAST_FILING_INFO` 記錄。
  2. 執行程式。
  3. 檢查 `DocTypeIndic=OECD1`，新 DocRefId。
  4. 檢核：正確。

### TC-CRC-003 — 已有先前 FilingInfo → OECD0
- **目的**：驗證已有 FilingInfo 時正確使用 OECD0 沿用。
- **流程**：
  1. 確保存在 `V_LAST_FILING_INFO` 記錄。
  2. 執行程式。
  3. 檢查 `DocTypeIndic=OECD0`，DocRefId 沿用。
  4. 檢核：正確。

### TC-CRC-004 — GeneralSection 更正需帶 CorrDocRefId
- **目的**：驗證更正版 GeneralSection 除 `DocTypeIndic=OECD2/OECD3`、DocRefId 含 "-G-" 外，還要正確帶出 `CorrDocRefId`（指向被更正的原始 DocRefId），這是更正檔案跟新增檔案最大的差異。
- **流程**：
  1. 取一筆 `V_EXT_GSEC_GIR102.GS_ID`。
  2. 執行程式。
  3. 比對 DocTypeIndic、DocRefId 格式、以及 `CorrDocRefId=LAST_CTS_DOC_REF_ID`。
  4. 檢核：三者皆正確。

### TC-CRC-005 — JurisdictionSection 更正
- **目的**：同上邏輯換成 JurisdictionSection，後綴 "-J-"。
- **流程**：
  1. 取 `V_EXT_JSEC_GIR102.JS_ID`。
  2. 執行程式。
  3. 比對三欄位。
  4. 檢核：正確。

### TC-CRC-006 — Summary 更正
- **目的**：同上換成 Summary，後綴 "-S-"。
- **流程**：
  1. 取 `V_EXT_SMRY_GIR102.SMRY_ID`。
  2. 執行程式。
  3. 比對三欄位。
  4. 檢核：正確。

### TC-CRC-007 — UTPRAttribution 更正
- **目的**：同上換成 UTPRAttribution，後綴 "-U-"。
- **流程**：
  1. 取 `V_EXT_UTPR_GIR102.UPAT_ID`。
  2. 執行程式。
  3. 比對三欄位。
  4. 檢核：正確。

### TC-CRC-008 — 刪除記錄（OECD3）萃取
- **目的**：驗證「刪除」這種特殊更正（SQL4 的 DELETE RECORDS 分支）能正確被萃取進 Correction & Deletion 檔案，標記為 OECD3。
- **流程**：
  1. 準備一筆落在 `V_EXT_*_GIR102` DELETE 分支的記錄（原始 DOC_TYPE_IND 屬於 OECD_0/1/2/3），`RPT_YR=2025`。
  2. 執行程式。
  3. 確認該記錄出現在更正檔案中，`DocTypeIndic=OECD3`。
  4. 檢核：刪除語意正確表達。

---

## 12. Database Update — Step E

驗證 XML 產生後回寫資料庫（GIR_MSG_SPEC 及六張明細表）的欄位對應與稽核欄位是否正確，這是保證「XML 內容」與「資料庫紀錄」一致的最後一關。

### TC-DBU-001 — GIR_MSG_SPEC 初始寫入值
- **目的**：驗證每次產生訊息時，`GIR_MSG_SPEC` 初始狀態欄位（`APRVD_STS='P'`、`SGD_STS=NULL`、`GIR_CNT`/`AMEND_CNT`）正確寫入。
- **流程**：
  1. 執行程式產生一個 2025 GIR 訊息。
  2. 查詢 `GIR_MSG_SPEC` 該筆記錄。
  3. 確認 `APRVD_STS='P'`、`SGD_STS=NULL`，計數欄位已由 `UPDATE_TX_CNT` 計算。
  4. 檢核：初始狀態符合規格。

### TC-DBU-002 — GIR_CNT 由 Stored Procedure 更新
- **目的**：獨立驗證 `GIR_CNT` 確實由 `UPDATE_TX_CNT` 更新，而非寫死或漏更新。
- **流程**：
  1. 插入 GIR101 記錄。
  2. 執行程式。
  3. 查詢 `GIR_MSG_SPEC.GIR_CNT` 是否正確反映筆數。
  4. 檢核：計數正確。

### TC-DBU-003 — AMEND_CNT 由 Stored Procedure 更新
- **目的**：同上，驗證 `AMEND_CNT` 在插入 GIR102 記錄後正確更新。
- **流程**：
  1. 插入 GIR102 記錄。
  2. 執行程式。
  3. 查詢 `GIR_MSG_SPEC.AMEND_CNT`。
  4. 檢核：計數正確。

### TC-DBU-004 — FILING_INFO 欄位映射
- **目的**：驗證 `FILING_INFO` 表每個欄位（ID、DOC_REF_ID、CORR_DOC_REF_ID、DOC_TYPE_IND、FCE_TIN、MS_ID）都正確對應到 XML 的 `/FilingInfo/DocSpec` 內容。
- **流程**：
  1. 產生任一 FilingInfo XML 元素。
  2. 查詢 `FILING_INFO` 表對應記錄。
  3. 逐欄比對。
  4. 檢核：全部欄位一致。

### TC-DBU-005 — GEN_SEC 欄位映射
- **目的**：同上邏輯換成 `GEN_SEC` 表，驗證 `PTP_GS_ID`／`PTP_FI_ID`／`FILING_INFO_ID` 等外鍵關聯正確。
- **流程**：
  1. 產生 GeneralSection 元素。
  2. 查詢 `GEN_SEC` 表。
  3. 逐欄比對含外鍵關聯。
  4. 檢核：一致。

### TC-DBU-006 — JDX_SEC 欄位映射
- **目的**：同上換成 `JDX_SEC` 表。
- **流程**：
  1. 產生 JurisdictionSection 元素。
  2. 查詢 `JDX_SEC` 表。
  3. 逐欄比對。
  4. 檢核：一致。

### TC-DBU-007 — SMRY 欄位映射
- **目的**：同上換成 `SMRY` 表。
- **流程**：
  1. 產生 Summary 元素。
  2. 查詢 `SMRY` 表。
  3. 逐欄比對。
  4. 檢核：一致。

### TC-DBU-008 — UTPR_ATTR 欄位映射
- **目的**：同上換成 `UTPR_ATTR` 表。
- **流程**：
  1. 產生 UTPRAttribution 元素。
  2. 查詢 `UTPR_ATTR` 表。
  3. 逐欄比對。
  4. 檢核：一致。

### TC-DBU-009 — 記錄確實寫入 GIR_* 相關表
- **目的**：確認每次產生 XML 後記錄真的有寫進資料庫（不是只產檔不寫庫），六張表都要有對應新增。
- **流程**：
  1. 產生一份完整 XML 檔案（含四個 Section）。
  2. 查詢 `GIR_MSG_SPEC`、`FILING_INFO`、`GEN_SEC`、`JDX_SEC`、`SMRY`、`UTPR_ATTR` 六張表。
  3. 確認皆有新增記錄。
  4. 檢核：無漏寫。

### TC-DBU-010 — 共通稽核欄位正確填入
- **目的**：驗證六張表共通稽核欄位（`LAST_UPD_AT`/`CREATED_AT`=當下時間、`LAST_UPD_BY`/`CREATED_BY`=2）都正確填入。
- **流程**：
  1. 任選一筆剛新增的記錄。
  2. 檢查上述四個稽核欄位。
  3. 確認時間為執行當下、BY 欄位=2。
  4. 檢核：稽核欄位齊全正確。

---

## 13. XML Validation & Character Handling — Step F

驗證特殊字元轉義、不可接受字元阻擋、以及輸出 XML 的 schema 合規性，這關乎資料能否通過 OECD 端的威脅掃描與 schema 驗證，出錯會導致整批檔案被拒收。

### TC-VAL-001 — `&` 轉義為 `&amp;`
- **目的**：驗證 `&` 正確轉義。
- **流程**：
  1. 準備內容含 `&` 的來源資料。
  2. 執行程式產生 XML。
  3. 檢查輸出是否已轉為 `&amp;`。
  4. 檢核：轉義正確。

### TC-VAL-002 — `<` 轉義為 `&lt;`
- **目的**：驗證 `<` 正確轉義。
- **流程**：同上模式，字元換成 `<`，預期輸出 `&lt;`。

### TC-VAL-003 — `>` 轉義為 `&gt;`
- **目的**：驗證 `>` 正確轉義。
- **流程**：同上模式，字元換成 `>`，預期輸出 `&gt;`。

### TC-VAL-004 — `'` 轉義為 `&apos;`
- **目的**：驗證 `'` 正確轉義。
- **流程**：同上模式，字元換成 `'`，預期輸出 `&apos;`。

### TC-VAL-005 — `"` 轉義為 `&quot;`
- **目的**：驗證 `"` 正確轉義。
- **流程**：同上模式，字元換成 `"`，預期輸出 `&quot;`。

### TC-VAL-006 — 阻擋不可接受字元：雙破折號 `--`
- **目的**：驗證來源資料含 `--`（常見注入手法）時，產出檔案裡不能出現這個組合。
- **流程**：
  1. 準備含 `--` 的來源欄位。
  2. 執行程式。
  3. 檢查產出檔案不含 `--`。
  4. 檢核：正確阻擋。

### TC-VAL-007 — 阻擋不可接受字元：`/*`
- **目的**：同上邏輯，驗證 `/*` 不會出現在產出檔案中。
- **流程**：同上模式，字元換成 `/*`。

### TC-VAL-008 — 阻擋不可接受字元：`&#`
- **目的**：同上邏輯，驗證 `&#`（常用於構造 XML entity 注入）不會出現在產出檔案中。
- **流程**：同上模式，字元換成 `&#`。

### TC-VAL-009 — 輸出符合 OECD GIR XML Schema
- **目的**：確認整份 XML 通過 OECD 官方 GIR XML Schema 驗證，這是能否被夥伴國接受的硬性門檻。
- **流程**：
  1. 產生任一完整 XML 檔案。
  2. 用 OECD GIR XSD 做 schema validation（可用 xmllint 或對應驗證工具）。
  3. 確認無 validation error。
  4. 檢核：完全合規。

### TC-VAL-010 — 三種 XML 檔案類型皆有產出
- **目的**：確認在需要三種檔案類型的情境下，程式真的全部產出，不會漏產其中一種。
- **流程**：
  1. 準備一個同時需要三種檔案的 2025 情境。
  2. 執行程式。
  3. 清點產出檔案，確認 Filing Entity Correction／New Records／Correction & Deletion 三種都有。
  4. 檢核：三種齊全。

### TC-VAL-011 — 轉義與阻擋機制的交互作用（規格模糊點）
- **目的**：這是規格模糊點——若內容同時含 `&` 和「本來要組成 `&#` 的片段」，轉義的執行順序有沒有可能反而「製造出」`&#` 這個不允許的組合、繞過威脅掃描？需要實測確認兩道機制的先後順序不會互相破壞。
- **流程**：
  1. 準備內容同時含 `&` 與一個會被誤組成 `&#` 的片段。
  2. 執行程式。
  3. 檢查最終輸出是否仍不含 `&#`，且轉義正確。
  4. 檢核：兩機制無衝突，若有異常需回報釐清順序規則。

---

## 14. File Placement / Naming — Step F

驗證產出檔案的實體路徑與命名規則正確，這樣 CTAP5002 核准流程才找得到檔案。

### TC-FILE-001 — Payload 檔案路徑正確性
- **目的**：驗證 Payload XML 檔案正確放置於 `\data\out\pending\<MessageRefId>.xml`。
- **流程**：
  1. 執行程式產生一筆 `MessageRefId=MR001` 的檔案。
  2. 透過 WinSCP 連到 Outbox／out 相關路徑檢查。
  3. 確認檔案存在於 `\data\out\pending\MR001.xml`。
  4. 檢核：路徑與檔名正確。

### TC-FILE-002 — Metadata 檔案路徑正確性
- **目的**：驗證對應 Metadata 檔案正確放置於同路徑、命名為 `<MessageRefId>.metadata.xml`，且與 Payload 成對存在。
- **流程**：
  1. 沿用 TC-FILE-001 的 MessageRefId。
  2. 檢查 `\data\out\pending\MR001.metadata.xml` 是否存在。
  3. 確認 Payload 與 Metadata 成對出現。
  4. 檢核：成對且命名正確。

---

## 15. DocRefId Construction（跨步驟共用公式）

這組把散落在 Step B/C/D 各處的 DocRefId 組值公式抽出來單獨驗證，確保「同一套公式」在不同 Section 上都套用一致，不會有的地方對、有的地方錯。

### TC-DRI-001 — MessageSpec DocRefId
- **目的**：驗證 MessageSpec 層級 DocRefId 公式 `HK+RPT_YR+REC_JDX_CTRY+FCE_TIN+'-'+<yyyyMMdd>+'-'+<9999>` 正確。
- **流程**：
  1. 代入 `RPT_YR=2025、US、T123、日期20260720、序號0001`。
  2. 執行程式。
  3. 比對輸出是否為 `HK2025UST123-20260720-0001`。
  4. 檢核：字串完全吻合。

### TC-DRI-002 — FilingInfo DocRefId 含 `-F-`
- **目的**：驗證 FilingInfo 的 DocRefId 公式含 `-F-` 後綴正確。
- **流程**：
  1. 沿用同組輸入。
  2. 比對輸出是否為 `HK2025UST123-F-20260720`。
  3. 檢核：吻合。

### TC-DRI-003 — GeneralSection DocRefId 含 `-G-`
- **目的**：驗證 GeneralSection DocRefId 公式除 `-G-` 後綴外，還要正確附加原始 DOC_REF_ID。
- **流程**：
  1. 沿用輸入，另加 `V_EXT_GSEC_GIR101.DOC_REF_ID=7`。
  2. 比對輸出是否為 `HK2025UST123-G-20260720-7`。
  3. 檢核：吻合。

### TC-DRI-004 — 各 Section 後綴字母正確
- **目的**：驗證四個 Section 各自的後綴字母（`-G-`/`-J-`/`-S-`/`-U-`）對應正確，不會混用。
- **流程**：
  1. 分別產生四個 Section 的元素。
  2. 檢查各自 DocRefId 中的後綴字母是否正確對應 GeneralSection/JurisdictionSection/Summary/UTPRAttribution。
  3. 檢核：四個都對。

### TC-DRI-005 — ReportingPeriod 組值
- **目的**：驗證 ReportingPeriod 固定為「年度+12-31」的組值公式正確。
- **流程**：
  1. 代入 `RPT_YR=2025`。
  2. 比對輸出是否為 `2025-12-31`。
  3. 檢核：吻合。

---

## 16. Negative / Edge Cases from Spec Ambiguities

這組是從規格書裡挑出「文字本身有矛盾、命名不一致、或沒講清楚」的地方，專門設計來逼出程式實際行為並跟業主／開發確認，是整份測試最有「找 bug 價值」的一組。

### TC-EDG-001 — OECD 值拼法一致性（OECD1 vs OECD_1 vs OECD_11）
- **目的**：規格書 Step B/C/D 表格用 `OECD1`/`OECD2`（無底線），但附錄二流程圖用 `OECD_1`/`OECD_11`（有底線、且有擴充碼），需確認實際輸出與資料庫比對邏輯用的是哪一種拼法、是否一致。
- **流程**：
  1. 準備 `DOC_TYPE_IND='OECD_1'` 的來源記錄，`RPT_YR=2025`。
  2. 執行程式。
  3. 檢查輸出 XML 的 DocTypeIndic 拼法是否與內部比對邏輯（GIR_CNT 判斷式用的是 OECD_1/OECD_11）一致，schema 是否接受。
  4. 檢核：拼法一致且 schema-valid，若不一致需回報釐清。

### TC-EDG-002 — View 命名 typo 不影響解析
- **目的**：規格書附錄一 SQL5 的 View 名稱寫成 `V_LAST_FILING_INO`（少一個 F），但其他地方都引用 `V_LAST_FILING_INFO`，需驗證實際系統是否已修正、不會因命名對不上而查詢失敗。
- **流程**：
  1. 查詢實際部署的 View 名稱。
  2. 執行任一會用到「最後受理 FilingInfo」的測試案例（如 TC-NEW-003）。
  3. 確認查詢能正確 resolve，未因 typo 失敗。
  4. 檢核：功能正常，若發現 typo 未修正需回報。

### TC-EDG-003 — 查無任何待交換記錄
- **目的**：驗證資料庫完全沒有符合條件記錄時，程式能優雅結束，不報錯、不產生垃圾檔案或髒寫入。
- **流程**：
  1. 確保 2025 Auto 模式下沒有任何符合條件的記錄。
  2. 執行程式。
  3. 確認程式正常結束、無產出檔案、無資料庫寫入、無錯誤訊息。
  4. 檢核：優雅處理空結果。

### TC-EDG-004 — Ad-hoc 指定非有效交換夥伴國家
- **目的**：驗證 Ad-hoc 模式指定一個目前非有效交換夥伴的國家時，不會誤把該國記錄萃取出來。
- **流程**：
  1. 帶入一個 `IS_PARTNER='N'` 的國家參數執行。
  2. 執行程式。
  3. 確認該國沒有任何記錄被萃取。
  4. 檢核：正確排除。

---

## 17. Application Flow — Core Decision Logic

這組直接對應規格書第 3 節的流程圖，逐一驗證「判斷模式→分組→MessageSpec→三選一分支→迴圈結束」這條主線邏輯的每一步。

### TC-AF-001 — 「有無輸入參數？」→ No 分支
- **目的**：驗證流程圖裡「無參數」分支，確認走 Auto 模式抓取全部符合條件的 TIN。
- **流程**：
  1. 不帶參數執行程式。
  2. 確認取得所有符合條件的 FilingCE TIN＋對應年度與國家。
  3. 檢核：對應 Img2 No 分支行為。

### TC-AF-002 — 「有無輸入參數？」→ Yes 分支
- **目的**：驗證流程圖裡「有參數」分支，確認走 Ad-hoc 模式。
- **流程**：
  1. 帶入 `"US 2025"` 執行。
  2. 確認只取得指定的 TIN 清單。
  3. 檢核：對應 Img2 Yes 分支行為。

### TC-AF-003 — 分組步驟
- **目的**：驗證流程圖裡「分組」這一步確實發生在產檔之前。
- **流程**：
  1. 準備混合多組合的 2025 記錄。
  2. 執行程式。
  3. 確認記錄先被分組（TIN／年度／國家）才進入產檔迴圈。
  4. 檢核：順序正確。

### TC-AF-004 — MessageSpec 產生粒度（規格模糊點）
- **目的**：規格圖上寫「每個 Tax Jurisdiction 與 Fiscal Year 組合各產生一個 MessageSpec」，但未明講是否要再細分到 TIN，需實測確認產生粒度。
- **流程**：
  1. 準備 US/2025 與 JP/2025 兩組各自需要產檔的資料。
  2. 執行程式。
  3. 檢查 MessageSpec 是否確實以「每個 XML 檔案」為單位各自產生，並確認是否也依 TIN 區分。
  4. 檢核：釐清粒度並確認符合預期。

### TC-AF-005 — 迴圈分支 1：FilingInfo 異動 → Filing Entity Correction
- **目的**：驗證迴圈內第一個判斷分支正確導向只含 FilingInfo 的 GLOBEBody。
- **流程**：
  1. 準備一筆只有 FilingInfo 異動的 2025 記錄。
  2. 執行程式。
  3. 確認產出 GLOBEBody 只含 FilingInfo。
  4. 檢核：分支 1 正確。

### TC-AF-006 — 迴圈分支 2：新記錄 → New Records
- **目的**：驗證第二個判斷分支（無 FilingInfo 變動但有新記錄）正確導向含新增四個 Section 的 GLOBEBody。
- **流程**：
  1. 準備無 FilingInfo 變動但有新 Section 資料的記錄。
  2. 執行程式。
  3. 確認 GLOBEBody 含 FilingInfo＋新增的四個 Section。
  4. 檢核：分支 2 正確。

### TC-AF-007 — 迴圈分支 3：更正記錄 → Correction Records
- **目的**：驗證第三個判斷分支（無 FilingInfo 變動、無新記錄，但有更正資料）正確導向含更正四個 Section 的 GLOBEBody。
- **流程**：
  1. 準備符合此情境的記錄。
  2. 執行程式。
  3. 確認 GLOBEBody 含 FilingInfo＋更正的四個 Section。
  4. 檢核：分支 3 正確。

### TC-AF-008 — 分支優先順序（FilingInfo 優先，規格模糊點）
- **目的**：規格圖三個分支的判斷順序隱含「FilingInfo 優先」，這裡用一筆同時符合 FilingInfo 異動＋Section 異動的記錄來確認實際優先順序，避免程式誤判走錯分支。
- **流程**：
  1. 準備一筆同時有 FilingInfo 變動與 Section 變動的記錄。
  2. 執行程式。
  3. 確認被導向 Filing Entity Correction 分支（FilingInfo 優先於新增／更正判斷）。
  4. 檢核：優先順序符合預期，若不符需回報釐清。

### TC-AF-009 — 迴圈結束條件
- **目的**：驗證迴圈的結束條件（End of records?）正確運作，處理完所有記錄才結束。
- **流程**：
  1. 準備多筆待處理記錄。
  2. 執行程式。
  3. 確認迴圈持續處理直到全部記錄跑完才進入 END。
  4. 檢核：無提早結束或漏記錄。

### TC-AF-010 — 三個分支皆不符合
- **目的**：驗證一筆三個分支都不符合（無 FilingInfo 異動、無新記錄、無更正）的記錄會被正確跳過，不會被誤塞進任何 GLOBEBody。
- **流程**：
  1. 準備完全無異動的記錄。
  2. 執行程式。
  3. 確認該記錄未出現在任何產出 GLOBEBody 中，迴圈繼續處理下一筆。
  4. 檢核：正確跳過。

---

## 18. GIR Count Logic（GIR_CNT）

驗證 `GIR_CNT` 這個計數欄位的判斷邏輯——只有「真正的新申報（GIR101 ＋ OECD_1/11）」才算一筆 GIR，逐一情境驗證計或不計。

### TC-GCNT-001 — GIR_101 ＋ FilingInfo OECD_1 → 計入
- **目的**：驗證最基本的「新申報」情境會被計入 `GIR_CNT`。
- **流程**：
  1. 準備 `MessageTypeIndic='GIR_101'`、`FilingInfo.DocTypeIndic='OECD_1'` 的記錄。
  2. 執行程式。
  3. 確認 `GIR_CNT +1`。
  4. 檢核：正確計入。

### TC-GCNT-002 — GIR_101 ＋ FilingInfo OECD_11（擴充碼）→ 計入
- **目的**：規格附錄的擴充碼 `OECD_11` 是否也視同 `OECD_1` 計入，此案例實測確認擴充碼的有效性。
- **流程**：
  1. 準備 `DocTypeIndic='OECD_11'` 的記錄。
  2. 執行程式。
  3. 確認 `GIR_CNT +1`。
  4. 檢核：擴充碼有效，若無法識別需回報釐清。

### TC-GCNT-003 — GIR_101 ＋ FilingInfo OECD_0 → 不計入
- **目的**：驗證「續傳（OECD0）」不算新申報，不該被計入 `GIR_CNT`。
- **流程**：
  1. 準備 `DocTypeIndic='OECD_0'` 的記錄。
  2. 執行程式。
  3. 確認未被計入 `GIR_CNT`。
  4. 檢核：正確排除。

### TC-GCNT-004 — GIR_102 → 不計入 GIR
- **目的**：驗證任何 GIR_102（更正類型）訊息都不算 GIR，不管 FilingInfo 類型為何。
- **流程**：
  1. 準備 `MessageTypeIndic='GIR_102'` 的記錄。
  2. 執行程式。
  3. 確認未被計入 `GIR_CNT`。
  4. 檢核：正確排除。

### TC-GCNT-005 — 同一 MessageSpec 內多筆 FilingInfo
- **目的**：驗證同一 MessageSpec 內有多筆混合類型的 FilingInfo（OECD_1/OECD_11/OECD_0）時，計數邏輯會逐筆正確判斷、加總正確。
- **流程**：
  1. 準備 3 筆 FilingInfo：OECD_1、OECD_11、OECD_0。
  2. 執行程式。
  3. 確認 `GIR_CNT` 正確加總為 2（只有前兩筆計入）。
  4. 檢核：加總正確。

### TC-GCNT-006 — GIR_CNT 正確持久化
- **目的**：確認最終計算結果有正確寫回資料庫（不只是記憶體中算對，DB 也要對）。
- **流程**：
  1. 執行含 N 筆符合條件 FilingInfo 的產檔情境。
  2. 查詢 `GIR_MSG_SPEC.GIR_CNT`。
  3. 確認等於 N。
  4. 檢核：持久化正確。

---

## 19. Correction Count Logic（AMEND_CNT）

這組比 GIR_CNT 更複雜——`AMEND_CNT` 的判斷需先排除 GIR_102（一律不計）、再排除 GIR_101 裡本身就是更正類型的 FilingInfo，最後才檢查 GIR_101 的 OECD_0/10 底下四個子 Section 是否有更正。逐一情境驗證這個多層判斷。

### TC-CCNT-001 — GIR_102 → 計為更正
- **目的**：驗證任何 GIR_102 訊息本身都算一次更正。
- **流程**：
  1. 準備 `MessageTypeIndic='GIR_102'` 的記錄。
  2. 執行程式。
  3. 確認 `AMEND_CNT +1`。
  4. 檢核：正確計入。

### TC-CCNT-002 — GIR_101 ＋ FilingInfo OECD_2 → 計入
- **目的**：驗證 GIR_101 訊息裡若 FilingInfo 本身就是更正類型（OECD_2），也算一次更正。
- **流程**：
  1. 準備 `DocTypeIndic='OECD_2'` 的記錄。
  2. 執行程式。
  3. 確認 `AMEND_CNT +1`。
  4. 檢核：正確計入。

### TC-CCNT-003 — GIR_101 ＋ FilingInfo OECD_12（擴充碼）→ 計入
- **目的**：驗證擴充碼 `OECD_12` 是否也視同 `OECD_2` 計入。
- **流程**：
  1. 準備 `DocTypeIndic='OECD_12'` 的記錄。
  2. 執行程式。
  3. 確認 `AMEND_CNT +1`。
  4. 檢核：擴充碼有效。

### TC-CCNT-004 — GIR_101 ＋ FilingInfo OECD_3 → 計入
- **目的**：驗證 FilingInfo 為刪除類型（OECD_3）也算一次更正。
- **流程**：
  1. 準備 `DocTypeIndic='OECD_3'` 的記錄。
  2. 執行程式。
  3. 確認 `AMEND_CNT +1`。
  4. 檢核：正確計入。

### TC-CCNT-005 — GIR_101 ＋ FilingInfo OECD_13（擴充碼）→ 計入
- **目的**：驗證擴充碼 `OECD_13` 也視同 `OECD_3` 計入。
- **流程**：
  1. 準備 `DocTypeIndic='OECD_13'` 的記錄。
  2. 執行程式。
  3. 確認 `AMEND_CNT +1`。
  4. 檢核：擴充碼有效。

### TC-CCNT-006 — FilingInfo OECD_0/10 ＋ 某 Section OECD_1/11 → 計入
- **目的**：驗證最複雜的情境——FilingInfo 本身是續傳（OECD_0），但底下四個 Section 之一是新增／更正（OECD_1/11），這種情況仍要算一次更正。
- **流程**：
  1. 準備 `FilingInfo=OECD_0`、`GeneralSection=OECD_1` 的記錄。
  2. 執行程式。
  3. 確認 `AMEND_CNT +1`。
  4. 檢核：正確計入。

### TC-CCNT-007 — FilingInfo OECD_0 ＋ 四個 Section 皆非 OECD_1/11 → 不計入
- **目的**：驗證反例——FilingInfo=OECD_0 但四個 Section 都不是 OECD_1/11 時，不該被計入。
- **流程**：
  1. 準備 `FilingInfo=OECD_0`、四個 Section 皆非 OECD_1/11 的記錄。
  2. 執行程式。
  3. 確認未被計入 `AMEND_CNT`。
  4. 檢核：正確排除。

### TC-CCNT-008 — FilingInfo OECD_1 不計為更正
- **目的**：驗證 `FilingInfo=OECD_1`（純新增，非更正）不該被誤算進 `AMEND_CNT`——這種情況應只計入 `GIR_CNT`（見 TC-GCNT-001），兩個計數欄位互斥。
- **流程**：
  1. 準備 `FilingInfo=OECD_1` 的記錄。
  2. 執行程式。
  3. 確認 `AMEND_CNT` 未 +1（但 `GIR_CNT` 應 +1）。
  4. 檢核：兩計數欄位互斥正確。

### TC-CCNT-009 — 子項檢查順序／單次遞增
- **目的**：驗證當 `FilingInfo=OECD_0` 且「多個」子 Section 同時是 OECD_1（如 Summary 與 UTPRAttribution 都是）時，這筆 FilingInfo 只計一次，不會因兩個 Section 都命中而重複加 2。
- **流程**：
  1. 準備 `FilingInfo=OECD_0`、Summary 與 UTPRAttribution 皆為 OECD_1 的記錄。
  2. 執行程式。
  3. 確認 `AMEND_CNT` 只 +1（非 +2）。
  4. 檢核：單次遞增，判斷順序符合規格（GeneralSection→Summary→JurisdictionSection→UTPRAttribution，任一命中即跳出）。

### TC-CCNT-010 — AMEND_CNT 正確持久化
- **目的**：確認最終 `AMEND_CNT` 計算結果正確寫回資料庫。
- **流程**：
  1. 執行含 M 筆符合更正條件 FilingInfo 的產檔情境。
  2. 查詢 `GIR_MSG_SPEC.AMEND_CNT`。
  3. 確認等於 M。
  4. 檢核：持久化正確。

---

## 20. End-to-End Outbound Workflow & Status Return

這組跳脫 CTBP5001 單一程式本身，驗證它在整個外圍流程（CTBP5010→CTBP5024→CTBP5001→CTBP5025→CTAP5002→CTBP5002→CTBP5026→SFTP→CTBP5011→CTBP5004）中的上下游銜接是否正確，確保萃取只是整條鏈的其中一環，不會單獨測過但接不上前後手。

### TC-E2E-001 — 上游流程依序執行
- **目的**：驗證 CTBP5001 執行前的上游程式（CTBP5010 更新文件參照、CTBP5024 報表）確實依序先跑完。
- **流程**：
  1. 以 2025 批次觸發整條上游流程。
  2. 依序確認 CTBP5010 → CTBP5024 → CTBP5001 依序執行完成。
  3. 檢核：順序不顛倒、無跳步。

### TC-E2E-002 — 萃取後報表正確產出
- **目的**：驗證 CTBP5001 執行完後，CTBP5025 報表（待傳輸予 Pillar Two 夥伴之 GIR 報表分析）正確產出。
- **流程**：
  1. 執行完 CTBP5001。
  2. 確認 CTBP5025 報表已產生，內容反映本次萃取結果。
  3. 檢核：報表存在且內容對應。

### TC-E2E-003 — 核准關卡：Approve 路徑
- **目的**：驗證 CTAP5002 核准後，流程正確往下走到 CTBP5002 簽章加密，不會卡住。
- **流程**：
  1. 在 CTAP5002 對待核准資料做 Approve 決定。
  2. 確認流程進入 CTBP5002。
  3. 檢核：核准後正確銜接。

### TC-E2E-004 — 簽章加密＋傳輸前報表
- **目的**：驗證 CTBP5002 簽章加密後，CTBP5026 報表（傳輸前資料封包分析）在傳輸前正確產出。
- **流程**：
  1. 執行 CTBP5002 完成簽章加密。
  2. 確認 CTBP5026 報表在實際透過 SFTP 傳輸前已產生。
  3. 檢核：報表先於傳輸產生。

### TC-E2E-005 — 對外傳輸與狀態更新
- **目的**：驗證資料封包透過 SFTP 成功傳送到 OECD，且 CTBP5011 正確更新傳輸狀態。
- **流程**：
  1. 執行傳輸。
  2. 確認封包送達 OECD 指定路徑。
  3. 確認 CTBP5011 更新對應的傳輸狀態欄位。
  4. 檢核：傳輸成功且狀態同步更新。

### TC-E2E-006 — 夥伴處理與狀態回傳
- **目的**：驗證 Pillar Two 夥伴國下載處理封包後，會透過 CTS 回傳狀態訊息，完成整個閉環。
- **流程**：
  1. 模擬／等待夥伴國下載並處理封包。
  2. 確認夥伴透過 CTS 回傳 Status Message。
  3. 檢核：狀態訊息成功回傳。

### TC-E2E-007 — 核准關卡：Reject 路徑
- **目的**：驗證 CTAP5002 若做出 Reject 決定，流程會正確中止簽章／傳輸，並讓記錄回到待處理狀態（回到 Start 重新處理），而不是誤放行或卡死。（原編號 TC-E2E-008，於整合時修正為連號 007）
- **流程**：
  1. 在 CTAP5002 對待核准資料做 Reject 決定。
  2. 確認未進入 CTBP5002（未簽章、未傳輸）。
  3. 確認相關記錄狀態回到待處理（Pending），可被下次流程重新撿取。
  4. 檢核：拒絕路徑正確中止並可重新處理。

---

*文件說明：本攻略對應 `CS501_CTBP5001_TestCases_Merged.xlsx` 的 120 筆測試案例，執行時請搭配該檔案「Test Cases 測試案例」工作表的 Test Data／Expected Result 欄位取得完整測試資料，並將結果填入 Actual Result 欄位。「測試環境參考 Reference」工作表另有 SQL Developer 查詢語法與 WinSCP 路徑等實際測試環境資訊可供準備測試資料時使用。*
