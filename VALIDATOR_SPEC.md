# Spec：遠端設定檢查工具

給接手的 agent。目標是**擋錯**，不是做編輯器——JSON 本身不難改，難的是改完不知道對不對。

## 要防的四類錯（都是 2026-09 真的踩過的）

| 錯誤 | 次數 | 後果 |
| :--- | :--- | :--- |
| 加了欄位只加一份，兩份飄移 | 1 | staging 驗出來的行為跟正式版不同，**沒有任何錯誤訊息** |
| 翻了旗標但線上版讀不到 | **3** | 以為止血了，其實什麼都沒發生 |
| 推上去後看 raw 以為沒推成功 | 1 | CDN 快取，白白重推 |
| 不確定旗標的用途／預設值 | 一直 | 每次都要翻 App 原始碼 |

`healthConnectEnabled`、`adHighTierEnabled`、`adPreloadEnabled` 三次都是同一件事：
**旗標只對「帶著讀取程式碼的版本」有效**，新欄位對舊版本完全沒有作用。

## 產出

在本 repo（`kevycheng/geosim-data`）建立三個檔案。**不要動 App 的程式碼。**

```
flags.json                        旗標登記表
validate.py                       檢查腳本（純標準函式庫，不要額外相依）
.github/workflows/validate.yml    push 時自動跑
```

### 1. `flags.json`

每個旗標一筆。`since` 是**第一個讀得到這個欄位的 versionCode**。

```json
{
  "flags": {
    "adPreloadEnabled": {
      "type": "boolean",
      "default": true,
      "since": 47,
      "desc": "true=開能量頁就先載一支廣告；false=按下才載（顯示率高但要等 1~3 秒）"
    }
  },
  "intentionalDifferences": [
    "bookmarksUrl", "minSupportedVersionCode", "inAppUpdateStalenessDays",
    "promo.codeHash", "healthDailyStepCap", "healthStepAdvancedEnabled",
    "messageTitle", "message", "firmwareVersion", "updatedAt"
  ],
  "liveVersionCode": 42
}
```

**怎麼填 `since`**：不要用猜的。到 `geosim-app` 用

```
git log --oneline -S'"欄位名"' -- app/src/main/java/com/example/mocklocation/RemoteConfigStore.kt
```

找出引入該欄位的 commit，再回推那次 bump 到哪個 versionCode（查 `CHANGELOG.md`）。
**查不出來的填 `null`**，腳本對 `null` 不做版本警告。不要編造版號。

欄位清單以 `RemoteConfigStore.kt` 的 `RemoteConfig` data class 與 `parseRemoteConfig()` 為準
（兩者必須一致，順便檢查有沒有「宣告了但沒 parse」的欄位）。

`liveVersionCode` 手動維護＝目前正式版上架且鋪滿的版號。

### 2. `validate.py`

讀 `remote_config.json` 與 `remote_config.staging.json`，檢查：

| # | 檢查 | 等級 |
| :-- | :--- | :--- |
| 1 | JSON 語法正確 | ✗ 失敗 |
| 2 | 沒有 `flags.json` 不認識的鍵——**順便對拼錯的鍵做近似比對並提示**（`adHighTierEnable` → 你是不是要寫 `adHighTierEnabled`） | ✗ 失敗 |
| 3 | 型別符合 `flags.json`（擋 `"false"` 字串當布林這種） | ✗ 失敗 |
| 4 | 兩份的鍵集合一致，`intentionalDifferences` 以外不得單邊缺少 | ✗ 失敗 |
| 5 | `minSupportedVersionCode` ≤ `liveVersionCode`（設成尚未 live 的號＝**撞強制更新牆但商店沒新版＝鎖死用戶**） | ✗ 失敗 |
| 6 | 某欄位的 `since` > `liveVersionCode` → 提醒「此旗標目前不會生效」 | ⚠ 警告 |
| 7 | `proPromote` / `proPurchaseEnabled` / `proAdUnlockEnabled` 的組合：`proPromote=false` 時另外兩個至少要有一個 `true`，否則玩家會遇到「功能被收走又沒有替代路徑」 | ⚠ 警告 |
| 8 | `freeTier.enabled=false` → 警告這是 kill-switch，等於全放行 | ⚠ 警告 |

**輸出格式**：每行 `✓ / ⚠ / ✗ 訊息`，有 ✗ 就 exit 1。訊息要講清楚**為什麼**，不是只說哪裡錯。

再加一個 `--remote` 模式：用
`gh api repos/kevycheng/geosim-data/contents/<file> -q .content | base64 -d`
抓線上實際內容來比對本機，確認推上去了。**不要用 raw.githubusercontent 判斷**——它有 CDN 快取，
會讓你以為沒推成功（踩過）。

### 3. GitHub Action

`push` 到 main 時跑 `validate.py`。這一項不能省：**設定常常是直接在 GitHub 網頁編輯的**，
本機的腳本或 pre-commit hook 攔不到。

## 驗收

1. `python validate.py` 對現況全過（現在兩份是同步的）
2. 手動把 `remote_config.staging.json` 的 `adPreloadEnabled` 刪掉 → 第 4 項要失敗
3. 手動把某個鍵拼錯 → 第 2 項要失敗，且要提示正確拼法
4. 把 `minSupportedVersionCode` 改成 `999` → 第 5 項要失敗
5. Action 在 PR 上會顯示結果
6. 把上述手動改動全部還原

## 不要做的

- 不要做互動式編輯器／網頁介面。價值在擋錯，不在編輯方便
- 不要引入第三方套件（`jsonschema` 之類）。純標準函式庫，Action 才不用裝東西
- 不要自動修正。只回報，讓人決定——自動改正式環境的設定太危險
- **不要編造 `since` 版號。** 查不到就填 `null`

## 背景

`README.md` 有兩份設定刻意不同的完整理由，以及各旗標的說明。先讀它。
