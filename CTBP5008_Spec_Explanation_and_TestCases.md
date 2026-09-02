# CTBP5008 Report Generation — 功能規格書解讀 + 測試用例 (Test Cases)

> 對應文件：`Functional Specification – CTBP5008 Report Generation v1.0 (working, 2026-06-16)`，全文 **6 頁**，含 **1 張 Program Flow 圖**（EMF 向量圖，Claude 已轉檔並成功讀取圖內容）。
> Part A = 逐節解讀（中文）；Part B = 英文 Test Cases（Test Case ID / Test Scenario / Test Data / Expected Result / Remark；Remark 為出處頁碼/圖）。
> **方向（Direction）**：**內部/報表（Reporting）**——產生 CTS-GloBE 系統各面向的統計報表，非資料交換本身。

---

## 0. 定位
CTBP5008 產生 CTS-GloBE 系統的**統計報表（PDF）**。在 CTBP5001 整體流程圖中出現的 **CTRP5024/5025/5026** 等「Analysis of To-be-Exchanged…」報表即由本程式產生（本程式是通用報表引擎，依 Report ID 產不同報表）。

### 術語速查
| 詞 | 義 |
|---|---|
| **Report ID** | 報表代號（CTRP9999 格式，如 CTRP5024） |
| **Frequency** | 報表頻率：Daily/Weekly/Monthly/Quarterly/Semi-yearly/Yearly/Ad-hoc |
| **Accumulating report** | 累計型報表（ad-hoc 模式下免給 Start Date） |
| **Run Day** | 該頻率的固定執行日 |

---

# Part A — 逐節內容解讀

## A1. Change History（p.1）
單筆：`2026-06-15 Initial version`。v1.0 初版。

## A2. Batch Job Properties（p.1）
- **Batch Job ID**：`CTBP5008`
- **Frequency**：`Daily at 04:00`（每日 04:00 排程）
- **Ad-hoc Run**：`Supported`
- **Ad-hoc Parameters**（三個）：
  - #1 `CTRP9999`：**Report ID**
  - #2 `DDMMYYYY`：**Report Start Date**
  - #3 `DDMMYYYY`：**Report End Date**

## A3. Description（p.1）
本程式產生 CTS-GloBE 系統各面向的統計報表。

## A4. Program Flow（p.2，Program Flow 圖）★ Claude 已讀取圖內容
1. Start
2. **Ad-hoc generation?**
   - **No（排程模式）** → **Get report definitions** → **Determine start date and end date** → **Retrieve statistics** → **Generate PDF report** → **Update next run date** → **All reports generated?**（No → 回「Get report definitions」迴圈；Yes → End）
   - **Yes（ad-hoc 模式）** → **Check Input Parameters** → **Retrieve statistics** → **Generate PDF report** → End

> 排程模式會**逐一跑完所有到期的報表定義**；ad-hoc 模式只跑指定 Report ID 一張。

## A5. 排程日期規則 — Determine Report Start / End / Next Run Date（p.2–4）
排程模式依 run date 與 frequency 推算起訖日；ad-hoc 由參數提供。各頻率規則（皆覆蓋「上一期」）：

| Frequency | Run Day | 覆蓋範圍 | 範例（Run date → Start / End / Next run） |
|---|---|---|---|
| **Daily** | 硬編碼 0（每日） | 前一日全日 | 2019-02-01 → 2019-01-31 00:00:00 / 2019-01-31 23:59:59 / 2019-02-02 |
| **Weekly** | 週日 | 上週日～上週六 | 2019-02-01(五) → 2019-01-20(日) / 2019-01-26(六) / 2019-02-05(日) |
| **Monthly** | 每月 1 日 | 上一整月 | 2019-02-01 → 2019-01-01 / 2019-01-31 23:59:59 / 2019-03-01 |
| **Quarterly** | 季首日 | 上一季 | 2019-01-01 → 2018-10-01 / 2018-12-31 23:59:59 / 2019-04-01 |
| **Semi-yearly** | 1/1 或 7/1 | 前 6 個月 | 2019-01-01 → 2018-07-01 / 2018-12-31 23:59:59 / 2019-07-01 |
| **Yearly** | 1/1 | 前一年 | 2019-01-01 → 2018-01-01 / 2018-12-31 23:59:59 / 2020-01-01 |

> **累計型報表**：ad-hoc 模式下**免給 Start Date**（`*` 註記）。

## A6. List of Reports（p.5–6）
本引擎支援的報表清單（Report ID / 名稱 / 頻率 / 備註）：

