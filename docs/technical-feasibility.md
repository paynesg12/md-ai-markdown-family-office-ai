# 技術可行性研究：單一 HTML 家庭辦公室原型

研究日期：2026-10-23

## 結論

指定的第一版架構可行：一個由本機 HTTP 伺服器提供的 HTML 檔，可用 ES module import map 載入固定版 `three@0.160.0`，並以程式生成場景、HTML 表單與 `localStorage` 完成互動原型。最大的交付風險不是 Three.js API，而是外部 CDN / 模組載入失敗、WebGL 不可用或執行中 context 遺失、以及 `file:` 直接開檔下模組與儲存行為不可靠。因此文字總覽與資料編輯層必須獨立於 3D 初始化，並隨檔案提供本機伺服器啟動方式。

## 可行的元件與約束

| 主題 | 結論 | 實作含意 |
| --- | --- | --- |
| ES module + import map | 可行。瀏覽器對 bare module specifier 需要 import map；未解析時會丟出 `TypeError`。 | import map 必須放在模組腳本前；初始化要有可見的錯誤介面。 |
| 單檔 HTML | 可行。模組可直接內嵌於 `type="module"` 腳本。 | 所有 UI、資料與生成邏輯可放同一檔；仍需 HTTP server 供測試。 |
| Three.js addons | 可行。`OrbitControls` 和 `CSS2DRenderer` 都是需明確匯入的 addon。 | 將 `three` 與 `three/addons/` 同時映射到完全相同的 0.160.0 CDN 版本，避免核心與 addon 版本混用。 |
| 3D 標籤 | 可行，但 CSS2DRenderer 有限制。 | 它適合 HTML 標籤，但只支援平移與 2D 旋轉，且官方說明僅支援 100% 瀏覽器／顯示縮放；不能讓標籤成為唯一操作入口。 |
| 自適應尺寸 | 可行。`ResizeObserver` 已廣泛可用。 | 觀察 3D 容器、在尺寸變化時更新 camera/renderer/CSS2DRenderer；不要在 observer 中反覆改同一觀察元素尺寸，必要時用 `requestAnimationFrame` 延後，避免 resize loop。 |
| reduced motion | 可行。`matchMedia()` 所得的 `MediaQueryList` 可用 `change` 事件監聽狀態改變。 | 首次讀取 `matches`，並在變更後停止自動動畫、將相機移動切換為立即完成；CSS 同步移除非必要 transition。 |
| 本機保存 | 可行，但不是保證可用。 | 每一次取得、讀取、寫入 `localStorage` 都包在 `try/catch`；資料 schema 驗證失敗時退回記憶體中的範例資料並顯示非阻塞提示。 |
| WebGL 降級 | 必要且可行。 | 先渲染不依賴 WebGL 的資訊與表單；只有在 WebGL / Three 初始化成功後掛入 canvas。初始化或 CDN 模組失敗時保留文字版，並提供清楚錯誤訊息與重試方式。 |

## 主要風險與處置

### 1. 不可用 `file:` 直接開檔作為正式交付路徑

模組有 CORS 安全要求，MDN 明確指出以 `file://` 載入本機 HTML 進行模組測試會碰到 CORS 錯誤，應經由 server 測試。此外，`localStorage` 在 `file:` URL 的要求未定義、各瀏覽器可能不同，甚至存取時可拋出 `SecurityError`。交付 README 應提供本機 HTTP server 指令與精確 URL；程式仍要在直接開檔失敗時留下文字總覽與提示。

### 2. CDN 與模組載入為單點失敗

`type="module"` 的跨來源載入使用 CORS；因此網路、CDN、企業網路政策或回應 CORS header 任一失敗，都可能使 Three 模組不可用。採用釘選的 `0.160.0` 可避免版本漂移，但不消除可用性風險。建議：

- 以 UI 先行、3D 漸進增強（progressive enhancement）。
- 在 `window` error / `unhandledrejection` 與 Three 初始化 `try/catch` 中顯示「3D 視覺未載入，文字功能仍可用」。
- 若未來要離線使用，將已核可的 Three 模組併入交付物或改由受控內網 server 提供；這會改變「只載 CDN」的目前方向，需另行決定。

Three.js 官方安裝指南也以相同形式示範 CDN import map，並提醒核心與 addons 應使用同一版本與來源；混用可能出現重複程式碼或非預期失敗。本案給定的 `three@0.160.0/build/three.module.js` 與 `three@0.160.0/examples/jsm/` 因而是相容的配對。import map 必須在任何使用其 bare specifier 的 module script 之前處理，且 `/` 結尾的 prefix key / value 必須成對保留。

### 3. WebGL context 可能不存在或在使用期間遺失

WebGL 規格定義了 `webglcontextlost` 和 `webglcontextrestored`。context 恢復後，先前的 buffer、texture 等資源不再有效，程式必須重建資源和狀態。處置：

- 建立 renderer 前偵測 context 是否可建立並捕捉 renderer 建構錯誤。
- canvas 監聽 `webglcontextlost`：停止 render loop、顯示 3D 暫停狀態，且呼叫 `preventDefault()` 才能接收後續 restored 事件。
- `webglcontextrestored` 走完整的場景／renderer 資源重建流程，或提供明確「重新載入 3D」按鈕；文字與資料層不可依賴該流程。
- 對手機維持像素比上限與較少裝飾物，降低 GPU 記憶體與電量壓力。這是工程上的風險緩解，不代表能保證裝置不會遺失 context。

