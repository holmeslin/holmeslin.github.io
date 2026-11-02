---
title: "url.URL.Redacted() 不夠用：連線字串裡的 token 會原樣進到 log"
date: 2026-11-02T13:00:00+08:00
lastmod: 2026-11-02T13:00:00+08:00
draft: true
description: "Go 的 Redacted() 只遮密碼。token 放在 username、query 或 fragment 時都會原樣印出來，url.Parse 的錯誤訊息也會。這篇整理哪些情況會漏，以及一個只輸出 scheme 和 host 的寫法。"
categories: [backend]
tags: [go, logging]
---

服務啟動時把連線字串印到 log，方便確認連到哪台機器，這是很常見的寫法。Go 的 `net/url` 有一個 `Redacted()`，看名字會以為它能把敏感資訊遮掉，但它只遮「密碼」這一格。很多連線字串的祕密根本不放在那一格。

這篇用實際跑過的測試，整理 `Redacted()` 會漏掉的幾種情況，以及 `url.Parse` 錯誤訊息的同類問題，最後給一個簡單的替代寫法。

<!--more-->

## Redacted() 到底遮了什麼

官方文件寫得很直接：

> Redacted is like URL.String but replaces any password with "xxxxx". Only the password in u.User is redacted.

也就是說，只有 `user:password@host` 裡冒號後面那段會被換成 `xxxxx`。其他部分原封不動。

我用 Go 1.25 跑了四種連線字串，祕密都是 `s3cr3t`：

| 原始字串 | `Redacted()` 的結果 |
|---|---|
| `redis://user:s3cr3t@cache:6379/0` | `redis://user:xxxxx@cache:6379/0` |
| `nats://s3cr3t@broker:4222` | `nats://s3cr3t@broker:4222` |
| `redis://cache:6379/0?password=s3cr3t` | `redis://cache:6379/0?password=s3cr3t` |
| `https://api.example.com/hook#token=s3cr3t` | `https://api.example.com/hook#token=s3cr3t` |

只有第一種被遮掉。另外三種都是真實會遇到的寫法：

- **token 放在 username**：NATS 支援 `nats://<token>@host` 這種寫法。nats.go 在 URL 只有 username、沒有密碼時，會把 username 當成 auth token（原始碼註解是 `if no password, assume username is authToken`）。對 `Redacted()` 來說這只是一個使用者名稱，所以照印。
- **密碼放在 query**：Redis URI 的規格（IANA 登記的 `redis` scheme）寫明，AUTH 用的密碼可以來自 userinfo 的密碼欄，也可以來自 query 參數 `password`。`Redacted()` 完全不看 query。
- **token 放在 path 或 fragment**：有些 webhook 或預先簽好的 URL 把 token 放在路徑或 `#` 後面，同樣不會被處理。

## 另一個漏洞：url.Parse 的錯誤訊息

就算你已經小心處理了成功的情況，失敗的情況還是會漏。`url.Parse` 回傳的是 `*url.Error`，它的錯誤訊息會把原始字串整段引用出來：

```go
raw := "nats://s3cr3t@broker:42x22" // port 打錯
_, err := url.Parse(raw)
return fmt.Errorf("connect broker: %w", err)
```

實際印出來是：

```text
connect broker: parse "nats://s3cr3t@broker:42x22": invalid port ":42x22" after host
```

設定檔打錯一個字，token 就跟著錯誤訊息進了 log。用 `%w` 包裝本身沒錯，問題是被包裝的那個錯誤已經帶著祕密。如果你用的 client library 內部也呼叫 `url.Parse`，並把錯誤原樣往上傳，就算你自己沒有 parse，拿到的錯誤也會是同一個樣子。記錄 driver 的錯誤前，先確認它的內容。

## 解法：只輸出 scheme 和 host

與其想辦法把每一種藏祕密的位置都遮掉，不如反過來：log 只需要知道「連到哪裡」，那就只輸出 scheme 和 host，其他全部丟掉。

```go
// ForLog renders a connection URL for logs: scheme and host only.
// ok is false when raw does not parse; the caller must then log nothing of it.
func ForLog(raw string) (safe string, ok bool) {
	u, err := url.Parse(raw)
	if err != nil || u.Host == "" {
		return "", false
	}
	return (&url.URL{Scheme: u.Scheme, Host: u.Host}).String(), true
}
```

同樣四個字串的結果：

