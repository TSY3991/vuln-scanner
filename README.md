# Vuln Scanner

**幫你看一眼，這個專案/這個開源工具，安全嗎。**

> Scan what you're about to depend on.

**作者**：**TSY**
🔗 [Facebook](https://www.facebook.com/TSY.Microglow)
(https://github.com/TSY3991/vuln-scanner)

---

## 起源

開源工具越來越好用，但裝進專案前很少有人真的檢查過：

```
依賴的套件，有沒有已知漏洞？
程式碼本身，有沒有明顯的安全洞？
有沒有人不小心把 API key 寫進 commit 裡？
這個 repo 還有人維護嗎？
```

大多時候我們是看 star 數、看 README 寫得漂不漂亮，就決定要不要用。

這個工具的起點就是這個問題：**與其憑印象判斷「這個工具能不能用」，不如用幾個業界標準的開源掃描器，把已知漏洞、程式碼弱點、機密洩漏、維護健康度這些事實攤開，再由你決定要不要用、要不要先修。**

---

## 中文介紹

Vuln Scanner 是一個 **唯讀分析工具**，串接三個業界標準的開源安全掃描器 + 1 個供應鏈健康度 API。

它不會：

- 修改程式碼
- 自動修復弱點
- 自動對外回報漏洞或開 issue

它只會告訴你：

- 依賴套件有沒有已知漏洞（CVE/OSV）
- 程式碼本身有沒有常見安全弱點（SAST）
- 有沒有機密資訊（API key/token/密碼）洩漏在程式碼或 git history 裡
- （GitHub repo）這個專案的維護健康度如何

---

## 四個維度

| 維度 | 工具 | 偵測內容 |
|---|---|---|
| 依賴套件已知漏洞 | [osv-scanner](https://github.com/google/osv-scanner) | CVE / OSV 資料庫比對 `package-lock.json`、`requirements.txt`、`go.mod`、`Cargo.lock` 等鎖定檔 |
| 程式碼層級弱點（SAST） | [semgrep](https://github.com/semgrep/semgrep) | SQL injection、command injection、XSS、不安全 deserialization 等 OWASP Top 10 類型問題 |
| 機密資訊洩漏 | [gitleaks](https://github.com/gitleaks/gitleaks) | API key、token、密碼等寫死在程式碼或 git history 中的憑證 |
| 供應鏈健康度（僅 GitHub repo） | [OpenSSF Scorecard](https://github.com/ossf/scorecard) API | 維護是否活躍、是否要求 code review、branch protection 等供應鏈風險訊號 |

掃描結束後在對話中產出**風險等級摘要**（Critical / High / Medium / Low）與**修復建議**，依優先順序排序——**機密資訊洩漏永遠優先處理**，因為憑證一旦進 git history，就算刪除也視為已洩漏，需要輪替憑證而非只刪檔案。

---

## 三步驟工作流

```
Step 1: 給目標
   └─ 本機專案路徑，或一個 GitHub repo URL
        ↓
Step 2: 工具依序跑 osv-scanner / semgrep / gitleaks（+ GitHub repo 額外查 Scorecard）
   └─ 三層掃描分別檢查依賴漏洞、程式碼弱點、機密洩漏
        ↓
Step 3: 你看風險摘要，自己決定
   └─ 要不要修？要不要先別用這個套件？
```

**工具只做 Step 1-2。Step 3 永遠是你的判斷。**

---

## 安裝

把 `vuln-scanner/` 整個資料夾放到 `~/.claude/skills/`（Windows: `C:\Users\<user>\.claude\skills\`）。

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
| osv-scanner | `winget install Google.OSVScanner --silent --disable-interactivity` | `brew install osv-scanner` | 下載 [release binary](https://github.com/google/osv-scanner/releases) 放入 PATH |
| semgrep | `python -m pip install semgrep` | `pip install semgrep` 或 `brew install semgrep` | `pip install semgrep` |
| gitleaks | `winget install gitleaks --silent --disable-interactivity` | `brew install gitleaks` | 下載 [release binary](https://github.com/gitleaks/gitleaks/releases) 放入 PATH，或用套件管理工具（如 apt 需另加 repo）|

> semgrep 在 Windows 上需要**完整版 Python**（`python -m pip install semgrep`）。若 `python` 指令指向 Microsoft Store 的 app-execution-alias stub（`python -m pip` 沒輸出、exit code 異常），先 `winget install Python.Python.3.12 --silent --disable-interactivity` 裝真正的 Python。
>
> `winget install` 加 `--silent --disable-interactivity` 可避免冗長的下載進度條輸出。

OpenSSF Scorecard 維度透過公開 API 查詢，**不需安裝任何東西**，僅需網路連線。

---

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

---

## ⚠️ 已知限制

- Scorecard 僅支援 GitHub repo，且該 repo 須已被 [OpenSSF Scorecard](https://github.com/ossf/scorecard) 收錄，否則回 404（無資料，不影響其他三項）
- `semgrep --config auto` 第一次執行需要網路連線下載 ruleset；離線環境請改用 `--config p/owasp-top-ten` 等本機 ruleset
- osv-scanner 僅能比對有鎖定檔（lockfile）的依賴，沒有 `package-lock.json` / `requirements.txt` 等檔案的專案無法檢查依賴漏洞
- 大型 repo 掃描可能耗時數分鐘，建議背景執行

---

## 適合誰

- 想用某個開源工具前，先確認安全性的人
- 接手別人專案、不確定有沒有踩到漏洞或洩漏憑證的人
- 想定期檢查自己專案依賴套件健康度的人

## 不適合誰

如果你需要：

- Auto Fix
- 自動回報漏洞 / 自動開 issue
- 持續監控（CI/CD 整合）

這個工具不是做這件事的——它是一次性的快速健檢。

---

## FAQ

**Q: 跟 GitHub 內建的 Dependabot / CodeQL 有什麼差？**
A: Dependabot/CodeQL 需要該 repo 自己開啟，而且只對 repo owner 有用。這個工具是**你自己**對任意專案（包括別人的開源 repo）跑掃描，不需要對方設定什麼。

**Q: 掃描會修改程式碼或對外回報漏洞嗎？**
A: **不會。** 這是唯讀分析工具，只輸出報告。是否要修復、是否要聯絡上游維護者，都由你自己決定。

**Q: Scorecard 顯示「無資料」是什麼意思？**
A: 代表這個 GitHub repo 還沒被 OpenSSF Scorecard 收錄（常見於較小型或非熱門的 repo），不代表這個 repo 有問題，只是缺少這項額外資訊。

**Q: 我的專案沒有 `package-lock.json`，osv-scanner 還能用嗎？**
A: osv-scanner 是比對鎖定檔，沒有鎖定檔就無法檢查依賴漏洞，這項會在報告中註明「未執行」，但 semgrep / gitleaks 仍會正常執行。

---

## License

[MIT License](LICENSE) — 可改、可用、可商用，保留 LICENSE 檔即可。

---

## 致謝

這個工具源自一個很單純的念頭：**裝一個開源套件之前，花一分鐘看一下它到底安不安全，總比裝完才發現有問題好。**

osv-scanner、semgrep、gitleaks、OpenSSF Scorecard 都是社群已經做得很好的工具，這個 skill 只是把它們串成一個「掃一下就好」的流程，剩下的判斷交還給人。

如果這個工具幫到你，歡迎來信或私訊交流：[Facebook](https://www.facebook.com/TSY.Microglow)

---

**⭐ 覺得有用的話，歡迎 Star / Fork。**