### 4. CSS2D 標籤與可及性

CSS2DRenderer 可把 DOM 元素包進 `CSS2DObject` 並依場景定位，適合島嶼標籤；但其官方限制使縮放情境可能不準確，且被遮住、觸控命中與 canvas 疊層管理都可能造成可用性問題。保留頁面內五島清單與詳情面板按鈕，作為完全等價的文字操作路徑；表單區需置於 canvas / label pointer events 之外，確保輸入不被 3D 事件攔截。

### 5. OrbitControls 與動畫週期

OrbitControls 在 `enableDamping` 或 `autoRotate` 啟用時需要每幀 `update()`。此需求和「沒有動畫或互動時避免不必要重繪」有衝突。建議只在下列情況排程 requestAnimationFrame：使用者拖曳、相機 800ms 飛行、damping 尚未靜止，或有意義的環境動畫；靜止後停止 render loop。使用者開始拖曳時立刻取消相機 fly-to，符合互動規格。

### 6. localStorage 是本機、每一 origin 的資料，不是同步

`localStorage` 會跨瀏覽器 session 保留，但資料對 protocol/host/port（origin）隔離；HTTP 與 HTTPS 即使同一網域也會得到不同 storage。因此 README 要把預覽 URL 的 port 當成資料保存位置的一部分，且 UI 不得稱它為多人或跨裝置同步。無痕模式與封鎖持久化的環境也可能拒絕存取。JSON 匯入要 `JSON.parse` 捕捉錯誤、做白名單 schema／數值有限性檢查，插入畫面一律用 `textContent` 或安全的 DOM 節點 API，不能串接不受信任字串到 `innerHTML`。

## Three.js 設定核對

- `WebGLRenderer.outputColorSpace` 是輸出色彩轉換的設定點；顯示 canvas 應使用 sRGB，因此使用 `THREE.SRGBColorSpace` 合理。
- `OrbitControls` 官方範例支援軌道、縮放與平移，並指出手動改相機 transform 後須 `update()`；這可支援島嶼選取後的相機飛行。
- CSS2D label 不是 3D mesh renderer 的替代品：CSS2DRenderer 會忽略 mesh / point cloud，只處理 DOM label，所以每個 frame 仍要呼叫 WebGL renderer 和 CSS2DRenderer 的 render。
- `ACESFilmicToneMapping`、soft shadow、instancing、以距離隱藏裝飾物均屬合理的視覺／效能權衡；但裝置實測仍是必要驗收項目，不能只由設定保證手機效能。

## 建議的驗證案例

1. 以 HTTP server 開啟 HTML，驗證 import map 模組和五個 addon 的載入；網路中斷／封鎖 CDN 時，確認文字總覽仍可操作並顯示錯誤。
2. 在有 WebGL 與禁用 WebGL（或使 renderer 建構失敗）的環境各測一次；兩者都要能看到資料和討論／待辦 UI。
3. 觸發或以 `WEBGL_lose_context` 測試 context loss；確認不白畫面、不持續 render，並可恢復或清楚要求重建。
4. 將瀏覽器縮放調離 100%，確認 CSS2D label 的文字清單仍可完成選島；以窄寬手機尺寸測試 44px 觸控目標與表單輸入。
5. 切換 OS 的 reduced motion 設定，確認 `MediaQueryList` change 會停止非必要動畫並取消平滑飛行。
6. 阻擋網站資料、餵入損壞 JSON，或寫入錯誤格式的既有 storage；確認程式使用範例資料回退且不崩潰。

## 主要來源（官方／規格）

- [MDN：JavaScript modules（local `file://`、import maps、bare specifier）](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules)
- [MDN：`script`（module scripts 與 CORS、`importmap`）](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/script)
- [MDN：`importmap` 類型的腳本（處理順序與 prefix 規則）](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/script/type/importmap)
- [Three.js：Installation（CDN import map、addons、同版同來源建議）](https://threejs.org/manual/en/introduction/Installation.html)
- [MDN：`Window.localStorage`（持久性、`SecurityError`、`file:` 未定義行為）](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage)
- [Khronos：WebGL 1.0 Specification（context lost/restored 及資源重建）](https://registry.khronos.org/webgl/specs/1.0/)
- [Three.js：WebGL addon（可用性檢測與錯誤訊息）](https://threejs.org/docs/pages/WebGL.html)
- [Three.js：OrbitControls](https://threejs.org/docs/pages/OrbitControls.html)
- [Three.js：CSS2DRenderer](https://threejs.org/docs/pages/CSS2DRenderer.html)
- [Three.js：Color Management／`outputColorSpace`](https://threejs.org/manual/en/color-management.html)
- [MDN：ResizeObserver（用途與 resize loop 風險）](https://developer.mozilla.org/en-US/docs/Web/API/ResizeObserver)
- [MDN：MediaQueryList（`matchMedia` 的 `change` 事件）](https://developer.mozilla.org/en-US/docs/Web/API/MediaQueryList)
- [MDN：`prefers-reduced-motion`](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/At-rules/%40media/prefers-reduced-motion)