| Report ID | 名稱 | 頻率 | 備註 |
|---|---|---|---|
| CTRP5013 | Analysis on GIR Transmitted to GIR Exchange Partners | Ad-hoc | 傳輸後產生 |
| CTRP5014 | Control List of GIR Transmitted to GIR Exchange Partners | Ad-hoc | 傳輸後產生 |
| CTRP5015 | Report on Correction Data Files Transmitted to GIR Exchange Partners | Ad-hoc | 傳輸後產生 |
| CTRP5024 | Analysis of To-be-Exchanged GIR | Ad-hoc | 傳輸前產生 |
| CTRP5025 | Analysis of To-be-Exchanged GIR Pending Transmission to GIR Exchange Partners | Ad-hoc | 傳輸前產生 |
| CTRP5026 | Analysis of Data Packets Containing To-be-Exchanged GIR Reports | Monthly, Ad-hoc | 傳輸前產生 |
| CTRP5027 | Report on GIR Received from GIR Exchange Partners | Monthly (Run Day 1) | |
| CTRP5030 | List of Unmatched HK Entity Cases in GIR Incoming Data File (NOTIN Cases) | Monthly (1) | |
| CTRP5031 | List of Unmatched HK Entity Cases (Empty or All Zeroes Cases) | Monthly (1) | |
| CTRP5032 | List of Unmatched HK Entity Cases (Other Unmatched Cases) | Monthly (1) | |
| CTRP5033 | List of Unmatched HK Entity Cases Updated via CTAP5009 | Monthly (1) | |
| CTRP5034 | Statistics and Analysis on Number of Matched HK Entity Cases in GIR Incoming Data File | Monthly (1) | |
| CTRP5035 | List of GIR Data Files Received from GIR Exchange Partners | Monthly (1) | |
| CTRP5036 | List of GIR Data Files Transmitted but Status Message Not Yet Received | Monthly (1) | |

## A7. 待釐清處
1. **參數格式**：Start/End Date 為 `DDMMYYYY`（非 ISO），測試需以此格式輸入。
2. **累計型報表清單**：規格以 `*` 標示但 List of Reports 表內 Accumulate 欄多為空，建議確認哪些報表屬累計型。
3. **失敗/部分失敗**：排程模式「All reports generated?」迴圈中，若某張報表失敗是否中止或續跑其餘，未明示。

---

# Part B — Test Cases (English)

> Format: **Test Case ID | Test Scenario | Test Data | Expected Result | Remark**. Direction = **Reporting/Internal** for all. "p.N" = page N; "Flow" = Program Flow diagram (p.2).

## B1. Batch Properties & Trigger

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-08-001 | Scheduled daily run | Trigger daily at 04:00 | CTBP5008 runs on the 04:00 daily schedule | p.1 |
| TC-08-002 | Ad-hoc run with all three parameters | #1=CTRP5024, #2=01012025, #3=31012025 | Program runs ad-hoc for CTRP5024 over 01–31 Jan 2025 | p.1 |
| TC-08-003 | Ad-hoc parameter date format | Start/End as DDMMYYYY (e.g. 01012025) | Dates parsed as DDMMYYYY | p.1 (A7-1) |
| TC-08-004 | Accumulating report ad-hoc without start date | Accumulating report ID, only End Date given | Runs without a Start Date (accumulating rule) | p.4 (A7-2) |

## B2. Mode Branching (Program Flow)

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-08-005 | Scheduled mode processes all due reports | Scheduled run with several due reports | Loops: get definitions → determine dates → retrieve stats → generate PDF → update next run date; until all generated | p.2, Flow |
| TC-08-006 | Ad-hoc mode processes a single report | Ad-hoc with Report ID | Check input parameters → retrieve stats → generate PDF → End (single report) | p.2, Flow |
| TC-08-007 | Scheduled mode updates next run date | A report generated on schedule | Next run date updated for that report | p.2, Flow |
| TC-08-008 | Ad-hoc mode does not loop over all reports | Ad-hoc with one Report ID | Only the specified report is generated | p.2, Flow |

