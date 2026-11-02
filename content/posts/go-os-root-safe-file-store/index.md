---
title: "用 os.Root 寫一個安全的 Go 檔案儲存"
date: 2026-11-02T14:00:00+08:00
lastmod: 2026-11-02T14:00:00+08:00
draft: true
description: "用 Go 1.24 的 os.Root 擋住路徑跳脫與 symlink，再補上原子寫入和 FIFO 防護，做出一個不怕使用者亂給檔名的上傳儲存區。"
categories: [backend]
tags: [go, filesystem]
---

讓使用者上傳檔案、存在伺服器的某個目錄底下，是很常見的需求。最直覺的寫法只要兩行，但這兩行藏了三個問題：檔名可以跳出目錄、寫到一半當機會留下殘檔、讀檔時可能被一個 FIFO 卡住整個 goroutine。

這篇從最直覺的寫法開始，逐一修掉這三個問題。主角是 Go 1.24 新增的 `os.Root`。

<!--more-->

## 最直覺的寫法

```go
func Save(dir, name string, data []byte) error {
	return os.WriteFile(filepath.Join(dir, name), data, 0o600)
}

func Load(dir, name string) ([]byte, error) {
	return os.ReadFile(filepath.Join(dir, name))
}
```

`name` 來自使用者，問題就從這裡開始。

## 問題一：檔名跳出目錄

`filepath.Join("uploads", "../config.yaml")` 的結果是 `config.yaml`，已經在 `uploads` 外面了。這是最經典的 path traversal。

常見的補法是先檢查檔名：

```go
if strings.Contains(name, "..") {
	return errors.New("bad name")
}
```

但這只擋得住字串。如果 `uploads` 裡面已經有一個 symlink（可能是另一個功能建立的，也可能是攻擊者透過其他管道放的），檔名本身完全正常，`os.ReadFile` 還是會照著連結讀到目錄外面：

```go
// uploads/report -> ../secret.txt
os.ReadFile(filepath.Join("uploads", "report")) // 讀到 secret.txt
```

就算改成先用 `os.Lstat` 檢查是不是 symlink 再開檔，檢查和開檔之間還有時間差，這段空檔裡檔案可以被換掉。這就是 TOCTOU（time-of-check to time-of-use）。

### 解法：os.Root

`os.Root` 代表「只能在這個目錄底下操作」。在大部分平台上，`os.OpenRoot` 會對目錄開一個 file descriptor，之後的操作都以它為起點、一層一層解析路徑，只要有任何一段跑到根目錄外面就回傳錯誤，跟著 symlink 跳出去也一樣：

```go
root, err := os.OpenRoot("uploads")
if err != nil {
	return err
}
defer root.Close()

f, err := root.Open("report") // symlink 指向外面 → 回傳錯誤
```

幾個要知道的規則：

- 根目錄**裡面**的 symlink 可以正常使用，只是不能指到外面，也不能是絕對路徑。
- `root.Open("../x")` 直接失敗，不用自己檢查 `..`。
- 目錄被搬走時，Root 會跟著原本的目錄走，因為它握的是 descriptor 而不是路徑字串。

即使有了 Root，我還是會限制檔名只能是單一層、而且不能用 `.` 開頭。這不是為了安全（Root 已經擋住了），而是讓儲存區保持扁平，並把 `.` 開頭的名字保留給下一節的暫存檔：

```go
func checkName(name string) error {
	if name == "" || name != filepath.Base(name) || name[0] == '.' {
		return fmt.Errorf("%w: %q", ErrBadName, name)
	}
	return nil
}
```

## 問題二：寫到一半當機

`os.WriteFile` 會先把檔案截成 0 再寫入。如果在寫入途中程式崩潰或主機斷電，留下的就是一個只寫了一半的檔案，而且舊的內容已經沒了。

標準的做法是「先寫暫存檔，再改名」，因為同一個檔案系統裡的 rename 是原子操作：讀的人要嘛看到舊檔，要嘛看到完整的新檔。不過要真的做到斷電也不壞，順序很重要：

