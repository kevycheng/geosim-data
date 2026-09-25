# geosim-data

GeoSim 的遠端設定與官方內容來源。App 直接讀 `raw.githubusercontent.com`，**推上 main 就生效**
（CDN 有數分鐘快取）。

| 檔案 | 誰在讀 |
| :--- | :--- |
| `remote_config.json` | **正式版**（release build） |
| `remote_config.staging.json` | staging / debug build |
| `official_bookmarks.json` / `.staging.json` | 官方精選書籤 |
| `official_routes.json` | 官方路線 |
| `vip.json` | VIP 裝置碼白名單 |

---

## 改之前先看這裡

**旗標只對「帶著讀取程式碼的版本」有效。** 新欄位對舊版本完全沒有作用——它們的
`parseRemoteConfig` 根本沒有那個鍵。想靠遠端設定止血之前，先確認目標版本讀得到。

這件事踩過三次：`healthConnectEnabled`、`adHighTierEnabled`、`adPreloadEnabled`，
每次都是「以為能翻旗標，結果非發版不可」。

**改完務必用 API 複驗**，不要只看 raw（raw 有 CDN 快取，會讓你以為沒推上去）：

```bash
gh api repos/kevycheng/geosim-data/contents/remote_config.json -q .content | base64 -d
```

---

## 兩份設定「刻意」不同的地方

同步兩份時**不要**把下面這些抹平——它們是故意的：

| 欄位 | 正式版 | staging | 為什麼 |
| :--- | :--- | :--- | :--- |
| `bookmarksUrl` | `official_bookmarks.json` | `...staging.json` | staging 要能試新書籤不影響玩家 |
| `minSupportedVersionCode` | 只設**已上架且鋪滿**的版號 | 可以很舊（如 14） | 正式版設錯會**鎖死用戶**（撞強制更新牆但商店沒新版）。staging 放寬方便測舊版 |
| `inAppUpdateStalenessDays` | `1` | **`0`** | Internal App Sharing 的版本不屬於任何軌道，Play 回報的 staleness 是 null（程式當 0）。門檻留著就整個跳過，看起來像功能壞了 |
| `promo.codeHash` | 正式優惠碼 | 測試碼 | 測試碼不該能在正式版兌換 |
| `healthDailyStepCap` | `25000` | `48000` | staging 用大一點的數字方便測上限行為 |
| `healthStepAdvancedEnabled` | 不設（false） | `true` | staging 顯示細部上限 UI 才驗得到 |
| `messageTitle` / `message` / `firmwareVersion` / `updatedAt` | 各自獨立 | 各自獨立 | 公告本來就不該共用。**`firmwareVersion` 不換，看過的人不會再跳** |

其餘欄位**應該保持一致**。新增旗標時請**兩份都寫**，即使值相同——
只寫一份會讓下次 diff 變成「缺 vs 有」，分不出是刻意還是忘了。

---

## 幾個容易誤會的旗標

- **`freeTier.enabled`** 是 kill-switch。設 `false` ＝**全放行**（不 gating、不限時間），
  只在出大問題要緊急放行時用，不是日常開關。
- **`adHighTierEnabled`**：激勵廣告的兩層瀑布流。**預設 `false`**。
  2026-09 實測過一次，混合 eCPM 反而下降、一般層品質被拖累，已停用。
  詳見 geosim-app issue #7。
- **`adPreloadEnabled`**：`true` ＝開啟能量頁就先載一支（按下立刻播）；
  `false` ＝按下才載（等 1～3 秒，但顯示率接近 100%）。
- **`proPromote` / `proPurchaseEnabled` / `proAdUnlockEnabled`** 是一組，
  切換商業模式時**要同一次翻**，否則玩家會遇到「功能被收走又沒有替代路徑」。
  手冊在 geosim-app 的 `docs/commercial-cutover-checklist.md`。
