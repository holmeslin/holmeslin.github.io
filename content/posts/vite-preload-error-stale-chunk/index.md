---
title: "部署後舊分頁點不動：用 vite:preloadError 自動救回"
date: 2026-11-02T12:00:00+08:00
lastmod: 2026-11-02T12:00:00+08:00
draft: true
description: "Vite SPA 部署新版後，停在舊版的分頁會因為找不到舊 chunk 而點不動。用 vite:preloadError 自動重新整理，並處理無限重整、重複觸發、時鐘被調回過去與 storage 被停用等狀況。"
categories: [frontend]
tags: [vite, vue]
---

Vite 打包的 SPA 部署新版之後，常會收到這種回報：「選單點了沒反應，重新整理就好了。」原因是使用者的分頁還停在舊版，舊版要載入的 chunk 在伺服器上已經不存在了。

Vite 官方文件提供了 `vite:preloadError` 事件，範例只有一行 `window.location.reload()`。這一行能解決大部分情況，但它沒處理無限重整、同一次失敗觸發好幾次，也沒顧慮使用者手上正在進行的工作。這篇從官方範例出發，逐一補上這些狀況。

<!--more-->

## 為什麼舊分頁會點不動

Vite 建置時，每個 chunk 的檔名都帶有內容雜湊，例如 `SettingsView-3fa1c2.js`。路由用 `() => import('./views/SettingsView.vue')` 延後載入，使用者第一次進入該頁時才會下載。

部署新版之後：

1. 新版 chunk 的檔名雜湊都換了，舊檔通常會被刪掉（官方文件也提到託管服務可能會刪除舊版的 asset）。
2. 使用者的分頁還開著舊版的 `index.html`，記得的是舊檔名。
3. 使用者點了一個還沒載入過的頁面，瀏覽器去抓 `SettingsView-3fa1c2.js`。
4. 如果伺服器對找不到的路徑一律回傳 `index.html`（SPA 常見的 fallback 設定），瀏覽器拿到的是 HTML，不是 JavaScript，module script 的 MIME type 檢查不通過，動態 import 失敗。
5. vue-router 拿不到元件，導覽中止。畫面上什麼都沒發生，使用者只覺得「點不動」。

## 官方範例：失敗就重新整理

Vite 在動態 import 失敗時會在 `window` 上發出 `vite:preloadError`，`event.payload` 是原本的錯誤。呼叫 `event.preventDefault()` 的話，Vite 就不會再把錯誤丟出去。官方範例是這樣：

```ts
window.addEventListener('vite:preloadError', (event) => {
  window.location.reload()
})
```

重新整理會抓到新版的 `index.html`，新版的 chunk 都在，問題就解決了。用這個事件而不是 `router.onError` 的好處是範圍比較大：我看了 Vite 8 的原始程式碼和建置結果，建置時每個動態 import 都會包進 Vite 的 preload helper，chunk 本身或它依賴的 CSS 載入失敗都會發出這個事件。所以除了路由，延後載入的 store、元件也都涵蓋到了。

但這一行有幾個問題。

## 問題一：無限重新整理

如果 `index.html` 被瀏覽器或 CDN 快取住，重新整理後拿到的還是舊版，再點一次又失敗，又重新整理……分頁就會不停重整。

最直覺的修法是「同一個 URL 只自動重整一次」，用一個旗標記下來。但旗標什麼時候清？如果永遠不清，**下一次部署**時這個分頁就再也不會自動救了；如果載入成功就清，又得在每個成功的地方加程式碼。

比較簡單的做法是用時間窗：記下「上次自動重整的時間」，10 秒內不再重整第二次。窗外的失敗會被當成新的部署，照樣可以自動救。

這個時間必須存在 `sessionStorage`，不能放在模組變數裡，因為重新整理會清掉 JavaScript 的記憶體。依照 MDN 的說明，`sessionStorage` 屬於單一分頁，重新整理後仍然保留，所以一個分頁的紀錄不會影響其他分頁。

## 問題二：一次導覽失敗好幾次

冷啟動時，layout 和 view 可能是兩個不同的 chunk，平行下載、一起失敗。從 Vite 的 preload helper 原始程式碼可以看到，每一個載入失敗的依賴都會各自發出一次 `vite:preloadError`。

時間窗剛好也解決了這個問題：第一個事件觸發重整並寫下時間，第二個事件看到「剛剛才重整過」就不會再呼叫一次 `reload()`。

## 問題三：使用者正在做事

如果使用者正在匯出一份大報表，或者表單填到一半，自動重整會讓這些工作直接消失。這種時候寧可不要自動重整，改成提示「網站已更新，請重新整理頁面」，讓使用者自己決定時機。

## 問題四：時鐘被調回過去

時間窗的判斷是 `elapsed = now - lastReloadAt`。如果使用者的系統時鐘被調回過去，`elapsed` 會變成負數。只寫 `elapsed < 10_000` 的話，負數永遠成立，這個分頁從此只會收到提示，再也不會自動重整。所以要同時要求 `elapsed >= 0`，負數就當成在窗外。

## 問題五：sessionStorage 不能用

MDN 列出 `sessionStorage` 在某些情況會丟 `SecurityError`，例如使用者設定瀏覽器不允許網站儲存資料。沒有地方記錄上次重整的時間，就擋不住無限重整，所以這時候選擇不重整、只提示。

