---
title: "檢查 ?redirect= 防 open redirect"
date: 2026-11-02T10:00:00+08:00
lastmod: 2026-11-02T10:00:00+08:00
draft: false
description: "SPA 登入頁常用 ?redirect= 把使用者帶回原本的頁面。只檢查開頭是不是 / 擋不住反斜線、tab 和編碼過的路徑，這篇用四層檢查加上 new URL 比對 origin，把它寫成一個 TypeScript type guard。"
categories: [security]
tags: [typescript, web]
---

使用者在未登入時打開 `/orders/42`，前端把他導到 `/login?redirect=/orders/42`，登入成功後再導回去。這個流程幾乎每個 SPA 都有，但 `redirect` 是網址上的參數，誰都能改。只要改成 `?redirect=https://evil.com`，使用者就會在剛輸入完帳號密碼、最信任你的那一刻，被送到攻擊者的頁面。這就是 open redirect。

這篇從最常見的檢查開始，說明它怎麼被繞過，再一層一層補起來。

<!--more-->

## 最常見的檢查

```ts
function isSafeRedirectNaive(p: string): boolean {
  return p.startsWith('/') && !p.startsWith('//')
}
```

開頭必須是 `/`，所以 `https://evil.com` 和 `javascript:alert(1)` 進不來；不能是 `//`，所以 protocol-relative 的 `//evil.com` 也進不來。看起來很完整，但下面三個值都會通過：

| 輸入 | 瀏覽器實際解析成 |
|---|---|
| `/\evil.com` | `https://evil.com/` |
| `/` + tab + `/evil.com` | `https://evil.com/` |
| `/%2F%2Fevil.com` | 本站的 `/%2F%2Fevil.com`，但 decode 一次就是 `///evil.com` |

前兩個是 URL 解析規則造成的。瀏覽器和 Node 的 `new URL` 都依循 WHATWG URL 標準，標準裡規定：

- 解析前會先「移除所有 ASCII tab 和換行」，所以 `/\t/evil.com` 進到解析器時已經是 `//evil.com`。
- 對 `http`、`https` 這類 special scheme，反斜線 `\` 會被當成 `/` 處理，所以 `/\evil.com` 也等於 `//evil.com`。

字串檢查看到的跟瀏覽器解析的不是同一個東西，這就是繞過的來源。

第三個比較特別：`new URL('/%2F%2Fevil.com', origin)` 本身是安全的，`%2F` 不會被當成斜線。問題在於只要後面任何一層（後端的轉址程式、你自己寫的工具函式）會先 decode 再使用，它就變成 `///evil.com`，而這同樣會被解析成 `evil.com`。你很難保證整條鏈上沒有人會 decode。

## 四層檢查

```ts
export function isSafeRedirect(p: unknown, origin: string): p is string {
  if (typeof p !== 'string' || p === '') return false
  // 1. 必須是路徑：一個斜線開頭，不能兩個
  if (!p.startsWith('/') || p.startsWith('//')) return false
  // 2. 不能有 scheme 分隔符號，也不能有反斜線
  if (p.includes(':') || p.includes('\\')) return false
  // 3. decode 一次再檢查，防止後面有人再 decode
  let decoded: string
  try {
    decoded = decodeURIComponent(p)
  } catch {
    return false
  }
  if (decoded.startsWith('//') || decoded.includes('\\')) return false
  // 4. 最後一道：用自己的 origin 解析，結果必須還是自己的 origin
  try {
    return new URL(p, origin).origin === origin
  } catch {
    return false
  }
}
```

每一層各自都能被繞過，疊起來才可靠：

