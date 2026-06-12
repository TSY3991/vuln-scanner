---
name: vuln-scanner
description: 對本機專案資料夾或開源 GitHub repo 進行安全弱點掃描，涵蓋依賴套件已知漏洞（CVE/OSV）、程式碼層級弱點（SAST）、機密資訊洩漏（API key/token/密碼）三個維度，並產出風險等級摘要與修復建議。當使用者說「弱點掃描」「security scan」「掃描這個專案有沒有漏洞」「幫我看這個 repo 安全嗎」「檢查依賴套件安全性」「掃描這個 GitHub 工具」「這個套件能用嗎/安全嗎」「audit 一下安全性」時務必觸發此 skill，即使使用者只是貼一個 GitHub 連結並問「這個能用嗎」也要觸發。
---

# vuln-scanner

對目標（本機專案資料夾 或 GitHub repo URL）執行三層掃描，產出風險等級摘要與修復建議。

## 第 0 步：確認目標與範圍

- 目標是 **本機路徑** 還是 **GitHub URL**？
- 若是 GitHub URL → clone 到暫存目錄（見「Repo 取得」）。
- 若使用者只給專案名稱沒給路徑，先用 `Glob`/`ls` 確認目前工作目錄下是否有對應資料夾，避免掃錯地方。

## 第 1 步：檢查掃描工具是否已安裝

依序檢查以下三個命令是否存在（PowerShell: `Get-Command <tool> -ErrorAction SilentlyContinue`）：

| 工具 | 用途 | 缺少時的安裝指引 |
|---|---|---|
| `osv-scanner` | 依賴套件已知漏洞（CVE/OSV-DB） | `winget install Google.OSVScanner`（或從 https://github.com/google/osv-scanner/releases 下載執行檔放入 PATH）|
| `semgrep` | SAST 程式碼層級弱點 | `pip install semgrep` |
| `gitleaks` | git history / 檔案中機密資訊洩漏 | `winget install gitleaks`（或從 https://github.com/gitleaks/gitleaks/releases 下載）|

**任一工具缺少時**：

1. 告知使用者缺少哪個工具及其安裝指令。
2. 詢問使用者是否要先安裝（給出指令），或是否跳過該維度繼續掃其他兩項。
3. 不要自己擅自執行安裝指令（`winget install` / `pip install` 屬於改動使用者系統環境，需使用者確認）——這點與一般 bash 操作不同，因為會安裝全域工具。
4. 若使用者明確說「幫我裝」，才執行安裝指令。

若三個工具都已存在，直接進入第 2 步，不用浪費時間提示。

## 第 2 步：Repo 取得（僅當目標是 GitHub URL）

```powershell
$tmp = New-Item -ItemType Directory -Path "$env:TEMP\vuln-scan-$(Get-Random)" -Force
git clone --depth 1 <repo-url> $tmp.FullName
```

掃描完成後，第 4 步報告產出後詢問使用者是否要保留 clone 下來的程式碼（預設清理掉暫存目錄，因為這只是用來掃描，留著佔空間又可能含未審查的程式碼）。

### 2a. OpenSSF Scorecard（僅 GitHub repo 目標，供應鏈健康度）

針對「這個開源工具能不能用」這類問題，光看程式碼本身的漏洞不夠，還要看這個 repo 的維護健康度（多久沒更新、是否有 code review、是否啟用 branch protection 等）——這些是供應鏈風險的早期訊號。

直接查詢 OpenSSF 公開 API（不需安裝任何工具）：

```powershell
curl "https://api.securityscorecards.dev/projects/github.com/<owner>/<repo>"
```

若該 repo 尚未被 Scorecard 收錄會回 404，這時在報告中註明「Scorecard 無資料」即可，不影響其他掃描項目。回傳的 `score`（0-10）與各檢查項（`checks`，如 `Maintained`、`Vulnerabilities`、`Code-Review`、`Branch-Protection`）納入第 4 步報告。

## 第 3 步：執行三層掃描

針對目標目錄（本機路徑或 clone 下來的暫存目錄）依序執行：

### 3a. 依賴套件已知漏洞（osv-scanner）

