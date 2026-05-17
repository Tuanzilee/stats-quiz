# CLAUDE.md

## Repo 用途

心理與教育統計學(心統)轉學考刷題 PWA。台大 / 台綜成大 / 東吳 / 政大共 326 題,單頁前端 + localStorage。

線上部署:**https://tuanzilee.github.io/stats-quiz/**(GitHub Pages,push 後 1-2 分鐘自動部署)

姊妹專案:`~/Desktop/psy/`(普心題庫)。**架構不同**,別套錯。

## 檔案結構

```
stats/
├── index.html              ← 主程式(~3,400 行)。源 = 部署檔,直接改
├── 題庫.json               ← 326 題,runtime fetch 載入(不 inline)
├── sw.js                   ← Service Worker,PWA 離線快取
├── manifest.json           ← PWA 設定
├── README.md
├── 心統刷題系統_SOP.md     ← 使用者 SOP(操作 / 部署 / 刷新)
└── CLAUDE.md               ← 本檔
```

### 跟 psy 的關鍵差異(2026-05-16 工程化後)

| | psy | stats |
|---|---|---|
| Build pipeline | ✅ `build_html.py` + template | ❌ **無**(`index.html` 直接是源) |
| GitHub Actions | ✅ `.github/workflows/build.yml` | ❌ **無** |
| 題庫架構 | 21 個 school-year JSON,build inline 進 index.html | 單檔 `題庫.json`,runtime fetch |
| PWA | ❌ | ✅ `sw.js` + `manifest.json` |

→ stats **絕對不要**為了「跟 psy 一致」去加 build / Actions / template。輕量化是設計選擇。

## 勘誤工作流

```
1. claude.ai 出 prompt:「NTU-108-03 答案應該是 B 不是 C,因為 ...」
2. Claude Code 在 題庫.json 找到 id="NTU-108-03",改對應欄位(answer / solution_steps / ai_hint ...)
3. bump sw.js 的 CACHE 版本號(見下方規則)
4. git add 題庫.json sw.js
5. git commit -m "修正 NTU-108-03 答案"
6. git push origin main
7. GitHub Pages 1-2 分鐘自動部署
```

### 題庫.json schema(可改欄位)

每題 object:
```
{
  "id":           "NTU-108-01",          ← 不要改
  "source":       "NTU",                  ← 不要改
  "year":         108,                    ← 不要改
  "type":         "計算 / 選擇 / 是非 / ...",
  "question":     "題目文字",
  "answer":       "B",                    ← 常勘誤這個
  "options":      ["...", "...", ...],    ← 可能不存在(非選擇題)
  "solution_steps": ["步驟1", "步驟2", ...],
  "key_concepts": "...",
  "ai_hint":      "...",
  "concepts":     ["..."],
  "difficulty":   "easy / medium / hard",
  "formula_used": "...",
  "primary_concept": "...",
  "group_id":     "NCKU-111-G-B",          ← 題組 ID;null 表示獨立題
  "shared_stem":  "情境共用題幹(支援 markdown table)" ← 群組共用情境;null 表示無
}
```

→ **改 `index.html` 不會改變題目資料**;題目資料只在 `題庫.json`。

### Schema convention(鎖定)

- **是非題 storage 統一 α 派**(2026-05-17 起):
  - `type` = `"是非題"`
  - `options` = `["A. 正確", "B. 錯誤或無法判斷"]`
  - `answer` = `"A"` or `"B"`
  - 不要再用 `options=[]` 或 `answer=""`(舊 NTU-113-TF 派已轉換)
- **題組**(2026-05-17 起):
  - `group_id` = `"{SOURCE}-{YEAR}-G-{字母}"` 例 `"NCKU-111-G-B"`
  - `shared_stem` 為整組共用情境(markdown 格式,支援 table)
  - 同一 `group_id` 的題目 `shared_stem` 內容必須完全一致(冗餘儲存,要勘誤情境需 grep 同組全改)

## PWA 快取規則(2026-05-16 起)

`sw.js` 第 1 行的 `CACHE` 常數控制版本:

```js
const CACHE = 'stats-quiz-v2026-05-16-1';
```

**任何 `index.html` 或 `題庫.json` 內容變動,都必須 bump 這個版本號**,否則:
- 使用者本機已快取 → 永遠看不到新版
- iPad PWA 加在主畫面的 → 同樣卡在舊版

格式 `stats-quiz-vYYYY-MM-DD-N`,同日二次發 N+1。範例:`stats-quiz-v2026-05-16-2`。

## 保留的子系統

| 子系統 | 主要 function | localStorage key |
|---|---|---|
| 答題紀錄(對/錯/不確定標記) | `loadRecords` / `saveRecords` / `markQ` | `quiz_records_v1` |
| 模考系統(分校年抽題、計時、自評) | `startMockExam` / `submitExam` / `renderMockReport` | `mock_history` |
| **AI hint(練習時跟 AI 討論觀念,核心功能)** | `openStepAIChat`(逐步引導) / `openWrongAIChat`(錯題討論) / `callCalcAI` / `promptApiKey` / `saveApiKey` | `anthropic_api_key` |
| 手寫板(計算過程記下) | `hwInit` / `hwSave` / `hwPointerDown` 等 | (內嵌在答題紀錄裡) |
| 公式速查 panel(右側滑入,inline 在 index.html) | `openFormulaPanel` / `switchFormulaTab` | — |

