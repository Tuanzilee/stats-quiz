# 心統刷題系統

## 資料夾結構
```
心統刷題/
├── index.html    ← 主程式（幾乎不需要動）
├── 題庫.json     ← 題庫（更新題目只動這個）
├── manifest.json ← PWA 設定
└── sw.js         ← 離線快取
```

## 本機啟動（Mac）
```bash
cd 心統刷題
python3 -m http.server 8080
```
然後打開瀏覽器：http://localhost:8080

## GitHub Pages 部署（iPad 也可用）
1. 在 GitHub 建立新 repo（如 `stats-quiz`）
2. 把整個資料夾上傳
3. Settings → Pages → Deploy from main branch
4. 網址：https://你的帳號.github.io/stats-quiz

## 題庫更新流程
1. 把新的 `題庫.json` 覆蓋舊的
2. 本機：重新整理瀏覽器
3. GitHub：上傳覆蓋，等 1-2 分鐘自動更新

## 答題紀錄
- 存在瀏覽器 localStorage
- 清除瀏覽器快取會消失
- 換裝置無法同步（之後可考慮加入帳號功能）

## 快捷操作
- 考點學習：考點頁 → 點考點 → 刷相關題目
- 錯題複習：主頁 → 錯題複習
- 弱點追蹤：主頁下方自動顯示答錯率最高的考點