注意事件已經被 `preventDefault()` 吃掉了。如果這裡讓例外往外丟，結果會是既沒重整也沒提示，使用者還是只看到「點不動」。

## 完整的寫法

把判斷抽成一個純函式，storage、現在時間和「是否忙碌」都由參數傳入，方便測試：

```ts
// 存在 sessionStorage：重新整理會清掉記憶體，紀錄要撐過重新整理，而且每個分頁各一份
const KEY = 'app:stale-chunk-reloaded-at'
export const RELOAD_WINDOW_MS = 10_000

type KV = Pick<Storage, 'getItem' | 'setItem'>

export function claimReload(storage: KV, now: number, isBusy: () => boolean): boolean {
  try {
    if (isBusy()) return false
    const elapsed = now - Number(storage.getItem(KEY))
    // 時鐘被調回過去時 elapsed 是負的，當成窗外
    if (elapsed >= 0 && elapsed < RELOAD_WINDOW_MS) return false
    storage.setItem(KEY, String(now))
    return true
  } catch {
    return false
  }
}

export function installStaleChunkReload(isBusy: () => boolean, notify: (msg: string) => void): void {
  window.addEventListener('vite:preloadError', (event) => {
    event.preventDefault() // 不呼叫的話，Vite 會把原本的錯誤再丟出去
    if (claimReload(window.sessionStorage, Date.now(), isBusy)) {
      window.location.reload()
    } else {
      notify('網站已更新，請重新整理頁面')
    }
  })
}
```

第一次沒有紀錄時，`getItem` 回傳 `null`，`Number(null)` 是 `0`，`elapsed` 就是現在的時間戳，一定在窗外，所以會重整。

在 `main.ts` 裡，建立 app 之前就先掛上：

```ts
installStaleChunkReload(
  () => useTaskStore().running, // 你的「長時間任務進行中」判斷
  (msg) => toast.error(msg),
)
```

`isBusy` 會在 `try` 裡面呼叫，所以就算它丟例外（例如 store 還沒初始化），結果也只是不重整。

## 測試

純函式可以直接用 Node 24 內建的 test runner 測，不需要瀏覽器。Node 24 可以直接執行 `.ts` 檔：

```ts
import { test } from 'node:test'
import assert from 'node:assert/strict'
import { claimReload, RELOAD_WINDOW_MS } from './staleChunkReload.ts'

function memStorage(): Pick<Storage, 'getItem' | 'setItem'> {
  const m = new Map<string, string>()
  return { getItem: (k) => m.get(k) ?? null, setItem: (k, v) => void m.set(k, v) }
}
const idle = () => false

test('second failure inside the window does not reload', () => {
  const s = memStorage()
  assert.equal(claimReload(s, 1_000_000, idle), true)
  assert.equal(claimReload(s, 1_000_000, idle), false) // 同時失敗的兩個 chunk
  assert.equal(claimReload(s, 1_000_000 + RELOAD_WINDOW_MS - 1, idle), false)
})

test('clock set backwards counts as outside the window', () => {
  const s = memStorage()
  claimReload(s, 1_000_000, idle)
  assert.equal(claimReload(s, 500_000, idle), true)
})
```

我實際跑過的六個測試涵蓋：

- 第一次失敗會重整
- 時間窗內的第二次（包括同一時間的兩次）不重整
- 時間窗外的失敗（下一次部署）會再重整
- 時鐘被調回過去時照樣重整
- 忙碌時不重整，而且不會寫下紀錄
- storage 丟 `SecurityError` 時不重整

執行方式是 `node --test staleChunkReload.test.ts`，在 Node 24.22 上六個全部通過。

## 這只是安全網

`vite:preloadError` 只在載入已經失敗之後才出手。要減少失敗，得從部署端下手：

- **`index.html` 不要快取**：Vite 官方文件建議對 HTML 設定 `Cache-Control: no-cache`，否則重新整理後可能還是拿到舊版，時間窗只能防止它變成無限重整，救不回來。
- **舊版的 asset 保留一段時間**：如果部署時不刪舊 chunk，舊分頁照樣能載入，根本不會觸發這個事件。帶雜湊的檔名不會互相覆蓋，保留幾個版本的成本很低。

還有一個細節：`preventDefault()` 之後，Vite 不會丟出錯誤，這次動態 import 的結果是 `undefined`。走到「只提示、不重整」那條路時，該次導覽一樣不會成功，提示的目的就是請使用者自己重新整理。

## 小結

| 狀況 | 沒處理會怎樣 | 處理方式 |
|---|---|---|
| `index.html` 被快取，重整後又失敗 | 無限重整 | `sessionStorage` 記錄時間，10 秒內不重整第二次 |
| 下一次部署 | 「只重整一次」的旗標讓分頁再也救不回來 | 用時間窗取代旗標 |
| 同一次導覽多個 chunk 失敗 | 重整好幾次 | 同一個時間窗吸收 |
| 使用者正在進行長時間任務 | 工作消失 | `isBusy()` 時改成提示 |
| 系統時鐘被調回過去 | 分頁之後只剩提示 | `elapsed >= 0` 才算在窗內 |
| `sessionStorage` 被停用 | 無限重整，或既不重整也不提示 | 讀寫丟例外就只提示 |

真正的根本解還是部署端：`index.html` 不快取、舊 asset 保留一段時間。前端這段程式負責收尾：漏網的情況也能自己恢復，不必等使用者回報。