## 已砍的子系統(2026-05-16)

| 子系統 | 為什麼砍 | 對應 localStorage |
|---|---|---|
| in-app 勘誤管理(編輯題目、產勘誤清單) | 勘誤該直接改源頭 `題庫.json`,不該在 client 端臨時改 | `stat_corrections_v1`(已在 init 一次性 `removeItem`) |
| 個人筆記(每題本地註記) | 跟 in-app 勘誤共用編輯面板,一起砍 | `stat_notes_v1`(已在 init 一次性 `removeItem`) |
| header「⬇ 匯出題庫」按鈕 | 沒勘誤系統後等於原檔下載,意義不大 | — |
| `toolbox.html`(獨立公式工具書頁) | 孤兒檔,index.html 從未連結;內部 side panel(`openFormulaPanel`)已涵蓋同份資料 | — |

→ 未來如果想加類似功能,**先想清楚目的**:勘誤是工程流程、不是 app feature。

## 與 Claude 的協作風格

- **直接動手,做完再報**。不需要逐步請示;只在「會動到大量檔案 / 不可逆操作 / 需求模糊」時才停下確認
- **改檔優先用 Edit(str_replace),少用 Write 整檔覆蓋**
- **不要重複問同樣的事**。已經寫進這份 CLAUDE.md 的事,直接照辦
- 改 `index.html` 後**記得 bump `sw.js` CACHE 版本**,否則使用者拿不到新版
- 改完後本機開 `python3 -m http.server 8080` 開瀏覽器點一下確認沒事(寫 HTML 不會跑 type check,自己測比較安全)

## 已知狀態(2026-05-16)

### 題庫分布(共 326 題)

| 學校 | 題數 |
|---|---|
| NTU | 46 |
| NCKU | 128 |
| SCU | 56 |
| NCCU | 96 |

題型:計算、選擇、是非、申論。

### 題目 ID 規則

`學校-年份-題號`,例 `NTU-108-01`、`NCKU-114-23`、`SCU-109-05`、`NCCU-111-12`

### 勘誤紀錄

- **2026-05-17 NCKU-107-08**:answer B → A(原題幹「配對設計」與選項 df 不符,實為獨立樣本 t test;重寫 question / options / solution_steps / ai_hint / concepts / formula_used / primary_concept / key_concepts)
- **2026-05-17 NCKU-111**:登記 4 個題組
  - G-A(01-06):COVID 疫苗施打資料
  - G-B(07-16):170 份問卷分數
  - G-C(17-25):Moderna 第一/二劑頭暈反應
  - G-D(29-30):ANOVA 結果表
- **2026-05-17 NCKU-111-29**:ANOVA 數據表移到 `shared_stem`(question 內不再重複)
- **2026-05-17 NTU-113-TF-01..06, 10**:統一是非題 α 派格式(`options=['A. 正確','B. 錯誤或無法判斷']`;answer 從 solution_steps 推得 B/A/B/A/B/A/A)

### 待勘誤候選(需原卷對照)

| ID | 命中啟發式 | 備註 |
|---|---|---|
| NCKU-107-20 | solution_steps 寫「答案推測:B(待確認)」 | 需原卷 verify |
| NCKU-107-21 | solution_steps 寫「答案推測:D(待確認)」 | 需原卷 verify |
| NCKU-107-24 | solution_steps 寫「答案推測:B(待確認)」 | 需原卷 verify |
| NCKU-108-19 | solution_steps「答:D(待確認)」+ hint「需確認答案」(巢狀設計 df 不確定) | 需原卷 verify |
| NCKU-113-13 | ai_hint hedging:「若題目答案為 D,可能涉及特定計算細節」 | 軟候選 |
| NCKU-113-19 | ai_hint hedging:「題目脈絡可能將 E 視為需要確認的選項」 | 軟候選 |

### Phase 2 待辦

- **抽題綁定**:隨機抽到題組中一題,是否強制整組連著出?
- **模考題組行為**:模考設定要不要加「題組綁在一起出」開關?
- **錯題本連帶情境**:目前 `buildContext` 已 inline 顯示前題鏈 + shared_stem,但要不要連帶整組?
- 待勘誤候選(上表)拿到原卷時 verify 一輪

### Known issues

- **AI bottom sheet 在 iPad 軟鍵盤彈出時可能被擠壓,input bar 可能被遮住**。
  尚未做 `visualViewport.addEventListener('resize')` 補償。
  Workaround:拖 drag handle 把 sheet 往上拉到 90vh,input bar 會浮到鍵盤上方。
  等實際用起來有痛點再處理。
