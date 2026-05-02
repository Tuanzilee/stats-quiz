# 心統刷題系統 SOP

## 一、首次部署

### 1. 下載系統
- 從 Claude 下載 `心統刷題系統.zip`
- 解壓縮，得到「刷題系統」資料夾
- 確認資料夾內有這 5 個檔案：
  ```
  index.html
  題庫.json
  manifest.json
  sw.js
  README.md
  ```

### 2. 本機測試（Mac）
```bash
cd ~/Desktop/刷題系統
python3 -m http.server 8080
```
打開瀏覽器輸入 `http://localhost:8080`，確認畫面正常。

### 3. 建立 GitHub Repo
1. 打開 [github.com](https://github.com)，登入
2. 右上角 **+** → **New repository**
3. Repository name 填：`stats-quiz`
4. 選 **Public**
5. 按 **Create repository**

### 4. 上傳檔案
1. 點頁面中的 **uploading an existing file**
2. 把「刷題系統」資料夾內的**全部 5 個檔案**拖進去
   （拖檔案，不是拖資料夾）
3. 按 **Commit changes**

### 5. 開啟 GitHub Pages
1. 點 repo 上方的 **Settings**
2. 左側找 **Pages**
3. Source 選 **Deploy from a branch**
4. Branch 選 **main**，資料夾選 **/ (root)**
5. 按 **Save**

### 6. 等待部署
等 1-2 分鐘，網址為：
```
https://tuanzilee.github.io/stats-quiz
```

### 7. iPad 加到主畫面（PWA）
1. Safari 打開上面的網址
2. 點底部**分享**按鈕（方框加箭頭）
3. 選**加入主畫面**
4. 按**新增**

---

## 二、日常使用

### Mac 本機刷題
```bash
cd ~/Desktop/刷題系統
python3 -m http.server 8080
```
打開 `http://localhost:8080`

### iPad / 手機刷題
直接打開主畫面的 App，或瀏覽器輸入：
```
https://tuanzilee.github.io/stats-quiz
```

---

## 三、勘誤與補題流程

> 不管什麼更新，最後都只需要在 GitHub 換一個 `題庫.json`，`index.html` 幾乎不需要動。

### 情況 A：發現題目有錯誤（勘誤）

**你做的事：** 直接告訴 Claude，格式如下：
```
NTU-108-03 答案錯了，應該是 B 不是 C
SCU-109-15 第3個解題步驟說錯了，正確應該是...
NCCU-111-12 解題步驟漏掉了公式
```

**Claude 做的事：** 修正後回傳新的 `題庫.json`

**你做的事（更新 GitHub）：**
1. 打開 GitHub repo
2. 點 `題庫.json`
3. 右上角點**鉛筆圖示**（Edit this file）
4. 點右上角 **...** → **Upload file** 或直接拖曳新檔案覆蓋
5. 按 **Commit changes**
6. 等 1-2 分鐘，Mac 和 iPad 自動更新

### 情況 B：補充缺漏題目

**你做的事：**
1. 把考卷截圖傳給 Claude
2. 說明學校和年份：「這是政大 113 年第 5-10 題」

**Claude 做的事：** 把題目轉成 JSON 加進題庫，回傳新的 `題庫.json`

**你做的事：** 同情況 A，覆蓋 GitHub 上的 `題庫.json`

### 目前各校題庫完整度

| 學校 | 現有題數 | 缺漏 | 優先度 |
|------|---------|------|--------|
| 東吳 | 56 題 | 無 | ✅ 完整 |
| 台大 | 46 題 | 約 12 題 | 🟡 小缺漏 |
| 台綜成大 | 43 題 | 約 10 題 | 🟡 小缺漏 |
| 政大 | 97 題 | 約 98 題 | 🔴 嚴重缺漏 |

**補題優先順序（政大）：**
- 113 年缺 32 題 ← 最優先
- 114 年缺 23 題
- 111 年缺 18 題
- 112 年缺 17 題
- 109 年缺 8 題

---

## 四、題庫更新後如何刷新

### Mac
```
Cmd + Shift + R（強制重新整理）
```

### iPad
1. 關閉主畫面 App，重新打開
2. 或在 Safari 清除快取後重新整理

---

## 五、答題紀錄說明

- 答題紀錄存在**瀏覽器 localStorage**
- 清除瀏覽器快取會消失
- Mac 和 iPad 的紀錄是**分開的**，無法同步
- 建議主要在一個裝置上刷題

---

## 六、題目 ID 規則

| 學校 | 前綴 | 範例 |
|------|------|------|
| 台大 | NTU | NTU-108-01 |
| 台綜成大 | NCKU | NCKU-107-03 |
| 東吳 | SCU | SCU-109-05 |
| 政大 | NCCU | NCCU-111-12 |

格式：`學校-年份-題號`

---

## 七、常見問題

**Q：打開網頁顯示「載入失敗」**
A：確認有用 `python3 -m http.server 8080` 啟動，不能直接點兩下 index.html 開啟

**Q：GitHub Pages 網址打不開**
A：等多幾分鐘，或到 Settings → Pages 確認有顯示網址

**Q：題庫更新後沒有變化**
A：強制重新整理：Mac 按 `Cmd + Shift + R`，iPad 清除 Safari 快取

**Q：iPad 加到主畫面後題庫沒更新**
A：需要關閉 App 重新打開，或在 Safari 先重新整理再加入主畫面
