# fitness

lenbot 的健身紀錄子專案。

## 功能

- **菜單** (admin)：建立健身菜單，第一個動作可以是多個動作組成的 superset
- **記錄** (任何登入使用者)：依菜單記錄每組重量與次數
- **大家的成長** (任何登入使用者)：看每個動作每個人的成長曲線

## DB 結構 (在 portal-main 的 init.sql 維護)

- `workout_menus` — 菜單本體
- `workout_menu_exercises` — 動作清單，`position`+`slot`，多個 slot = superset
- `workout_logs` — 一次訓練 session
- `workout_log_sets` — session 內的單一組

## 路由

- 前端：`/projects/fitness` (透過 portal-main 的 sub-project glob 自動掛載)
- API：
  - `GET    /api/fitness/menus` — list
  - `GET    /api/fitness/menus/:id` — detail with grouped exercises
  - `POST   /api/admin/fitness/menus` — admin 建立 (含 exercises array)
  - `DELETE /api/admin/fitness/menus/:id` — admin 刪除
  - `POST   /api/fitness/logs` — 紀錄一次 session (含多組 sets)
  - `GET    /api/fitness/stats` — 公開儀表板資料 + 自己的最近紀錄

## 圖表

純 SVG 自繪折線圖 (`<Chart>` component 定義在 App.vue 的 setup 區塊)。x=日期、y=當日最大重量。每個使用者一條線，顏色依 username hash 決定。
