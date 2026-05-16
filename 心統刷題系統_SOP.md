# 心統刷題系統 SOP

> 線上版:**https://tuanzilee.github.io/stats-quiz/**
> 工程化 Repo:`~/Desktop/stats/`
> 工程化說明(給 Claude 看):見同目錄 `CLAUDE.md`

---

## 一、日常使用

### Mac 本機刷題

```bash
cd ~/Desktop/stats
python3 -m http.server 8080
```

打開瀏覽器:`http://localhost:8080`

> ⚠ 不能直接點兩下 `index.html` 開啟(`fetch('./題庫.json')` 在 `file://` 會失敗)

### iPad / 手機刷題

直接打開主畫面 App,或 Safari 進:
```
https://tuanzilee.github.io/stats-quiz
```

### 加入 iPad 主畫面(PWA)

1. Safari 打開上面的網址
2. 點底部**分享**按鈕(方框加箭頭)
3. 選**加入主畫面** → **新增**

---

## 二、勘誤 / 補題流程(2026-05-16 起改用工程化流程)

### 流程總覽

```
看到題目錯 / 想補題
   ↓
打開 claude.ai (網頁版),貼題目截圖 + 描述「哪題哪欄錯」
   ↓
讓 Claude 生出 prompt(會包含:題目 id、要改的欄位、新值、依據)
   ↓
把 prompt 丟給本機的 Claude Code(在 ~/Desktop/stats 目錄)
   ↓
Claude Code 改 題庫.json + bump sw.js + commit + push
   ↓
等 1-2 分鐘 GitHub Pages 部署
   ↓
強制重整(下方第三節)→ 拿到新版
```

### 為什麼改用這流程

- **直接改源頭**:題目資料只在 `題庫.json`,改它就好,不必經過 app 內的勘誤面板再匯出再貼進來
- **可追蹤**:每次勘誤都是一個 git commit,有歷史
- **PWA 一定同步**:sw.js 版本 bump 後,所有裝置強制抓新版

### 對 claude.ai 出 prompt 的範例

> 截圖一張 NTU 108 的考卷,告訴 Claude:「NTU-108-03 的標準答案是 B,但題庫.json 寫 C,幫我生 prompt 給 Claude Code 改。」

它會回類似:

```
修正 NTU-108-03 答案

在 ~/Desktop/stats/題庫.json:
  找到 id="NTU-108-03"
  把 answer 從 "C" 改成 "B"
  在 answer_note 補上「來源:標準解答 PDF p.5」

接著 bump sw.js 的 CACHE 版本(stats-quiz-v2026-MM-DD-N)。
最後 commit「修正 NTU-108-03 答案」並 push。
```

把這段 paste 進 `cd ~/Desktop/stats` 後啟動的 Claude Code,等它做完。

---

## 三、題庫更新後如何刷新

> 已經 bump 過 sw.js 的情況下,還是要強制重整一次才會立刻生效。

### Mac
```
Cmd + Shift + R(強制重新整理)
```

### iPad PWA
1. 關閉主畫面 App,**完全結束**(背景滑掉),重新打開
2. 或 Safari 清除快取後重新整理

---

## 四、答題紀錄說明

- 答題紀錄、模考歷史、AI 對話用的 API key,都存在**該裝置的 localStorage**
- 清除瀏覽器快取 / 重新安裝 PWA 會消失
- Mac 跟 iPad 的紀錄**不互通**(沒做帳號同步)
- 建議主要在一台裝置上刷題

---

## 五、題目 ID 規則

| 學校 | 前綴 | 範例 |
|---|---|---|
| 台大 | NTU | NTU-108-01 |
| 台綜成大 | NCKU | NCKU-107-03 |
| 東吳 | SCU | SCU-109-05 |
| 政大 | NCCU | NCCU-111-12 |

格式:`學校-年份-題號`

---

## 六、AI hint 功能(刷題時)

主程式內保留兩個 AI 對話入口,需先填 Anthropic API Key:

- **💬 AI 逐步引導**(答題前可點):分階段提示思路,不直接給答案
- **錯題討論**(答錯後出現):跟 AI 討論為什麼錯、怎麼想

API key 第一次點到時會跳框輸入,存在本機 localStorage(`anthropic_api_key`),不會外傳。換裝置要重填。

---

## 七、常見問題

**Q:打開網頁顯示「載入失敗」**
A:確認用 `python3 -m http.server 8080` 啟動,不能直接點兩下 `index.html`

**Q:GitHub Pages 網址打不開**
A:等多幾分鐘,或到 repo Settings → Pages 確認部署狀態

**Q:題庫更新後沒變化**
A:八成是 PWA 快取問題。檢查 commit 有沒有同時 bump `sw.js` 的 CACHE 版本;有的話強制重整(第三節)

**Q:Claude Code 改完 push 了,但網頁還是舊版**
A:同上,先強制重整;iPad 上把 PWA App 完全結束再開