```powershell
osv-scanner scan source <target-dir>
```

會自動偵測 `package-lock.json` / `requirements.txt` / `go.mod` / `Cargo.lock` 等鎖定檔，比對 OSV 資料庫。

**實測行為**：若專案沒有任何依賴鎖定檔（例如只有 `package.json` 但沒有 `package-lock.json`），會輸出 `No package sources found, --help for usage information.` 並回傳非 0 exit code（不是執行失敗，是「沒東西可掃」）。看到這個訊息時，在報告中註明「無依賴鎖定檔，未執行」即可，不要當成掃描錯誤。

### 3b. SAST 程式碼弱點（semgrep）

```powershell
semgrep --config auto <target-dir>
```

`--config auto` 會依語言自動套用 community ruleset（涵蓋 OWASP Top 10 類型問題如 SQL injection、command injection、XSS、不安全的 deserialization 等），**不需要 `semgrep login`**（未登入會少一些 Registry 額外規則，但 community ruleset 已可正常掃描並回傳 0 findings 之類結果）。專案很大時可能執行較久，先告知使用者預估時間。

**實測行為**：掃描結果預設只統計 git 追蹤的檔案，輸出最後會有 `Scan completed successfully` 與 `Findings: N (M blocking)`。若 `Findings: 0`，報告中直接寫「✅ 未發現程式碼層級弱點」。

### 3c. 機密資訊洩漏（gitleaks）

```powershell
gitleaks detect --source <target-dir> --no-banner
```

若目標是 git repo 會連同歷史 commit 一起掃；若只是一般資料夾（非 git repo），改用：

```powershell
gitleaks detect --source <target-dir> --no-git --no-banner
```

## 第 4 步：整理成風險摘要

掃描結束後，把三個工具的原始輸出整理成**對話內文字摘要**（不需要額外存檔，除非使用者要求）：

```markdown
## 弱點掃描摘要：<目標名稱>

### 風險等級總覽
| 等級 | 數量 | 說明 |
|---|---|---|
| 🔴 Critical | N | ... |
| 🟠 High | N | ... |
| 🟡 Medium | N | ... |
| 🟢 Low/Info | N | ... |

### 依賴套件漏洞（osv-scanner）
- <套件名>@<版本>：<CVE/OSV ID>，<嚴重度>，建議升級至 <版本>
（若無問題：✅ 未發現已知漏洞依賴）

### 程式碼弱點（semgrep）
- <檔案路徑>:<行號> — <規則名稱簡述> — 建議修法

### 機密資訊洩漏（gitleaks）
- <檔案路徑>:<行號> — <類型，如 AWS key / generic API token> — 建議：撤銷並輪替該憑證、加入 .gitignore，必要時改寫 git history

### 供應鏈健康度（OpenSSF Scorecard，僅 GitHub repo）
- 總分：<score>/10
- 關鍵檢查項：`Maintained`（是否持續維護）/ `Code-Review`（是否要求 review）/ `Branch-Protection`/ `Vulnerabilities`
（若無資料：ℹ️ 此 repo 尚未被 Scorecard 收錄）

### 整體建議
依風險等級排序給 1-3 個最優先要處理的項目。
```

排序原則：**Critical/High 優先**、**機密資訊洩漏永遠優先處理**（因為憑證一旦進 git history 就算刪除也視為已洩漏，需輪替憑證而非只刪檔案）。

## 注意事項

- 這是**唯讀分析**工具：掃描本身不修改使用者的程式碼。若使用者要求順手修復，那是另一個任務，先完成掃描報告再詢問是否要修。
- 掃描第三方開源工具時，掃描結果是給使用者參考是否要採用該工具，**不要**自動對外回報漏洞或公開 issue——是否聯絡上游維護者由使用者自行決定。
- semgrep `--config auto` 第一次執行需要網路連線下載 ruleset；若使用者離線，改用 `--config p/owasp-top-ten` 等本機已快取的 ruleset，或告知需要網路。
- 大型 repo（如 monorepo）掃描可能耗時數分鐘，使用 `run_in_background` 執行掃描指令，避免阻塞對話。