1. 用 `O_CREATE|O_EXCL` 開一個隨機名稱的暫存檔
2. 寫入資料
3. 對暫存檔 `Sync()`：確定資料已經寫到磁碟上
4. `Rename` 成正式檔名
5. 對**目錄** `Sync()`：確定「改名」這件事本身也寫到磁碟上

第 3 步如果省略，rename 可能比資料先落地，斷電後會得到一個名字正確、內容卻是空的檔案。第 5 步常常被忘記：目錄項目也是資料，沒有 sync 的話，重開機後可能發現 rename 根本沒發生。

第 1 步的 `O_EXCL` 也有用意：如果那個路徑已經存在（上次留下的殘檔，或是有人搶先放了一個 symlink），開檔會直接失敗，而不是把它截斷或跟著連結寫出去。

```go
func (s *Store) Save(name string, r io.Reader) error {
	if err := checkName(name); err != nil {
		return err
	}
	tmp, f, err := s.createTemp(name)
	if err != nil {
		return err
	}
	ok := false
	defer func() {
		if !ok {
			_ = f.Close()
			_ = s.root.Remove(tmp)
		}
	}()
	if _, err := io.Copy(f, r); err != nil {
		return err
	}
	if err := f.Sync(); err != nil {
		return err
	}
	if err := f.Close(); err != nil {
		return err
	}
	if err := s.root.Rename(tmp, name); err != nil {
		return err
	}
	ok = true
	return s.syncDir()
}

func (s *Store) createTemp(name string) (string, *os.File, error) {
	for range 5 {
		var b [8]byte
		_, _ = rand.Read(b[:]) // Go 1.24 起 crypto/rand.Read 不會回傳錯誤
		tmp := "." + name + "." + hex.EncodeToString(b[:]) + ".tmp"
		f, err := s.root.OpenFile(tmp, os.O_CREATE|os.O_EXCL|os.O_WRONLY, 0o600)
		if err == nil {
			return tmp, f, nil
		}
		if !errors.Is(err, os.ErrExist) {
			return "", nil, err
		}
	}
	return "", nil, errors.New("no unused temp name")
}

func (s *Store) syncDir() error {
	d, err := s.root.Open(".")
	if err != nil {
		return err
	}
	defer d.Close()
	return d.Sync()
}
```

如果 `Rename` 的目標是指向外面的 symlink，被取代的只有連結，外面的檔案不受影響。測試裡我也特別驗證了這一點。

## 問題三：讀到 FIFO 就卡住

Root 擋得住「跑到目錄外」，但官方文件也寫得很清楚：它**不會**阻止你存取 device file 這類特殊檔案。FIFO（named pipe）就是其中一種麻煩的：以唯讀模式開啟一個 FIFO 時，`open` 會一直等到有人從另一端開啟寫入為止。如果儲存區裡的檔案被換成 FIFO，每個來讀的 request 都會卡在 `open`，而且 `open` 不吃 context，沒有東西能把它叫醒。

直覺的修法是先 `Stat` 看看是不是一般檔案，但這又回到 TOCTOU：檢查完到真正開檔之間，檔案還是能被換掉。

正確的做法分兩步：

1. 用 `O_NONBLOCK` 開檔，讓 FIFO 的 `open` 立刻返回
2. 對**已經拿到的** descriptor 呼叫 `f.Stat()`，不是一般檔案就拒絕

`O_NONBLOCK` 本身不是防護，它只是讓 `open` 不會卡住；真正擋下 FIFO 的是第 2 步。因為檢查的是手上這個 descriptor，中間沒有空檔可以讓人換檔案。

```go
func (s *Store) Open(name string) (*os.File, error) {
	if err := checkName(name); err != nil {
		return nil, err
	}
	f, err := s.root.OpenFile(name, os.O_RDONLY|syscall.O_NONBLOCK, 0)
	if err != nil {
		return nil, err
	}
	st, err := f.Stat()
	if err != nil {
		f.Close()
		return nil, err
	}
	if !st.Mode().IsRegular() {
		f.Close()
		return nil, fmt.Errorf("%w: %s is %v", ErrNotRegular, name, st.Mode().Type())
	}
	return f, nil
}
```

對一般檔案來說，`O_NONBLOCK` 不影響讀取，所以回傳的檔案照常使用即可。

