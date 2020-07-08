---
title: "Go 語言 sync 包的應用詳解"
date: 2020-07-07T15:07:16+08:00
tags:
    - "golang"
categories:
    - "紀錄"
    - “學習”
description:
author: "holmes.lin"
contentCopyright: ''
menu: 
banner: "/banners/go-banner.jpg"
images:
    - ""
---

並發程式中的同步也就是我們通常說的鎖的主要作用是保證多個線程或者 `goroutine` 在訪問同一片內存時不會出現混亂的問題。

`Go` 語言的 `sync` 包提供了常見的並發程式同步 。 今天的文章裡讓我們回到應用層 ， 聚焦 `sync` 包裡這些同步的應用場景 ， 同時也會介紹 `sync` 包中的 `Pool` 和 `Map` 的應用場景和使用方法。

<!--more-->

## 前言

並發程式中的同步也就是我們通常說的鎖的主要作用是保證多個線程或者 `goroutine` 在訪問同一片內存時不會出現混亂的問題。

`Go` 語言的 `sync` 包提供了常見的並發程式同步 。 今天的文章裡讓我們回到應用層 ， 聚焦 `sync` 包裡這些同步的應用場景 ， 同時也會介紹 `sync` 包中的 `Pool` 和 `Map` 的應用場景和使用方法。

## sync.Mutex

`sync.Mutex` 可能是 `sync` 包中最常使用的 。 它允許在共享資源上互斥訪問 ( 不能同時訪問 ) :

```go
mutex := &sync.Mutex{}

mutex.Lock()
// Update 共享變數(比如切片，結構體指針等)
mutex.Unlock()
```

要注意的是，在第一次被使用後，不能再對 `sync.Mutex` 進行複製 。 ( sync 包的所有函式都一樣 ) 。

如果結構體具有同步程式字段，則必須通過指針傳遞它。

## sync.RWMutex

`sync.RWMutex` 是一個讀寫互斥鎖 ， 它提供了我們上面的剛剛看到的 `sync.Mutex` 的 `Lock` 和 `UnLock` 方法（因為這兩個結構都實現了 `sync.Locker` 接口）。

但是，它還允許使用 `RLock` 和 `RUnlock` 方法進行並發讀取：

```
mutex := &sync.RWMutex{}

mutex.Lock()
// Update 共享變數
mutex.Unlock()

mutex.RLock()
// Read 共享變數
mutex.RUnlock()
```

`sync.RWMutex` 允許至少一個讀鎖或一個寫鎖存在，而 `sync.Mutex` 允許一個讀鎖或一個寫鎖存在。

通過基準測試來比較這幾個方法的性能：

```shell
BenchmarkMutexLock-4       83497579         17.7 ns/op
BenchmarkRWMutexLock-4     35286374         44.3 ns/op
BenchmarkRWMutexRLock-4    89403342         15.3 ns/op
```

可以看到鎖定/解鎖 `sync.RWMutex` 讀鎖的速度比鎖定/解鎖 `sync.Mutex` 更快，另一方面，在 `sync.RWMutex` 上調用 Lock()/ Unlock() 是最慢的操作。

因此，只有在頻繁讀取和不頻繁寫入的場景裡，才應該使用 `sync.RWMutex`。

## sync.WaitGroup

`sync.WaitGroup` 也是一個經常會用到的函式，它的使用場景是在一個 `goroutine` 等待一組 `goroutine` 執行完成。

`sync.WaitGroup` 擁有一個內部計數器 。 當計數器等於 0 時，則 `Wait()` 方法會立即返回 。 否則它將阻塞執行 `Wait()` 方法的 `goroutine` 直到計數器等於 0 時為止。

要增加計數器，我們必須使用 `Add(int)` 方法 。 要減少它 ， 我們可以使用 `Done()`（ 將計數器減 1 ），也可以傳遞負數給 `Add` 方法把計數器減少指定大小 ， `Done()` 方法底層就是通過 `Add(-1)`實現的。

在以下範例中，我們將啟動八個 `goroutine`，並等待他們完成：