1. **開頭是單一 `/`**：擋掉完整網址和 `//evil.com`。
2. **不含 `:` 和 `\`**：`:` 擋掉任何 scheme，`\` 擋掉反斜線的變形。代價是合法路徑裡也不能有冒號，例如 `/meetings/10:30`。登入後的導回頁面通常不需要冒號，我寧可保守一點。
3. **decode 後再檢查一次**：擋掉 `/%2F%2Fevil.com`、`/%5Cevil.com`（`%5C` 是反斜線）這類編碼過的變形。編碼本身就有錯（例如 `/%E0%A4%A`）時，`decodeURIComponent` 會丟出 `URIError`，一樣視為不安全。
4. **`new URL(p, origin).origin === origin`**：這是真正的安全網。前三層是字串檢查，作用是把明顯的壞值提早擋掉，最後交給跟瀏覽器同一套解析規則判斷「導過去之後還在不在本站」。tab、換行這類前面沒有特別處理的情況，在這一層也會被抓到。

`origin` 寫成參數而不是直接讀 `window.location.origin`，是為了讓它可以在 Node 裡測試。在瀏覽器裡呼叫時傳入 `window.location.origin` 即可。

## 用 type guard 收窄型別

回傳型別寫成 `p is string`，呼叫端就能把 `unknown` 直接收窄成 `string`，不需要 `as string`：

```ts
export function redirectTarget(raw: unknown, origin: string): string {
  return isSafeRedirect(raw, origin) ? raw : '/'
}

// 登入成功後
const target = redirectTarget(route.query.redirect, window.location.origin)
router.replace(target)
```

參數型別用 `unknown` 是故意的。以 vue-router 為例，`route.query.redirect` 的型別可能是字串、`null`，或是 `?redirect=a&redirect=b` 時的陣列。與其讓呼叫端各自轉型，不如讓檢查函式自己處理「不是字串」的情況。檢查不通過時一律導回首頁，不要嘗試「修正」使用者給的值。

## 測試

我用 Node 24 內建的 `node --test` 直接跑 `.ts` 檔（Node 24 預設就會移除型別標註），涵蓋：

- 合法路徑：`/`、`/orders/42?tab=items#top`、`/search?q=a%20b`、`/a/../b`
- 要擋下的值：`https://evil.com`、`//evil.com`、`///evil.com`、`/\evil.com`、`\\evil.com`、含 tab 或換行的 `//evil.com`、`javascript:alert(1)`、`/%2F%2Fevil.com`、`/%5Cevil.com`、編碼錯誤的 `/%E0%A4%A`、空字串、沒有 `/` 開頭的 `orders`
- 非字串：`undefined`、`null`、數字、陣列、物件
- 對照組：最常見的檢查真的會放行 `/\evil.com`、含 tab 的路徑和 `/%2F%2Fevil.com`，而且前兩者用 `new URL` 解析後確實會跑到 `evil.com`

對照組很重要。少了它，測試只證明了「新寫法擋得住」，沒有證明「舊寫法真的有洞」，讀者也不會知道每一層到底在防什麼。

```ts
test('the naive check lets these through', () => {
  for (const p of ['/\\evil.com', '/\t/evil.com', '/%2F%2Fevil.com']) {
    assert.equal(isSafeRedirectNaive(p), true, JSON.stringify(p))
  }
  assert.equal(new URL('/\\evil.com', origin).origin, 'https://evil.com')
  assert.equal(new URL('/\t/evil.com', origin).origin, 'https://evil.com')
  assert.equal(decodeURIComponent('/%2F%2Fevil.com'), '///evil.com')
})
```

## 限制

- **前端檢查不能取代後端**：如果轉址是由後端回 `302` 完成的，後端也要做同樣的檢查。這篇只處理 SPA 自己用 router 導頁的情況。
- **只允許同源**：如果需要導到其他自家網域，改成比對一份明確的 origin 白名單，不要用 `endsWith('.example.com')` 這類字串比對。
- **冒號限制比較嚴格**：如果你的路由真的會用到冒號，可以拿掉第二層的 `:` 檢查，靠第一層和第四層擋 scheme。但要自己補測試確認。

## 小結

| 繞過手法 | 擋在哪一層 |
|---|---|
| `https://evil.com`、`javascript:` | 1、2 |
| `//evil.com` | 1 |
| `/\evil.com` | 2 |
| `/%2F%2Fevil.com`、`/%5Cevil.com` | 3 |
| 含 tab、換行的路徑 | 4 |

字串檢查擋的是你想得到的寫法，`new URL` 比對 origin 擋的是瀏覽器實際會怎麼解析。兩者都要有：前者讓明顯的壞值提早出局，後者才是最後的判斷依據。
