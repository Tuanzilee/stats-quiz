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

- **個人筆記**(2026-05-19 起):
  - localStorage key `user_notes_v1`(避開已清的 `stat_notes_v1`)
  - schema:`{ qid: [{id, ts, text}, ...] }`
  - **手動精華**,不自動存 AI 對話 raw,也不自動存練習填答 textarea raw
  - 顯示位置:練習(quiz)+ 模考檢討(review)題目下方;**模考作答中、browse 列表、考點頁 AI review** 不顯示
  - `updateNote` 會把 `ts` 更新成最後編輯時間(影響排序)

### 抽題排序(2026-05-19 起)

`startQuiz('all')` 和 `startConceptQuiz` 採分層抽題:
- tier 0:沒做過(`!records[qid]`)
- tier 1:不懂(`status='wrong'`)
- tier 2:不確定(`status='unsure'`)
- tier 3:懂了(`status='correct'`)

低 tier 先出,同 tier 內 random。`startQuiz('wrong')` / `startQuiz('unsure')` / `startMockExam` 維持純 random(避免破壞「複習已標記」「模考公平試」的精神)。

實作:`sortByPriority(pool)` helper(index.html ~line 1544)。

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
| 答題紀錄(對/錯/不確定標記;練習 + 考點頁刷題會依此分層抽題) | `loadRecords` / `saveRecords` / `markQ` / `sortByPriority` | `quiz_records_v1` |
| 模考系統(分校年抽題、計時、自評) | `startMockExam` / `submitExam` / `renderMockReport` | `mock_history` |
| **AI hint(練習時跟 AI 討論觀念,核心功能;overloaded 自動 retry 3 次 2s/4s/8s,失敗顯示中文 friendly error)** | `openStepAIChat`(逐步引導) / `openWrongAIChat`(錯題討論) / `callCalcAI` / `callAnthropicWithRetry` / `friendlyAIError` / `promptApiKey` / `saveApiKey` | `anthropic_api_key` |
| 手寫板(模考時 Apple Pencil 計算) | `hwInit` / `hwSave` / `hwPointerDown` 等 | 僅 in-memory(`mockNotes[qid]`,跟申論 textarea 共用 key,session 結束消失) |
| 公式速查 panel(右側滑入,inline 在 index.html;9 個 tab: 描述統計 / 機率分配 / 信賴區間 / 假設檢定 / 相關迴歸 / ANOVA / 效果量 / 快速查閱 / **申論題框架**) | `openFormulaPanel` / `switchFormulaTab` | — |
| 個人筆記(手動精華,per 題多筆) | `loadNotes` / `saveNotes` / `addNote` / `updateNote` / `deleteNote` / `getNotes` / `renderNotesBlock` | `user_notes_v1` |

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

### 題庫分布(共 347 題,2026-06-09 更新)

| 學校 (source) | 題數 |
|---|---|
| 台大 (NTU) | 47 |
| 成大 (NCKU) | 143 |
| 東吳 (SCU) | 57 |
| 政大 (NCCU) | 100 |

> source 欄位用中文校名(`台大`/`成大`/`東吳`/`政大`);index.html `sourceMap` 把 pill 代碼(NTU/NCKU/SCU/NCCU)對應到中文。成大 2026-06-09 起統一為「成大」(原「台綜（成大）」併入)。