| 原始字串 | `ForLog()` 的結果 |
|---|---|
| `redis://user:s3cr3t@cache:6379/0` | `redis://cache:6379` |
| `nats://s3cr3t@broker:4222` | `nats://broker:4222` |
| `redis://cache:6379/0?password=s3cr3t` | `redis://cache:6379` |
| `https://api.example.com/hook#token=s3cr3t` | `https://api.example.com` |

這是白名單的思路：只挑確定安全的欄位重新組出來，而不是從原字串裡刪掉已知危險的欄位。之後就算 token 換了地方放，這個函式也不用改。

兩個刻意的設計：

- **parse 失敗時回傳空字串和 `false`**，而不是回傳原字串或錯誤。呼叫端拿到 `ok == false` 時，應該直接拒絕連線，並且只記一句「連線字串格式錯誤」。這樣錯誤路徑上就沒有任何地方會碰到原始字串。
- **`u.Host == ""` 也算失敗**。像 `s3cr3t@broker` 這種漏了 scheme 的字串，`url.Parse` 不會報錯，但整段會被當成 path，`Host` 是空的。這種情況輸出 `ForLog` 結果沒有意義，不如一起當成格式錯誤。

實際用起來：

```go
safe, ok := ForLog(cfg.BrokerURL)
if !ok {
	return errors.New("broker URL is malformed") // 不要附上 cfg.BrokerURL，也不要 %w 包 url.Error
}
logger.Info("connecting", "broker", safe)
```

## 用測試鎖住

這種「不能漏」的需求很適合寫成測試，而且要包含**對照組**，證明 `Redacted()` 真的會漏、`url.Parse` 的錯誤真的會漏。否則哪天有人覺得 `ForLog` 多此一舉、改回 `Redacted()`，測試也不會提醒他：

```go
func TestRedactedMasksPasswordOnly(t *testing.T) {
	cases := map[string]bool{ // raw -> does Redacted still leak the secret?
		"redis://user:" + secret + "@cache:6379/0":     false,
		"nats://" + secret + "@broker:4222":            true,
		"redis://cache:6379/0?password=" + secret:      true,
		"https://api.example.com/hook#token=" + secret: true,
	}
	for raw, leaks := range cases {
		got := mustParse(t, raw).Redacted()
		if strings.Contains(got, secret) != leaks {
			t.Errorf("Redacted(%q) = %q, want leak=%v", raw, got, leaks)
		}
	}
}

func TestParseErrorLeaks(t *testing.T) {
	raw := "nats://" + secret + "@broker:42x22"
	_, err := url.Parse(raw)
	wrapped := fmt.Errorf("connect broker: %w", err)
	if !strings.Contains(wrapped.Error(), secret) {
		t.Fatalf("expected the wrapped error to leak, got %q", wrapped)
	}
	if got, ok := ForLog(raw); ok || got != "" {
		t.Fatalf("ForLog = %q, %v; want \"\", false", got, ok)
	}
}
```

`ForLog` 本身的測試則是反過來，對同一批字串斷言輸出裡找不到 `secret`。

## 限制

- **這只是給人看的字串**：`ForLog` 不保留 path 和 query。要拿來重新連線，請用原始設定，不要用它的輸出。
- **逗號分隔的多台主機**：有些 client 接受 `nats://a:4222,nats://b:4222` 這種清單。整串丟進 `url.Parse` 不會報錯，但 `Host` 會變成 `a:4222,nats:`，`ForLog` 也就印出一個看不懂的結果。要先用逗號切開，再逐段處理。
- **host 本身也可能是祕密**：內部主機名稱若不想出現在 log，這個函式幫不上忙，那是另一個層次的決定。
- **其他 log 來源**：driver 自己印的 debug log、panic 時的 stack trace，都不經過你的 `ForLog`。最保險的做法，還是讓 token 根本不進 URL（例如 nats.go 的 `nats.Token()`、Redis client 的密碼欄位），URL 只放 host。

## 小結

| 情況 | `Redacted()` | `ForLog()` |
|---|---|---|
| `user:password@` | 遮掉 | 移除 |
| `token@`（只有 username） | **照印** | 移除 |
| `?password=` | **照印** | 移除 |
| `#token=` | **照印** | 移除 |
| parse 失敗 | 不適用，`*url.Error` 會引用原字串 | 回傳 `false`，呼叫端什麼都不印 |

`Redacted()` 的名字很容易讓人以為它處理了所有敏感資訊，但它只負責一格。要放心地把連線字串寫進 log，與其列舉要遮哪些地方，不如只挑確定安全的部分重新組出來。
