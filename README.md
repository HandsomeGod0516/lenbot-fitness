# fitness

lenbot 的健身紀錄子專案，掛在 `/projects/fitness` 路由下。

> 這個資料夾不是獨立 Nuxt app，是個 `App.vue` 元件，被 [`portal-main`](https://github.com/HandsomeGod0516/lenbot-portal-main) 用 Vite glob 自動載入；API 也由 portal-main 提供。

## 功能

- **菜單** (admin)：建立健身菜單，第一個動作可以是多個動作組成的 superset
- **記錄** (任何登入使用者)：依菜單記錄每組重量與次數
- **大家的成長** (任何登入使用者)：看每個動作每個人的成長曲線

## DB 結構

在 portal-main 的 `db/init.sql` 維護：

- `workout_menus` — 菜單本體
- `workout_menu_exercises` — 動作清單；`position`+`slot`，多個 slot = superset
- `workout_logs` — 一次訓練 session
- `workout_log_sets` — session 內的單一組

## 路由

- 前端：`/projects/fitness`
- API（由 portal-main 提供）：
  - `GET    /api/fitness/menus`
  - `GET    /api/fitness/menus/:id` — detail with grouped exercises
  - `POST   /api/admin/fitness/menus` — admin 建立 (含 exercises array)
  - `DELETE /api/admin/fitness/menus/:id`
  - `POST   /api/fitness/logs` — 紀錄一次 session
  - `GET    /api/fitness/stats` — 公開儀表板資料

## 圖表

純 SVG 自繪折線圖 (`<Chart>` component 定義在 `App.vue` 的 setup 區塊)。x = 日期、y = 當日最大重量。每個使用者一條線，顏色依 username hash 決定。

## 開發

```bash
cd ../portal-main && npm run dev
# 瀏覽 http://localhost:3000/projects/fitness
```