## 完整的 Store

把三段組起來，加上建構函式：

```go
//go:build unix

package rootstore

var (
	ErrBadName    = errors.New("invalid file name")
	ErrNotRegular = errors.New("not a regular file")
)

type Store struct {
	root *os.Root
}

func Open(dir string) (*Store, error) {
	if err := os.MkdirAll(dir, 0o700); err != nil {
		return nil, err
	}
	root, err := os.OpenRoot(dir)
	if err != nil {
		return nil, err
	}
	return &Store{root: root}, nil
}

func (s *Store) Close() error { return s.root.Close() }

// checkName、Save、createTemp、syncDir、Open 如上
```

我替它寫了幾個測試，在 macOS 上用 Go 1.25 跑過，涵蓋：

- 第二次 `Save` 會取代舊檔，而且目錄裡不會留下暫存檔
- `../x`、`a/b`、`.hidden`、`/etc/passwd` 都被 `checkName` 擋下
- 儲存區裡放絕對路徑和相對路徑的 symlink，`Open` 都會失敗；對照組用 `os.ReadFile` 則會讀到外面的檔案
- 對 symlink 名稱 `Save`，外面的檔案內容不變
- 對 FIFO 呼叫 `Open` 立刻回傳 `ErrNotRegular`；對照組不加 `O_NONBLOCK` 則會卡住

FIFO 的測試大概長這樣，重點是用 `select` 加 timeout，讓「卡住」變成一個會失敗的測試，而不是讓整個測試程式掛著：

```go
func TestFIFODoesNotBlock(t *testing.T) {
	s, _ := newStore(t)
	if err := syscall.Mkfifo(filepath.Join(s.root.Name(), "pipe"), 0o600); err != nil {
		t.Skip("mkfifo:", err)
	}
	done := make(chan error, 1)
	go func() { _, err := s.Open("pipe"); done <- err }()
	select {
	case err := <-done:
		if !errors.Is(err, ErrNotRegular) {
			t.Fatalf("got %v, want ErrNotRegular", err)
		}
	case <-time.After(2 * time.Second):
		t.Fatal("Open blocked on a FIFO")
	}
}
```

## 限制與版本

- **Go 版本**：`os.Root` 和 `OpenRoot`、`Open`、`OpenFile`、`Remove` 等基本方法是 Go 1.24 加入的；這篇用到的 `Root.Rename`（以及 `MkdirAll`、`WriteFile`、`ReadFile` 等）是 Go 1.25 才有。
- **只適用 Unix**：`syscall.O_NONBLOCK` 在 Windows 不存在，目錄的 fsync 在 Windows 上的行為也和 Unix 不同，所以範例加了 `//go:build unix`。Windows 要另外處理。
- **Root 管不到的事**：官方文件列出 Root 不會阻止跨越檔案系統邊界、Linux bind mount、`/proc` 特殊檔案和 device file。這也是第三節要自己檢查 `IsRegular()` 的原因。
- **Unix 上的 Chmod／Chown／Chtimes** 仍有競態：操作途中檔案被換成 symlink 時，可能改到連結本身。這篇沒用到，但用的時候要留意。
- **檔名限制是政策，不是安全機制**：真正防止跳脫的是 Root。`checkName` 是為了讓儲存區扁平，並避免和暫存檔撞名。

## 小結

| 問題 | 直覺寫法 | 改成 |
|---|---|---|
| 檔名跳出目錄、symlink 逃逸 | `filepath.Join` ＋ 字串檢查 | `os.OpenRoot` |
| 寫到一半當機 | `os.WriteFile` | `O_EXCL` 暫存檔 → `Sync` → `Rename` → 目錄 `Sync` |
| FIFO 卡住讀取 | 先 `Stat` 再開檔 | `O_NONBLOCK` 開檔 → 對 fd `Stat` → 檢查 `IsRegular` |

三個修法都只用到標準函式庫。共通的原則是：**檢查要做在手上已經握著的東西上**，例如 Root 握著的目錄 descriptor，或剛開好的檔案。不要檢查一個之後還會再解析一次的路徑字串。
