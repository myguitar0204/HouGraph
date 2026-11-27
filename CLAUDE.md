# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案性質

房屋平面圖繪圖工具：取代手工在方格紙上畫房屋平面圖並計算面積的工作。介面採左右分欄：左側是圖形範本按鈕與輸入欄位，右側是方格畫布。整個應用就是單一檔案 `index.html`（HTML + CSS + 原生 JS + SVG），沒有建置步驟、沒有外部依賴，UI 文字一律繁體中文。

## 執行與測試

- 使用：瀏覽器直接開啟 `index.html`。
- Playwright MCP 會擋 `file://`，測試時要先起本機伺服器（背景執行），再連到 `http://127.0.0.1:8765/index.html`（用 `localhost` 會連不上）：
  ```
  uv run python -m http.server 8765
  ```
- 沒有自動化測試，靠 Playwright 驗證。注意：
  - 用 `dispatchEvent(new PointerEvent(...))` 模擬拖曳會因 `setPointerCapture` 找不到 active pointer 而失敗，拖曳要用 `page.mouse`。
  - 目標元素必須在 viewport 內，滑鼠事件才會命中。
  - 驗證面積要拿已知答案比對（例如 4.7×19 = 89.30），不能只看有沒有輸出。
  - 測完要刪掉 Playwright 在專案目錄產生的 `.playwright-mcp/`。

## 架構重點

**座標系**：文件內容一律用公尺（世界座標，y 軸朝下）。畫面轉換由 `view = {s, tx, ty}` 決定，`s` 是每公尺的像素數，`BASE = 40` 代表 100%。拖曳以 0.1 m 吸附（`snap`）。

**圖形登錄表 `SHAPES`**：每種圖形定義 `fields`（輸入欄位）、`pts(p)`（頂點）、`dims(p)`（尺寸線 `[起點, 終點, 數值]`）。
- 頂點必須是**畫面上的順時針**（shoelace 和為正）。尺寸線會畫在行進方向的左手法線 `(dy, -dx)` 側，只有順時針時才會在圖形外側。不規則形（`poly`）完成繪製時，若方向相反會反轉。
- `geom(sh)` 先以 90° 為單位旋轉（`rot` 0–3），再把外框左上角對齊到 `(sh.x, sh.y)`。面積與形心由 `centroid()`（shoelace）計算，所有圖形共用。
- 新增圖形種類只需加一筆登錄，再在 `validate()` 補檢查條件。

**文件模型 `doc`**：
- `shapes: [{id, type, params, label, floor, x, y, rot}]`，`poly` 的 `params.pts` 是相對座標
- `texts: [{id, text, size, x, y}]`，`size` 為公尺
- `meta`（列印表頭）、`nextId`

存在 `localStorage` 的 `hougraph-v1`，左側面板寬度存在 `hougraph-v1-side`。JSON 存檔、開檔的格式也是 `doc`。

**狀態變更流程**：修改 `doc` 後要呼叫 `save()`。它同時負責寫入 localStorage 和推入復原快照（`history` 存的是 JSON 字串，最多 100 步）。拖曳過程中不呼叫，放開時才存。整份 `doc` 被替換時（復原、開檔、清空），要再呼叫 `renderMeta()` 與 `select()`。

**繪製方式**：`render()` 每次都以 `innerHTML` 整個重建畫布 SVG，`pointermove` 也一樣。因此：
- 不可依賴 SVG 子元素上的 `click`、`dblclick`，因為點擊目標會被替換掉。不規則形的「雙擊完成」是在 `pointerdown` 判斷「再點同一頂點」；雙擊圖形進入頂點編輯則用 `lastTap`（400 ms）自行判斷。
- pointer capture 綁在 `#svg` 本身。
- `render()` 結尾會呼叫 `renderAreas()`，面積表永遠和畫面同步。
- 線寬使用 `vector-effect="non-scaling-stroke"`；文字和格線的線寬則以世界單位換算（例如 `1 / s`）。

**頂點編輯**（`editing`、`activeV`）：
- 範本圖形在第一次實際改動頂點時，由 `toPoly()` 轉成 `poly`。轉換時沿用 `geom()` 的頂點順序，把旋轉結果寫進世界座標並設 `rot = 0`，因此轉換前後的頂點索引一致。
- 拖曳結束時，若面積為 0 就還原；若方向變成逆時針，就反轉頂點順序。
- `select()` 選到其他物件時，會自動結束編輯。

**畫布模式提示列**：繪製不規則形（`drawing`）和頂點編輯（`editing`）共用同一個 `#drawHint`，由 `updateDrawHint()` 依狀態切換文字和顯示，`select()` 也會呼叫它。新增畫布模式時要接進這個函式，否則提示會殘留。

**已知限制**：沒有檢查多邊形的邊是否互相交叉。邊交叉的形狀（例如 8 字形）用 shoelace 算出的面積會是錯的。

**同一份繪圖碼三處共用**：`contentSVG()` 同時供畫面、PNG 匯出（`drawingSVG(b, 60, 'gx')`）和 A4 列印（`buildPrint()`）使用。
- `gridDefs(s, id)` 的 pattern id 在同一份 HTML 內必須唯一：畫面用 `g`，列印用 `gp`。
- 列印會從 `SCALES` 挑出能放進 `PRINT_W × PRINT_H`（mm）的最小比例尺，再用 `MM_PER_PX = 25.4/96` 換算成 CSS px。實際列印時縮放必須選 100%，比例尺才準確。

**面積分組**：`areaGroups()` 先依 `floor` 分組，以數字感知排序（1F < 2F < 10F）。同一樓層內 `label` **完全相同**的圖形會合併成一列，沒有標籤的各自獨立。

**CSS**：新增 class 前先搜尋是否撞名。曾發生面積表的 `.sub` 套到頁首副標題樣式的情況，現已改名為 `.subt`。