題型:單選、計算、申論、是非、填充、複選、配合、證明。

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
- **2026-05-19 NCCU-111-03**:answer C → A(變異數計算 548/7≈78.3 最接近 (A)78,原 solution_steps 已寫出此結論但又自我懷疑「但答案說 C(82)」「需確認」;重寫 solution_steps + ai_hint 移除矛盾。Tier 1 待勘誤候選,前次 #11 偵察 keyword 範圍未含 NCCU)
- **2026-05-19 SCU-113-02**:answer B → C(原卷題意混亂,「以下何者錯誤」實有 A/B/D 三個錯選項,C 為唯一正確;照老師答案改 C,question 維持原卷,ai_hint 加警語說明題意應為「何者正確」;formula_used 補入平均/中位數/眾數三公式)
- **2026-05-19 NCKU-107-24**:answer 從空字串 → B(原 solution_steps 已分析出 B 正確,只是末行「答案推測:B(待確認)」未斷言;老師確認後落定;ai_hint 原本已無 hedging,不動)
- **2026-05-19 NCKU-107-21**:answer 從空字串 → D(原 solution_steps 已分析出 D 正確,只是末行「答案推測:D(待確認)」未斷言;老師確認後落定;ai_hint 原本已無 hedging,不動)
- **2026-05-19 NCKU-107-20**:answer 從空字串 → B(交互作用直接表述,D 為 over-interpretation;原 solution_steps 已分析出 B 正確,只是末行「答案推測:B(待確認)」未斷言;老師確認後落定;ai_hint 原本已無 hedging,不動)
- **2026-05-19 SCU-109-03**:answer "34" → "4"(老師確認 (3) 考試成績寬鬆視為比率量尺,只有 (4) 態度量表 = Likert = ordinal 為錯;solution_steps 從「(3)(4) 皆錯」雙答案改成「(4) 為唯一錯」單答案;ai_hint 補 Likert 爭議說明;key_concepts 對齊新立場)
- **2026-05-19 NCKU-108-19**:answer D → A(**邏輯全反**;老師確認 (A) 為錯 — 教師為固定因子(fixed factor)非隨機因子,因為補習班只有 4 位老師,這就是全部師資,沒有「從更大母群抽樣」設計;原 solution_steps 把 A 標正確、D 標錯,邏輯全反;solution_steps / ai_hint / key_concepts / formula_used 全重寫;清掉「(待確認)」hedging)
- **2026-05-19 NCKU-112-01**:answer D → B(題幹「來自哪些學校」phrasing 模糊,可解讀為學校類別(nominal,用眾數→C)或聚合成各校人數(數值,可算母數→B / 用直方圖→D);老師詳解採「各校人數=數值」解讀;solution_steps 重寫為中立 4 選項分析 + 結論;ai_hint 寫雙解讀警語 + 考試策略;key_concepts 對齊新立場;不在原 Tier 1 待勘誤候選表內,老師另外指出)
- **2026-05-20 NCKU-113-13**:answer D → C(**翻面**;老師詳解 2×3 Two-way ANOVA(a=2, b=3, r=10, N=60, df_error=54);t 檢定限 2 組,不適 multi-level → (C) 為唯一錯;原 solution_steps 12 步 confused reasoning 最後答 D,跟老師相反;solution_steps / ai_hint / key_concepts 全重寫)
- **2026-05-20 NCKU-113-19**:answer E → D(**翻面**;老師詳解:變異數不同質**不必然**轉無母數,若母體常態仍可用 Welch's t-test adjustment;D 字面「可以」暗示「變異數不同質 → 改用無母數」conditional 邏輯錯;原 solution_steps 把 D 標對、E 答(自我懷疑「E 是正確的, 不是錯的」);solution_steps / ai_hint / key_concepts 全重寫;✅ Phase 1 #11 待勘誤候選表全部清零)
- **2026-05-20 NCCU-108-21**:answer D → A(**翻面**;老師詳解 W = X+Y where X,Y ~ Uniform(0,100) → Triangle(0,200), E(W)=100;(B)(C)(D) 皆錯,(A) 寬鬆視為對(Triangle 對稱鐘形 ≈ 常態,同 SCU-109-03 取分邏輯);原 solution_steps 13 步 confused reasoning 最後選 D,跟老師相反;solution_steps / ai_hint / key_concepts / formula_used 全重寫;**不在 Tier 1 候選表**,老師另指)
- **2026-05-21 NCCU-112-05**:answer "AD" → "A"(原為單選題卻填雙答 "AD",schema 違規;老師確認 (A) SE=σ/√n vs σ 是硬錯誤,(D) 「CLT 並非特性」字面負面 meta-claim 採寬鬆解讀視為是特性;solution_steps / ai_hint / key_concepts / formula_used 對齊;concepts 加「不偏性」)
- **2026-06-01 primary_concept 一致化 44 → 14 + 練習/弱點優先序加權**:
  - **問題**:全題庫累積 44 distinct `primary_concept` (含 28 個 singleton 多為長英文版同義雙寫,例「中央極限定理」vs「中央極限定理 (Central Limit Theorem)」)。弱點分析 / 按 pc 抽題 / 統計報表 在同義雙寫下會被切碎。
  - **動作 (Phase 1: 資料)**:依 14 類 family 改名 (82 題的 `primary_concept` 字串改),收齊到:
    - 描述統計 (46)、假設檢定 (27)、ANOVA (72)、機率分配 (27)、中央極限定理與抽樣分配 (15)、迴歸分析 (47)、t檢定 (20)、卡方與類別資料 (26,含 epidemiology Risk/Odds 系列)、相關 (16)、估計與信賴區間 (18)、條件機率與貝氏 (7)、無母數 (10)、抽樣與研究設計 (13)、效果量 (2)
    - 重命名: 「中央極限定理」→「中央極限定理與抽樣分配」、「卡方檢定」→「卡方與類別資料」、「抽樣與設計」→「抽樣與研究設計」、「資料視覺化」→「描述統計」、「樣本數估計」→「估計與信賴區間」
  - **動作 (Phase 2: 引擎)**:
    - 新增 top-level `CONCEPT_PRIORITY` 物件 (14 類 → 1~14 排名) + `conceptRank(q)` / `conceptRankOf(pc)` helpers
    - `sortByPriority(pool)` 增第一層 sort:`conceptRank → tier → random` (描述統計先出,效果量最後;同類內維持既有 tier 邏輯)
    - 弱點分析 (home page) 改用加權 score = 錯誤率 × (15 − rank) 排序;顯示用 `pct` 維持原語意 (s.wrong/s.total)
    - 模考 `startMockExam` **完全不動** (維持隨機 / 對應某校,沒呼叫 sortByPriority)
  - sw.js `v2026-06-01-11` → `v2026-06-01-12`
- **2026-06-01 NCCU-109-19~23 重整 + solution_steps 圖片渲染**(**勘誤帳 +1**):
  - **問題**:NCCU-109-19to23 是 5 題合 1 entry (前人壓縮版),題幹用列聯表文字描述,缺圖。
  - **動作**:
    - 拆 5 筆獨立 entry `NCCU-109-19..23`,套 `group_id: NCCU-109-G-A` + 共用 shared_stem (含吸煙×肺癌列聯表圖 `NCCU-109-19-table.png`)
    - 每筆獨立 question / options (4 個單選) / answer (A/D/A/A/C) / solution_steps / primary_concept (Risk / Risk ratio / Odds / Odds ratio / Risk vs Odds 抽樣設計)
    - 刪除舊 NCCU-109-19to23 entry
  - **連帶引擎修**:`solution_steps` 渲染路徑沒 markdown→img (line 2061-2065 練習頁 + line 2909-2912 檢討頁,皆 `<span>${s}</span>` 直丟)。新增 helper `renderInlineImages(s)` (top-level scope),兩個渲染點各包一層,把 inner `<span>` 改 `<div class="sol-step">` 支援 block-level `<figure>`。NTU-108-06 兩張 χ² 示意圖 (在 solution_steps 內) 同步生效。
  - assets 新增 `NCCU-109-19-table.png`。
  - 累計勘誤 26 → 27
  - sw.js `v2026-06-01-7` → `v2026-06-01-8`
- **2026-06-01 SCU-112 整批取代 15 題**(**勘誤帳 +1**):
  - **問題**:既有 `SCU-112-01..15` 是 AI 推導的精簡版 (compressed question + AI 推 answer/solution),非原卷忠實版。
  - **動作**:依原卷 spec 整批取代 15 筆 (填充 10 + 計算 4 + 申論 1),每筆完整 entry rewrite:大題標籤 (`【填充 N】` / `【計算 N】` / `【申論】`)、原卷情境敘述、圖片 markdown 引用 (assets/figures/...png × 6 張)、新版 primary_concept/concepts/solution_steps/key_concepts/ai_hint/formula_used。
  - **答案策略**:
    - `-01~-10` 已填 answer (填充題單純機械答案)
    - `-11~-15` `answer = ''` (計算/申論題待 user 親算驗證),`solution_steps` 已附完整推導
  - **6 張圖**:`SCU-112-10-stats-table` / `-12-exercise-hours` / `-13-cake-table` / `-14-regression-output` / `-15-anova-data` / `-15-ftable` (F 表單獨,無 dedup 需要)
  - **schema**:全 15 題單題 (`group_id` / `shared_stem` 都 del key),對齊原卷無題組結構。
  - **舊 mock_history / quiz_records 影響**:既有 SCU-112-01..15 qid 仍對得到題,但題目語意有更新 (尤其 -10 改為線性轉換表+偏態,計算題 -11~-15 改為含圖版),舊紀錄視為孤兒可接受。
  - sw.js `v2026-06-01-2` → `v2026-06-01-3`
  - 累計勘誤 25 → 26
- **2026-06-01 NCCU-111-31 對答案** (C → B,**勘誤帳 +1**):
  - answer `'C'` → `'B'`;solution 推導 χ² = 0.5 + 0.5 + 1.125 + 0.25 + 1.0 = 3.375,最接近 3.4 (B)。原 answer 標 (C) 3.6 與推導相違。
  - solution_steps 移除末條自我懷疑「但若答案是 (C) 3.6，需重算...」(原 7 條 → 6 條,結尾改以「最靠近 3.4，答:(B) 3.4」收。
  - sw.js `v2026-06-01-1` → `v2026-06-01-2`
  - 累計勘誤 24 → 25
- **2026-06-01 primary_concept 分類修正 22 題**(批次分類修正,**勘誤帳 +1**):
  - **問題**:全題庫 family-level audit 後發現 22 題 `primary_concept` 與 `concepts` / 題目本質不一致 (例:t 檢定題標成「假設檢定」過於籠統、卡方題標成「描述統計」、機率分配題沒對齊「機率分配」family)。會干擾按 primary_concept 出題的弱點分析。
  - **動作**:批次改 22 題的 `primary_concept` (僅此一欄,其他完全不動;git diff 確認 surgical)。
  - 主題分布:相關 ×4、機率分配 ×6、卡方檢定 ×2、描述統計 ×2、估計與信賴區間 ×2、t檢定 / 相依樣本t檢定 / 假設檢定 / 測量 / ANOVA / 研究設計 各 1。
  - sw.js `v2026-05-31-3` → `v2026-06-01-1`
  - 累計勘誤 23 → 24
- **2026-05-31 SCU-111 大整理(1 筆勘誤 + schema 一致化)**:
  - **問題 (SCU-111-10 補錄漏題,**勘誤帳 +1**)**:原卷大題 9 為 2(A)×2(B) 二因子受試者間 ANOVA 計算題 (含原始資料表 + ANOVA 摘要表,空格 40~50 共 11 個小題),舊版題庫漏收。本次新增 `SCU-111-10` entry,含兩張表圖 (`SCU-111-10-raw-data.png` + `SCU-111-10-anova-table.png`) + 完整 9 步 solution (df_A/B/AB = 1、MS_E ≈ 10935.9、F_A ≈ 26.4、F_B ≈ 2.46、F_AB = 4.51 反推 MS_AB ≈ 49320.9)。`answer = ''` 待 user 親算後 batch 修。
  - **schema 一致化** (**不入勘誤帳**):
    - SCU-111-01~06 套 `group_id: SCU-111-G-A` + 共用 `shared_stem` (24 個代號清單 a-x,代號對應內效度 / 外效度 / 各類抽樣 / 各類相關 / 各類圖表 / 假設檢定結果 / Type I/II error / power)
    - SCU-111-02 還原原卷完整 4 段抽樣情境 (越戰徵兵 366 球 / 高中男女比例 3:2 / 12 行政區隨機抽 4 + 街道普查 / 5000 玩偶 k=100 系統抽樣);舊版被簡化成括號內小字。
    - SCU-111-07 / 08 各自加 shared_stem (對應 ANOVA 表圖)
    - SCU-111-09 加 shared_stem (2×3 互動效果圖 + 顯著規則 ≥20)
  - sw.js `v2026-05-31-2` → `v2026-05-31-3`
  - 累計勘誤 22 → 23
- **2026-05-31 SCU-109 大整理(2 筆勘誤 + schema 一致化)**:
  - **問題 1 (SCU-109-20 真錯誤,**勘誤帳 +1**)**:既有 question 文字描述「男性效果斜率較大」,answer 標 `'4'` (男性效果較強)。對照原卷互動圖,實際是**女性線斜率較陡** (3.85 → 4.8,Δ ≈ +0.95) 而男性線平緩 (3.2 → 3.4,Δ ≈ +0.2) → 正確 answer 應為 `'3'` (女性效果較強)。重寫 question / options / answer / solution_steps / ai_hint / key_concepts 全部對齊圖事實 + 加 `shared_stem` 引圖。
  - **問題 2 (SCU-109-04 文字錯誤,**勘誤帳 +1**)**:既有 question 內聯「圖一 (右尾長)」「圖二 (左偏直方圖)」假描述 (沒有圖)。原卷實際有兩張圖,圖一是右偏 curve、圖二是雙峰分布 → 兩個都不算「典型」正偏態。改寫 question 移除假描述、加 `shared_stem` 引兩張圖、重寫 solution_steps / ai_hint / key_concepts 對齊圖事實。
  - **schema 一致化** (**不入勘誤帳**):
    - SCU-109-02:還原原卷完整 question + options 文字 (補回「根據上述敘述」「65 歲以上之老人」等被簡化的字)
    - 套 3 組 group_id + shared_stem:
      - `SCU-109-G-A` (05~08 員工壓力指數)
      - `SCU-109-G-B` (13~15 戒菸計畫,含前後測資料圖)
      - `SCU-109-G-C` (18~19 ANOVA F 值,含摘要表圖)
    - SCU-109-16 加散點圖
    - SCU-109-13 / 18 question 移除已搬到 stem 的內聯情境
  - sw.js `v2026-05-31-1` → `v2026-05-31-2`
  - 累計勘誤 20 → 22
- **2026-05-31 NCKU-109 重整(03/04 還原 + 05a..d → 05..08,單一勘誤 entry)**:
  - **問題**:NCKU-109-03/04 原 question 過度簡化(把「滿意度資料表」「列聯表 + 卡方臨界值」直接縮成括弧內小字),失去原卷情境與表格;NCKU-109-05a..d 用「字母後綴」舊慣例(對齊 NCKU-114 重整方向應改為連號 + group_id)。
  - **動作**:NCKU-109-03/04 原地改 question + 加 `shared_stem`(含【第 3 / 4 題】標籤 + 表格圖);NCKU-109-05a..d 重編為 NCKU-109-05..08,套 `group_id: NCKU-109-G-A` + 共用 `shared_stem`(含【第 5 題】標籤 + 描述統計表圖);type/solution_steps/answer/concepts 等其他欄位完全沿用舊 a/b/c/d。
  - **孤兒紀錄**:既有 NCKU-109-05a..d 答題紀錄變孤兒(qid 對到新題但語意對齊,內容語意一致)。
- **2026-05-31 NCKU-114-02 對答案(單一勘誤 entry)**:
  - **問題**:answer 寫「不變或變小 (...)」,語意游移無法當標準答案。
  - **動作**:answer 直接斷言「**變小**」+ 完整推導(移除值 8 > 中位數 7.8 → 排序第 52 筆或更後 → 新中位數 = (原第 50 筆 + 7.8) / 2 ≤ 7.8;「不變」只在多人並列時才成立,實務視為變小)。除 answer 外其他欄位完全不動。
- **2026-05-31 NCKU-114-G-E shared_stem 換 ANOVA 結果表圖**(schema 一致化,**不入勘誤帳**):
  - 17~20 共用 stem 原為 markdown table 寫死 df 表;改為 `![ANOVA 結果表](assets/figures/NCKU-114-q5-anova-table.png)` 引圖(對齊 NCKU-111-29/30 + NTU-108-Q6 的圖片化方向)。
- **2026-05-31 大題標籤 prepend**(schema 一致化,**不入勘誤帳**):
  - NCKU-114-G-A/B/C/D/E 開頭加「【一】~【五】」、NTU-108-G-A/B/C/D 加「【第 3 / 4 / 5 / 6 題】」、NCKU-109-G-A 加「【第 5 題】」(後者由 NCKU-109 重整一起做)
  - 統一視覺辨識,讓 user 在練習頁立刻看到該大題編號。
- **2026-05-29 NCKU-111 補圖(26 + 27 + 28 + 29 + 30,單一勘誤 entry)**:
  - **問題**:NCKU-111-26 ~ 30 共 5 題的圖表 / 直方圖 / ANOVA 表 / 迴歸表都用文字壓縮塞進 question(或舊 markdown table 版),失真且不直觀;前次 NTU-108 重整(e8f9e50)已開通 `renderSharedStemMarkdown` 圖片渲染,本次補套用。
  - **動作**:每筆只改 `group_id` / `shared_stem` / `question` 三欄(其他 type/options/answer/solution_steps 等完全保留):
    - 26 / 27 / 28:獨立題不歸組,`shared_stem` 加情境敘述 + `![...](assets/figures/...png)` 引圖
    - 29 / 30:沿用既有 `group_id: NCKU-111-G-D`,`shared_stem` 換成圖片版(`NCKU-111-29-anova.png` 共用)
    - 5 題 question 都縮短至 13-20 字(原長文移到 shared_stem 或刪除冗餘)
  - **typo 修正**:user 原 spec 把 29/30 寫成 `G-A`,實際 `G-A` 已被疫苗組 01..06 占用 → 確認後維持 `G-D`
  - sw.js `v2026-05-29-1` → `v2026-05-29-2`
  - 累計勘誤 17 → 18
- **2026-05-29 NTU-108 重整 + 還原原卷題幹 + 開通 shared_stem 圖片**(單一勘誤 entry):
  - **問題**:NTU-108-03a/3b/4a/4b/5a/5b/6a/6b 用「字母後綴」舊慣例,題幹被壓縮失真;原卷 Q3~Q6 每題 (1)(2) 兩小題、共用情境未抽出。NTU-108-02 被簡化,失去原卷情境。NTU-108-06 含直方圖,但 stem 渲染不支援圖片。
  - **動作**:8 筆 a/b 改 `NTU-108-03..10` 連號 + 套 `group_id`(NTU-108-G-A ~ G-D)+ `shared_stem`(對齊 NCKU-114 schema 2026-05-17 起新標準);NTU-108-02 question 還原為原卷「考生補習 + 條件機率」題;NTU-108-01 不動;舊 a/b 8 筆 type/answer/solution_steps 沿用,只改 id/group_id/shared_stem/question。
  - **新增圖片渲染**:`renderSharedStemMarkdown` 支援 markdown `![alt](path)` 語法 → `<figure class="stem-figure"><img>`;新 CSS `.stem-figure` 樣式(白底 + 圓角 padding,適暗色主題);NTU-108-09/10 的 shared_stem 引用 `assets/figures/NTU-108-q6-row1.png` + `row2.png`(repo 內首次採用 figures 目錄)。
  - **孤兒紀錄**:既有 `NTU-108-03a..06b` 在 mock_history / quiz_records 變孤兒(qid 對不到題);量小可接受。
  - sw.js `v2026-05-28-1` → `v2026-05-29-1`
  - 累計勘誤 entry 16 → 17
- **2026-05-27 NCKU-114 重整(20 小題,單一勘誤 entry)**:
  - **問題**:原 NCKU-114-01..05 把 5 大題每題的 4 小題壓縮塞進同一個 `question` 欄位,題幹過度簡化導致 user 看不出老師要問什麼。
  - **動作**:刪除舊 5 entry,新建 20 entry (`NCKU-114-01..20`),按 5 大題分 5 個 `group_id`(`NCKU-114-G-A` ~ `G-E`),`shared_stem` 放各大題情境(描述統計值 / 迴歸模型 / 1000 假設 / 12 位同學設計 / 3 組數據 + df 表)。
  - **還原的關鍵措辭**(原本壓縮失真):
    - 大題一 #2:中位數計算 hint(101 筆 = 第 51 筆;100 筆 = 第 50, 51 筆平均)
    - 大題二 #3, #4:殘差常態分布假設、95% 預測區間寬度約束公式
    - 大題三 #3:**新增 NPV 對稱小題**(舊版漏收;原卷 2/3 是 PPV/NPV 雙線,舊 entry 把 (2)(3) 誤合為「PPV + 增加方式」)
    - 大題四 #1, #2, #4:標準差 √6 / √3、配對與獨立的設計選擇
    - 大題五 #4:df 表 + 「可參考下表」
  - **`answer_confidence`**:18 題 high、2 題 medium(NCKU-114-02 中位數方向、NCKU-114-11 NPV 降低方式 — 後者題幹要求「降低不顯著中無效果比率」邏輯上不對稱於 PPV 提升,solution 已點出 trade-off)
  - **舊 ID 孤兒**:既有 mock_history / quiz_records 中 `NCKU-114-01..05` 的紀錄變孤兒(qid 對到新題但語意改變),user 確認可接受(線上量小)
  - sw.js `v2026-05-27-1` → `v2026-05-27-2`
- **2026-05-25 複選題系統性 audit + batch 勘誤(10 題)**:
  - 全題庫掃出 8 題已正確標 type=複選題(NCKU-111-26/27/28/29/30、NTU-113-MC-22/23/25),schema 已逐選項分析,本批次補:① `solution_steps[0]` prepend「【複選題:正解 = ABD】」顯式標頭;② `ai_hint` prepend「【複選題,每個選項都要獨立判斷,不是擇一】」meta-hint。
  - **NTU-113-MC-11**:type 「單選題」→「計算題」,answer "" → "0.582"(原本無 options 是 Bayes 計算題,被誤標單選空答案;solution 已算出 P(旁聽|考上)≈58.2%)
  - **NTU-113-MC-17**:answer "A" → "C"(**獨立 correctness bug**:原 answer 標 A=34,但 solution_steps 跟 ai_hint 都算出 N = df_error + ab = 24+8 = 32 = 選項C;對齊 solution 改 C;ai_hint 補勘誤註記)
  - 複選題 UI 支援不在本次 scope(stats 走單選 radio,複選題僅資料正確,user 作答顯示「部分對」由 grading 自行判斷)
- **2026-06-08 NCCU-114-36 對答案**(D → C,**勘誤帳 +1**):
  - answer `'D'` → `'C'`;官方答案確認為 (C) 495。組內平方和 SSW = Σ(nⱼ−1)sⱼ²(樣本變異數預設不偏,÷(n−1))= 9×20+9×15+9×20 = 495。
  - solution_steps 整段重寫(5 步),移除原本「但選D=550?」「等等,答案說D」「政大答案選D可能用有偏」整串自我懷疑;改以 495=(C) 為定論,(D)550 列為「把變異數誤當有偏(÷n)」的陷阱說明。
  - key_concepts / ai_hint 連帶對齊(原本主張「政大答案D=550 用有偏版本」與新答案矛盾,一併改正);formula_used 原本已寫 495 不動;primary_concept 維持 ANOVA。
  - sw.js `v2026-06-08-2` → `v2026-06-08-3`
  - 累計勘誤 27 → 28
- **2026-06-08 NCKU-107-25 對答案**(B → C,**勘誤帳 +1**):
  - answer `'B'` → `'C'`;官方答案確認為 (C) 組內MS=12.5。原 answer 標 (B) 但選項 B 寫「df=98」本身就錯(應 n1+n2−2=198),且原 solution_steps 算出 df=198 卻仍選 B、又同時說「C…正確!」自相矛盾。
  - primary_concept `ANOVA` → `t檢定`(兩組獨立樣本比較=獨立樣本 t,與單因子 ANOVA 等價 t²=F;本質是 t 檢定題)。
  - solution_steps 整段重寫(7 步):以 (C) MS_within=pooled variance=2475/198=12.5 為定論,逐一駁斥 (A)25=單組SD²、(B)df 應198非98、(D)組間MS=12.5非25、(E)df 應(1,198)非(2,198)。
  - key_concepts / ai_hint 連帶對齊(原本主張「選B」與新答案矛盾,一併改正);formula_used(df=198 + pooled 公式)仍正確不動。
  - sw.js `v2026-06-08-3` → `v2026-06-08-4`
  - 累計勘誤 28 → 29

- **2026-06-10 NCKU-111-21 對答案**(A → B,**勘誤帳 +1**):
  - answer `'A'` → `'B'`;老師詳解確認。設計為同一批受試者比較第 1 劑 vs 第 2 劑(受試者內 / RM-ANOVA),F 的分母是受試者內誤差 MS,不是受試者間變異 MS。受試者間變異(個別差異)已被 RM-ANOVA 單獨分離出來、不參與 F 比;p=0.02 顯著只能保證「組間(處理) > 受試者內誤差」,無法推論「組間 > 受試者間」。題目敘述比錯對象 → 錯誤敘述 → 答 B。
  - solution_steps 整段重寫(5 步:設計判定 → 變異拆解 → F 比結構 → 比錯對象 → 結論);concepts 補「重複測量ANOVA / 受試者間變異 / 受試者內變異」;key_concepts / ai_hint / formula_used 全對齊新立場(原本主張「F>1→組間>受試者間」邏輯錯,連同改正);primary_concept 維持 ANOVA。
  - sw.js `v2026-06-09-2` → `v2026-06-10-1`
  - 累計勘誤 30 → 31
- **2026-06-09 NCKU-112-10 對答案 + OCR 錯字**(B → C,**勘誤帳 +1**):
  - **錯字**:「暗談」→「晤談」共 3 處(題幹 2 + 選項 A 1,OCR 誤字)。
  - answer `'B'` → `'C'`;官方答案 (C)。原 answer 標 (B)「進行變異數分析」,但 2 水準受試者內本就可用重複測量 ANOVA,(B) 其實成立。
  - solution_steps 整段重寫(6 步):同一批學生期中 vs 期末=受試者內(相依)設計;(A)(B)(D) 皆正確,唯 (C)「須符合變異數同質」是獨立組間假設,相依設計看差異常態(相依t)/球形(RM-ANOVA)、不需變異數同質 → (C) 為錯誤敘述=答案。
  - key_concepts / ai_hint 連帶對齊(原主張「答B」與新答案矛盾);primary_concept 維持 ANOVA(涉重複測量 ANOVA);formula_used(df=n−1)仍對。
  - sw.js `v2026-06-09-1` → `v2026-06-09-2`
  - 累計勘誤 29 → 30
- **2026-06-09 成大 source 標籤統一**(schema 一致化,**不入勘誤帳**):
  - **問題**:成大題目 source 被拆成「台綜（成大）」(123) 與「成大」(20) 兩種標籤;且 index.html 三處過濾器只認「台綜（成大）」→ 那 20 筆「成大」永遠被學校篩選漏掉。
  - **動作**:① `題庫.json` 123 筆 `"source": "台綜（成大）"` → `"成大"`(replace_all);② `index.html` 同步改 3 處過濾 key + 顯示文字:browse pill(`browseFilter('NCKU')` 顯示文字 台綜→成大)、mock-pill `data-val`、filter-chip `data-val`、`renderBrowse()` 的 `sourceMap.NCKU`。統一後成大 = 143 筆全部歸一,學校篩選/模考對應都吃得到。
  - **殘留**:3 處 `ai_hint` 內文仍寫「台綜109年…」(純敘述,非標籤/過濾 key,不影響功能,暫留)。
  - sw.js `v2026-06-08-5` → `v2026-06-09-1`

### 待勘誤候選(需原卷對照)

| ID | 命中啟發式 | 備註 |
|---|---|---|
_(2026-05-20 全部清零 ✅)_

### Phase 2 待辦

- **抽題綁定**:隨機抽到題組中一題,是否強制整組連著出?
- **模考題組行為**:模考設定要不要加「題組綁在一起出」開關?
- **錯題本連帶情境**:目前 `buildContext` 已 inline 顯示前題鏈 + shared_stem,但要不要連帶整組?
- 待勘誤候選(上表)拿到原卷時 verify 一輪
- **填答持久化**(獨立議題,跟筆記系統脫鉤):
  - 練習 mode textarea 內容是否存 localStorage(觸發時機?per-題覆寫?)
  - 模考 `mockNotes` 是否寫進 `mock_history`(逐題 array,連同 mockAnswers)
  - **bug**:手寫板 dataURL 跟申論 textarea 文字共用 `mockNotes[qid]` key(line 3281 vs 2521),同題只能存一個;畫了又打字就會互蓋
  - `mock_history` 連帶完整作答後的 localStorage quota 風險評估
- **個人筆記 Phase B**:home 入口 / 跨題搜尋 / 匯出(.md or .json) / AI 摘要全部筆記

### Known issues

- **AI bottom sheet 在 iPad 軟鍵盤彈出時可能被擠壓,input bar 可能被遮住**。
  尚未做 `visualViewport.addEventListener('resize')` 補償。
  Workaround:拖 drag handle 把 sheet 往上拉到 90vh,input bar 會浮到鍵盤上方。
  等實際用起來有痛點再處理。
