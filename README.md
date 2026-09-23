# 威泰影城售票系統

軟體工程課程專案，以 React、TypeScript 與 Node.js 建立電影售票網站，涵蓋電影資訊瀏覽、場次與座位選擇、餐點加購、訂單及會員管理，並透過 TMDB 取得電影資料。

本資料夾包含程式碼、使用手冊與 PDF 文件。以下說明以目前程式碼為準；本版本仍有缺漏及模擬功能，執行前請先閱讀「目前限制」。

## 主要功能

- **電影資訊**：查詢熱映中、即將上映電影及電影詳情，串接 TMDB。
- **訂票流程**：選擇場次、座位與票種，加購餐點並建立訂單。
- **付款展示**：選擇付款方式，以模擬流程更新訂單狀態。
- **訂單管理**：我的票夾、訂單詳情、退票與部分狀態的訂單刪除。
- **會員管理**：本地帳號登入／註冊 API、Manus OAuth、個人資料、密碼修改、儲值紀錄及 VIP 升級邏輯。
- **密碼重設**：產生有時效的重設連結，目前輸出至伺服器主控台，尚未寄送電子郵件。

## 技術架構

| 層級 | 使用技術 |
| --- | --- |
| 前端 | React 19、TypeScript、Vite 7、Wouter |
| 介面 | Tailwind CSS 4、Radix UI／shadcn/ui、Lucide React |
| 資料請求 | tRPC 11、TanStack React Query |
| 後端 | Node.js、Express 4、Zod |
| 資料庫 | MySQL、Drizzle ORM、Drizzle Kit |
| 測試 | Vitest |
| 套件管理 | pnpm（`packageManager` 指定 10.4.1） |

開發模式由 Express 整合 Vite，API 掛載於 `/api/trpc`；正式模式由同一個伺服器提供編譯後的靜態檔案。

## 目錄結構

```text
軟體工程/
├── README.md
├── B1222022_B1222022/           # 專案程式碼
│   ├── client/                 # 前端入口及靜態資源
│   │   └── src/
│   │       ├── pages/          # 電影、訂票、付款、會員等頁面
│   │       ├── components/     # 共用元件與 UI 元件
│   │       ├── hooks/          # React hooks
│   │       ├── contexts/       # 主題等 Context
│   │       └── lib/            # tRPC 與共用工具
│   ├── server/
│   │   ├── _core/              # 伺服器入口、驗證、服務整合
│   │   ├── routers.ts          # tRPC API
│   │   ├── db.ts               # 資料庫存取
│   │   ├── tmdb.ts             # TMDB 服務
│   │   └── *.test.ts           # 測試
│   ├── drizzle/                # 資料表定義與 SQL 遷移
│   ├── shared/                 # 前後端共用型別與常數
│   ├── patches/                # pnpm 套件修補
│   ├── seed-data.mjs           # 基礎示範資料
│   ├── seed-showtimes.mjs      # 示範影廳、座位與場次資料
│   ├── seed-showtimes-tmdb.mjs  # 另一份場次建立腳本
│   ├── USER_MANUAL.md          # 系統使用手冊
│   ├── todo.md                 # 開發工作紀錄
│   └── package.json
└── *.pdf                       # 隨附報告與手冊
```

## 本機執行

### 1. 準備環境與安裝套件

準備支援 Vite 7 的 Node.js 環境、pnpm 與 MySQL（原手冊採用 MySQL 8.0），並備妥 TMDB API Key。專案未提供 `.env.example`。

在本 README 所在資料夾執行：

```bash
cd B1222022_B1222022
pnpm install
```

目前程式引用 `bcryptjs`，但 `package.json` 尚未宣告。若要補齊執行依賴，可執行：

```bash
pnpm add bcryptjs
```

此外，需補回 `client/src/pages/Register.tsx` 與 `client/src/pages/TicketVerification.tsx`，或調整引用它們的路由，才能完成前端編譯。

### 2. 設定環境變數

在 `B1222022_B1222022/` 建立 `.env`，依實際環境填寫：

```dotenv
DATABASE_URL=mysql://USER:PASSWORD@localhost:3306/vieshow
TMDB_API_KEY=YOUR_TMDB_API_KEY
JWT_SECRET=REPLACE_WITH_A_RANDOM_SECRET
PORT=3000

# Manus OAuth 設定（使用 OAuth 登入時需填入有效設定）
VITE_APP_ID=YOUR_MANUS_APP_ID
OAUTH_SERVER_URL=https://YOUR_OAUTH_SERVER
VITE_OAUTH_PORTAL_URL=https://YOUR_OAUTH_PORTAL
```

| 變數 | 用途 |
| --- | --- |
| `DATABASE_URL` | MySQL 連線字串，資料庫操作及遷移所需 |
| `TMDB_API_KEY` | 電影列表、搜尋及詳情的 API 金鑰 |
| `JWT_SECRET` | Session JWT 簽章與驗證 |
| `PORT` | 伺服器起始連接埠，預設 `3000` |
| `VITE_APP_ID` | Manus 應用程式識別碼 |
| `OAUTH_SERVER_URL` | 後端 OAuth 服務位置 |
| `VITE_OAUTH_PORTAL_URL` | 前端 OAuth 登入入口 |
| `OWNER_OPEN_ID` | 選用，擁有者身分設定 |
| `BUILT_IN_FORGE_API_URL`、`BUILT_IN_FORGE_API_KEY` | 選用，內建擴充服務使用 |
| `VITE_ANALYTICS_ENDPOINT`、`VITE_ANALYTICS_WEBSITE_ID` | `client/index.html` 的分析腳本設定；未使用時可移除該 script |