## B3. Date-Window Determination (Scheduled)

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-08-009 | Daily window | Run date 2019-02-01 00:00:00 | Start=2019-01-31 00:00:00; End=2019-01-31 23:59:59; Next run=2019-02-02 | p.3 |
| TC-08-010 | Weekly window (runs off-Sunday) | Run date 2019-02-01 (Friday) | Start=2019-01-20 (Sun); End=2019-01-26 23:59:59 (Sat); Next run=2019-02-05 (Sun) | p.3 |
| TC-08-011 | Monthly window | Run date 2019-02-01 | Start=2019-01-01; End=2019-01-31 23:59:59; Next run=2019-03-01 | p.3 |
| TC-08-012 | Quarterly window | Run date 2019-01-01 | Start=2018-10-01; End=2018-12-31 23:59:59; Next run=2019-04-01 | p.3 |
| TC-08-013 | Semi-yearly window | Run date 2019-01-01 | Start=2018-07-01; End=2018-12-31 23:59:59; Next run=2019-07-01 | p.3–4 |
| TC-08-014 | Yearly window | Run date 2019-01-01 | Start=2018-01-01; End=2018-12-31 23:59:59; Next run=2020-01-01 | p.4 |
| TC-08-015 | Weekly always covers Sun–Sat of last week | Any run day in a week | Window is last week's Sunday to Saturday regardless of actual run day | p.3 |

## B4. Report Generation & Output

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-08-016 | Retrieve statistics for window | A determined date window | Statistics retrieved for that window | p.2, Flow |
| TC-08-017 | Generate PDF output | Retrieved statistics | A PDF report is generated | p.2, Flow |
| TC-08-018 | Ad-hoc uses provided window | #2/#3 = 01012025 / 31012025 | Report covers 01–31 Jan 2025 exactly | p.1, p.2 |

## B5. Report Catalogue (List of Reports)

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-08-019 | CTRP5024 To-be-Exchanged GIR (pre-transmission, ad-hoc) | Report ID=CTRP5024 | Generates "Analysis of To-be-Exchanged GIR" before transmission | p.5 |
| TC-08-020 | CTRP5025 Pending Transmission (ad-hoc) | Report ID=CTRP5025 | Generates "Analysis of To-be-Exchanged GIR Pending Transmission" | p.5 |
| TC-08-021 | CTRP5026 Data Packets (Monthly + ad-hoc) | Report ID=CTRP5026 | Generates monthly/ad-hoc "Analysis of Data Packets Containing To-be-Exchanged GIR Reports" | p.5 |
| TC-08-022 | CTRP5013 Transmitted analysis (ad-hoc, post-transmission) | Report ID=CTRP5013 | Generates "Analysis on GIR Transmitted to GIR Exchange Partners" after transmission | p.5 |
| TC-08-023 | CTRP5014 Control List (ad-hoc) | Report ID=CTRP5014 | Generates "Control List of GIR Transmitted…" | p.5 |
| TC-08-024 | CTRP5015 Correction Data Files (ad-hoc) | Report ID=CTRP5015 | Generates "Report on Correction Data Files Transmitted…" | p.5 |
| TC-08-025 | CTRP5027 Received from Partners (Monthly, Run Day 1) | Report ID=CTRP5027 | Generates monthly "Report on GIR Received from GIR Exchange Partners" on day 1 | p.5–6 |
| TC-08-026 | CTRP5030 Unmatched NOTIN cases (Monthly) | Report ID=CTRP5030 | Generates monthly unmatched HK entity (NOTIN) list | p.6 |
| TC-08-027 | CTRP5035 GIR data files received (Monthly) | Report ID=CTRP5035 | Generates monthly "List of GIR Data Files Received…" | p.6 |
| TC-08-028 | CTRP5036 Transmitted but status not received (Monthly) | Report ID=CTRP5036 | Generates monthly "List of GIR Data Files Transmitted but Status Message Not Yet Received" | p.6 |
| TC-08-029 | Unknown Report ID | Report ID not in catalogue | Handled as invalid; no report generated | p.5–6 |

## B6. Edge Cases

| Test Case ID | Test Scenario | Test Data | Expected Result | Remark |
|---|---|---|---|---|
| TC-08-030 | No due reports on a scheduled run | Scheduled run, nothing due | Program completes; no report generated | p.2, Flow |
| TC-08-031 | Ad-hoc with missing required parameter | Report ID given, End Date missing (non-accumulating) | Handled as invalid parameter (confirm behaviour) | p.1 (A7) |
| TC-08-032 | One report fails during scheduled loop | Scheduled loop, one report errors | Confirm whether remaining reports still generate or run aborts | p.2, Flow (A7-3) |

---

### 統計
- 章節：Change History / Batch Job Properties / Description / Program Flow（圖，已判讀）/ 排程日期規則 / List of Reports 全數覆蓋。
- 測試用例：**32 條**，每條標註頁碼/圖。
- 待釐清：見 A7（日期格式、累計型清單、部分失敗處理）。
