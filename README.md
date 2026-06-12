# vuln-scanner

Claude Code skill：對本機專案資料夾或開源 GitHub repo 進行安全弱點掃描。

## 功能

掃描分為三個維度：

| 維度 | 工具 | 偵測內容 |
|---|---|---|
| 依賴套件已知漏洞 | [osv-scanner](https://github.com/google/osv-scanner) | CVE / OSV 資料庫比對 `package-lock.json`、`requirements.txt`、`go.mod`、`Cargo.lock` 等鎖定檔 |
| 程式碼層級弱點（SAST） | [semgrep](https://github.com/semgrep/semgrep) | SQL injection、command injection、XSS、不安全 deserialization 等 OWASP Top 10 類型問題 |
| 機密資訊洩漏 | [gitleaks](https://github.com/gitleaks/gitleaks) | API key、token、密碼等寫死在程式碼或 git history 中的憑證 |
| 供應鏈健康度（僅 GitHub repo） | [OpenSSF Scorecard](https://github.com/ossf/scorecard) API | 維護是否活躍、是否要求 code review、branch protection 等供應鏈風險訊號 |

掃描結束後在對話中產出**風險等級摘要**（Critical / High / Medium / Low）與**修復建議**，依優先順序排序（機密資訊洩漏永遠優先）。

## Prerequisites

這個 skill 本身不包含掃描工具，需要以下三個命令列工具（皆為獨立的開源工具，非 npm/pip 套件依賴）：

| 工具 | 用途 |
|---|---|
| [osv-scanner](https://github.com/google/osv-scanner) | 依賴套件已知漏洞 |
| [semgrep](https://github.com/semgrep/semgrep) | SAST 程式碼弱點 |
| [gitleaks](https://github.com/gitleaks/gitleaks) | 機密資訊洩漏 |

**不需要先裝**——第一次使用時 Claude 會自動偵測缺少哪些工具並提示安裝指令，你確認後才會安裝。但如果想先裝好，依平台參考下表：

| 工具 | Windows | macOS | Linux |
|---|---|---|---|
| osv-scanner | `winget install Google.OSVScanner` | `brew install osv-scanner` | 下載 [release binary](https://github.com/google/osv-scanner/releases) 放入 PATH |
| semgrep | `pip install semgrep` | `pip install semgrep` 或 `brew install semgrep` | `pip install semgrep` |
| gitleaks | `winget install gitleaks` | `brew install gitleaks` | 下載 [release binary](https://github.com/gitleaks/gitleaks/releases) 放入 PATH，或用套件管理工具（如 apt 需另加 repo）|

> semgrep 在 Windows 上需要 Python 環境（`pip install semgrep` 即可，目前已原生支援 Windows，不需 WSL）。

OpenSSF Scorecard 維度透過公開 API 查詢，**不需安裝任何東西**，僅需網路連線。

## 安裝

把 `vuln-scanner/` 整個資料夾放到 `~/.claude/skills/`（Windows: `C:\Users\<user>\.claude\skills\`），然後參考上方 Prerequisites 安裝掃描工具（或交給 Claude 第一次使用時自動引導安裝）。

## 使用方式

直接跟 Claude 說，例如：

- 「幫我掃描這個專案有沒有漏洞」
- 「掃描這個 GitHub repo：https://github.com/xxx/yyy」
- 「這個套件安全嗎，幫我 audit 一下」

### 流程

1. 確認掃描目標（本機路徑 / GitHub URL）
2. GitHub URL 會 `git clone --depth 1` 到暫存目錄，並查詢 OpenSSF Scorecard 評分
3. 依序執行 osv-scanner → semgrep → gitleaks
4. 產出風險等級摘要、Scorecard 結果與修復建議
5. 若是 clone 下來的暫存 repo，詢問是否清理

## 注意事項

- **唯讀分析**：掃描本身不會修改程式碼，順手修復是另一個任務。
- 掃描第三方開源工具的結果僅供使用者自行評估是否採用，是否聯絡上游維護者由使用者自行決定。
- `semgrep --config auto` 第一次執行需要網路連線下載 ruleset；離線環境請改用 `--config p/owasp-top-ten` 等本機 ruleset。
- 大型 repo 掃描可能耗時數分鐘，建議背景執行。

## License

MIT