```go
wg := &sync.WaitGroup{}

for i := 0; i < 8; i++ {
  wg.Add(1)
  go func() {
    // Do something
    wg.Done()
  }()
}

wg.Wait()
// blabla...
```

每次創建 `goroutine` 時，我們都會使用 `wg.Add(1)` 來增加 `wg` 的內部計數器。

我們也可以在 for 循環之前調用 `wg.Add(8)`。

與此同時 ， 每個 `goroutine` 完成時 ， 都會使用 `wg.Done()` 減少 `wg` 的內部計數器。

`main goroutine` 會在八個 `goroutine` 都執行 `wg.Done()` 將計數器變為 `0` 後才能繼續執行。

## sync.Map

`sync.Map` 是一個並發版本的 `Go` 的 `map` ， 我們可以 :

- 使用 `Store(interface {}，interface {})` 增加元素。

- 使用 `Load(interface {}) interface {}` 檢索元素。

- 使用 `Delete(interface {})` 刪除元素。

- 使用 `LoadOrStore(interface {}，interface {}) (interface {}，bool)` 檢索或添加之前不存在的元素 。

  如果 `key` 之前在 `map` 中存在，則返回 `true`。

- 使用 `Range` 遍歷元素。

```go
m := &sync.Map{}

// 增加元素
m.Store(1, "one")
m.Store(2, "two")

// 獲取元素1
value, contains := m.Load(1)
if contains {
  fmt.Printf("%s\n", value.(string))
}

// 返回已存 value ， 否則把指定的鍵值儲存到 map 中
value, loaded := m.LoadOrStore(3, "three")
if !loaded {
  fmt.Printf("%s\n", value.(string))
}

m.Delete(3)

// 遍歷所有元素
m.Range(func(key, value interface{}) bool {
  fmt.Printf("%d: %s\n", key.(int), value.(string))
  return true
})
```

上面會輸出：

```shell
one
three
1: one
2: two
```

如你所見 ， `Range` 方法接收一個類型為`func(key，value interface {})bool` 的函式跟參數。

如果函數返回了 `false` ， 則停止迭迴。

有趣的是 ， 即使我們在一定的時間後返回 `false` ， 最壞情況下的時間複雜度仍為 `O(n)`。

我們應該在什麼時候使用 `sync.Map` 而不是在普通的 map 上使用 `sync.Mutex`？

- 當我們對 map 有頻繁的讀取和不頻繁的寫入時。

- 當多個 `goroutine` 讀取，寫入和覆蓋不相交的 `key` 時。

這是什麼意思呢 ？

例如 ， 如果我們有一個分片實現 ， 其中包含一組 4 個 `goroutine`，每個 `goroutine` 負責 25％ 的鍵（每個負責的鍵不衝突）。

在這種情況下，`sync.Map` 是首選。

## sync.Pool

sync.Pool 是一個並發池 ， 負責安全地保存一組對象 。

它有兩個導出方法：

- `Get() interface{}` 用來從並發池中取出元素。

- `Put(interface{})` 將一個對象加入並發池。

```go
pool := &sync.Pool{}

pool.Put(NewConnection(1))
pool.Put(NewConnection(2))
pool.Put(NewConnection(3))

connection := pool.Get().(*Connection)
fmt.Printf("%d\n", connection.id)
connection = pool.Get().(*Connection)
fmt.Printf("%d\n", connection.id)
connection = pool.Get().(*Connection)
fmt.Printf("%d\n", connection.id)
```

输出：

```shell
1
3
2
```

需要注意的是 Get()方法会从并发池中随机取出对象，无法保证以固定的顺序获取并发池中存储的对象。

還可以為 sync.Pool 指定一個創建者方法：

```go
pool := &sync.Pool{
  New: func() interface{} {
    return NewConnection()
  },
}

connection := pool.Get().(*Connection)
```

每次調用 `Get()` 時 ， 將返回由在 `pool.New` 中指定的函數創建的對象（在本例中為指針）。

那麼什麼時候使用 `sync.Pool`？

有兩個使用案例：

第一個是當我們必須重用共享的和長期存在的對象（例如，數據庫連接）時。