`.env` 已列入專案的 `.gitignore`。`VITE_` 前綴的設定會提供給前端，請勿用來存放私密金鑰。

### 3. 建立資料庫與示範資料

先在 MySQL 建立與 `DATABASE_URL` 對應的空白資料庫，例如 `vieshow`。以下指令明確載入 `.env` 後執行遷移，再建立示範資料：

```bash
node --import dotenv/config ./node_modules/drizzle-kit/bin.cjs generate
node --import dotenv/config ./node_modules/drizzle-kit/bin.cjs migrate
pnpm exec tsx seed-data.mjs
pnpm exec tsx seed-showtimes.mjs
```

若已由執行環境匯入 `DATABASE_URL`，可改用 `pnpm db:push` 執行前兩項遷移工作。此指令實際執行的是 `generate` 與 `migrate`。

種子腳本含固定 ID、示範電影及日期，應先檢查再用於空白的開發資料庫；重複執行可能產生重複資料或衝突。`seed-showtimes-tmdb.mjs` 另有固定影城／影廳 ID 與連線設定，不屬於上述預設初始化流程。TMDB 電影與本地電影的 ID 對應也需另外確認。

### 4. 啟動開發伺服器

```bash
pnpm dev
```

預設瀏覽位置為 `http://localhost:3000`。若連接埠被占用，程式會往後尋找可用連接埠，請以終端機輸出為準。

### 5. 編譯與啟動正式模式

```bash
pnpm build
pnpm start
```

編譯結果輸出至 `dist/`。後端套件採外部依賴方式打包，因此執行正式模式仍需已安裝的依賴套件及環境設定。

## 常用指令與測試

以下指令皆在 `B1222022_B1222022/` 執行。

| 指令 | 說明 |
| --- | --- |
| `pnpm dev` | 啟動開發模式並監看變更 |
| `pnpm build` | 編譯前端與後端 |
| `pnpm start` | 執行編譯後的伺服器 |
| `pnpm check` | TypeScript 型別檢查 |
| `pnpm test` | 執行 Vitest 測試 |
| `pnpm format` | 使用 Prettier 格式化專案檔案 |
| `pnpm db:push` | 產生並執行資料庫遷移 |

現有測試包含登出 Cookie、電影／影城／場次／票種／餐點、會員資料、優惠及 TMDB 服務。部分測試需要資料庫、示範資料或有效 TMDB 金鑰，會員測試也會更新資料，請使用測試資料庫。

測試入口不會自動載入 `.env`；若需使用該檔案的設定，可執行：

```bash
node --import dotenv/config ./node_modules/vitest/vitest.mjs run
```

本 README 依靜態檔案檢查撰寫，未執行安裝、建置、資料庫遷移或測試，不代表目前版本已通過驗證。

## 目前限制

- **頁面檔案缺漏**：`App.tsx` 引用了不存在的 `Register.tsx` 與 `TicketVerification.tsx`。
- **依賴未完整宣告**：後端使用 `bcryptjs`，但套件清單未包含它。
- **本地登入 Session 不一致**：登入 API 將 `openId` 直接寫入 Cookie，驗證端則以 JWT 驗證；需修正後才能確保登入後的會員操作正常。
- **付款及儲值為展示邏輯**：付款頁模擬處理後更新狀態，儲值 API 直接更新餘額，未串接真實金流。
- **金額計算簡化**：建立訂單時以每席 300 元計算，未完整套用票種、餐點及優惠規則。
- **部分場次為虛擬資料**：未找到本地電影對應時，程式會產生未寫入資料庫的場次；完整訂票需使用有效的本地場次與座位資料。
- **文件與程式有差異**：原手冊的退票手續費、退款方式等描述與實作不同；目前退票 API 檢查開演前兩小時限制，並將訂單金額加回會員餘額。

## 隨附文件

- [系統使用手冊（Markdown）](B1222022_B1222022/USER_MANUAL.md)
- [開發工作紀錄](B1222022_B1222022/todo.md)
- [測試報告（PDF）](B1222022_B1222022%25E6%25B8%25AC%25E8%25A9%25A6%25E5%25A0%25B1%25E5%2591%258A.pdf)
- [系統使用手冊（PDF）](B1222022_B1222022%25E7%25B3%25BB%25E7%25B5%25B1%25E4%25BD%25BF%25E7%2594%25A8%25E6%2589%258B%25E5%2586%258A.pdf)
- [附加文件 1](B1222022_B1222022-2.pdf)
- [附加文件 2](B1222022_B1222022-3.pdf)

## 授權資訊

`package.json` 的授權欄位標示為 `MIT`，目前資料夾未附獨立 `LICENSE` 檔案。使用手冊另有版權聲明，文件授權請參考原文。