第二個是用於優化內存分配。

讓我們考慮一個寫入緩衝區並將結果持久保存到文件中的函數示例。

使用 `sync.Pool` ，我們可以通過在不同的函數調用之間重用同一對象來重用為緩衝區分配的空間。

第一步是檢索先前分配的緩衝區（如果是第一個調用，則創建一個緩衝區，但這是抽象的）。

然後，`defer` 操作是將緩衝區放回 `sync.Pool`中。

```go

func writeFile(pool *sync.Pool, filename string) error {
    buf := pool.Get().(*bytes.Buffer)

    defer pool.Put(buf)

    // Reset 緩存區，不然會連接上次調用時保存在緩存區裡的字符串foo
    // 編程foofoo 以此類推
    buf.Reset()

    buf.WriteString("foo")
    return ioutil.WriteFile(filename, buf.Bytes(), 0644)
}

```

## sync.Once

sync.Once 是一個簡單而強大的函式，可確保一個函數僅執行一次。

在下面的示例中，只有一個 goroutine 會顯示輸出消息：

```go
once := &sync.Once{}
for i := 0; i < 4; i++ {
    i := i
    go func() {
        once.Do(func() {
            fmt.Printf("first %d\n", i)
        })
    }()
}
```

我們使用了 Do(func())方法來指定只能被調用一次的部分。

## sync.Cond

`sync.Cond` 可能是 `sync` 包提供的函式中最不常用的一個 ， 它用於發出信號（一對一）或廣播信號（一對多）到 `goroutine`。

讓我們考慮一個場景，我們必須向一個 `goroutine` 指示共享切片的第一個元素已更新 。

創建 `sync.Cond` 需要 `sync.Locker` 對象（`sync.Mutex` 或 `sync.RWMutex`）：

```
cond := sync.NewCond(&sync.Mutex{})
```

然後，讓我們編寫負責顯示切片的第一個元素的函數：

```go
func printFirstElement(s []int, cond \*sync.Cond) {
    cond.L.Lock()
    cond.Wait()
    fmt.Printf("%d\n", s[0])
    cond.L.Unlock()
}
```

我們可以使用 `cond.L` 訪問內部的互斥鎖。

一旦獲得了鎖，我們將調用 `cond.Wait()` ， 這會讓當前 `goroutine` 在收到信號前一直處於阻塞狀態 。

讓我們回到 `main goroutine`。

我們將通過傳遞共享切片和先前創建的 `sync.Cond` 來創建 `printFirstElement` 池 。

然後我們調用 `get()` 函數，將結果存儲在 `s[0]` 中並發出信號：

```go
s := make([]int, 1)
for i := 0; i < runtime.NumCPU(); i++ {
    o printFirstElement(s, cond)
}

i := get()
cond.L.Lock()
s[0] = i
cond.Signal()
cond.L.Unlock()
```

這個信號會解除一個 `goroutine` 的阻塞狀態 ， 解除阻塞的 `goroutine` 將會顯示 s[0]中存儲的值 。

但是，有的人可能會爭辯說我們的代碼破壞了 `Go` 的最基本原則之一：

> 不要通过共享内存进行通信；而是通过通信共享内存。

確實 ， 在這個示例中，最好使用 `channel` 來傳遞 `get()` 返回的值 。

但是我們也提到了 `sync.Cond` 也可以用於廣播信號 。

我們修改一下上面的示例，把 `Signal()` 調用改為調用 `Broadcast()`。

```go
i := get()
cond.L.Lock()
s[0] = i
cond.Broadcast()
cond.L.Unlock()
```

這種情況下，所有 `goroutine` 都將被觸發。

眾所周知，channel 裡的元素只會由一個 `goroutine`接收到。通過 `channel` 模擬廣播的唯一方法是關閉 `channel`。

當一個 `channel` 被關閉後 ， `channel` 中已經發送的數據都被成功接收後 ， 後續的接收操作將不再阻塞 ， 它們會立即返回一個零值 。

但是這種方式只能廣播一次。因此，儘管存在很大爭議，但這無疑是 `sync.Cond` 的一個有趣的功能 。