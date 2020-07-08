---
title: "Goroutine 和 Channel 講解"
date: 2020-07-08T10:36:17+08:00
lastmod: 2020-07-08T10:36:17+08:00
draft: false
keywords: []
description: ""
tags: ["golang"]
categories: ["紀錄","“學習”"]
author: "holmees.lin"
---

## 前言

Go 語言中並發程序可以用兩種方式來實現。

一種是 `goroutine` 和 `channel`，其支持 "交談循序程式"（communicating sequential processes）或被簡稱為 CSP。

CSP 是一個現代的並發程式模型，在這種程式模型中值會在不同的運行實例(goroutine)中傳遞，儘管大多數情況下被限制在單一實例中。

另一種是傳統的並發模型，多線程共享內存(基於共享變量的並發) ， 會在後續單獨闡述。

<!--more-->

## goroutine

`goroutine` 是一個輕量級的線程 ， 它與線程的區別是線程是操作系統中對於一個獨立運行實例的描述，不同的操作系統中，線程的實現也不盡相同；

對於 goroutine，操作系統並不知道它的存在 ， `goroutine` 的調度是 Go 語言的運行時進行管理的。

啟動線程雖然比進程使用的資源少 ， 但依然需要上下文切換等大量工作 ， Go 語言有自己的調度器 ， 許多 `goroutine` 的數據都是共享的 ， 因此 `goroutine` 之間的切換會快很多 ， 啟動 `goroutine` 所耗費的資源也很少。

```go
package main

import (
    "fmt"
    "strconv"
)

func Info(name string)  {
    for i := 0; i < 3; i++ {
        fmt.Println(name + ":" + strconv.Itoa(i))
    }
}

func main()  {
    Info("info")

    go Info("goroutine1")

    go func(name string) {
        fmt.Println(name)
    }("goroutine2")

    var input string
    fmt.Scanln(&input)
    fmt.Println("done")
}

```

## channel

`channel` 被稱為通道 ， 是連接並發 `goroutine` 的管道 ， 可以從一個 `goroutine` 向通道發送值 ， 並在另一個 `goroutine` 中接收到這些值 。

每個 `channel` 都有一個特殊的類型 ， 也就是 `channel` 可發送數據的類型 。 一個可以發送 `int` 類型數據的 `channel` 一般寫為 `chan int`。

一個 `channel` 有發送和接收兩個主要操作 ， 都是通信行為 。

一個發送語句將一個值從一個 `goroutine` 通過 `channel` 發送到另一個執行接收操作的 goroutine 。

發送和接收兩個操作都是用 `<-` 運算符 。

在發送語句中 ， `<-` 運算符分割 `channel` 和要發送的值 ； 在接收語句中 ， `<-` 運算符寫在 `channel` 對象之前 ， 一個不使用接收結果的接收操作也是合法的。

`channel` 的發送操作將導致發送者 `goroutine` 阻塞 ， 直到另一個 `goroutine` 在相同的 `channel` 上執行接收操作 ， 當發送的值通過 `channel` 成功傳輸之後 ， 兩個 `goroutine` 可以繼續執行後面的語句 。

反之 ， 如果接收操作先發生 ， 那麼接收者 `goroutine` 也將阻塞 ， 直到有另一個 `goroutine` 在相同的 `Channels` 上執行發送操作。

```go
package main

import (
    "fmt"
    "strconv"
)

func Info(i int, ch chan string)  {
    msg := "訊息" + strconv.Itoa(i)
    ch <- msg
}

func main()  {
    chs := make([]chan string, 3)
    for i := 0; i < 3; i++ {
        chs[i] = make(chan string)
        go Info(i, chs[i])
    }

    for num, ch := range chs {
        msg := <- ch
        fmt.Println(num, msg)
    }

    fmt.Println("Done")
}

```

`channel` 還支持 `close` 操作 ， 用於關閉 `channel` ， 隨後對基於該 `channel` 的任何發送操作都將導致 `Panic` 異常 。

對一個已經被 `close` 過的 `channel` 接收操作依然可以接受到之前已經成功發送的數據 ； 如果 `channel` 中已經沒有數據的話將產生一個零值(nil)的數據 。

```go
close(ch)
```

### 帶緩存的 channel

帶緩存的 `channel` 內部有一個元素隊列 。 隊列的最大容量是在調用 `make` 函數創建 `channel` 時通過第二個參數指定的 。
下面的語法創建了一個可以持有三個字符串元素的帶緩存 `channel` 。

```go
ch = make(chan string, 3)
```

向緩存 `channel` 的發送操作就是向內部緩存隊列的尾部插入元素 ， 接收操作則是從隊列的頭部刪除元素 。

如果內部緩存隊列是滿的 ， 那麼發送操作將阻塞直到另一個 `goroutine` 執行接收操作而釋放了新的隊列空間。

相反，如果 `channel` 是空的 ， 接收操作將阻塞直到有另一個 `goroutine` 執行發送操作而向隊列插入元素 。

```go
package main

import (
    "fmt"
)

func main()  {
    ch := make(chan string, 3)

    ch <- "A"
    ch <- "B"
    ch <- "C"

    fmt.Println(<-ch)
    fmt.Println(<-ch)
    fmt.Println(<-ch)
}

```

## select

Go 語言的 `select` 的功能和 `select、poll、epoll` 相似 ， 就是監聽 IO 操作 ， 當 IO 操作發生時 ， 觸發響應的動作 。

`select` 語法和 `switch` 相似 ， 也會有幾個 `case` 和 `default` 分支 ， 每一個 `case` 代表一個通信操作(在某個 `channel` 上進行發送或者接收)並且會包含一些語句組成一個語句塊。

```go
package main

import (
    "fmt"
    "strconv"
)

func main()  {
    ch1 := make(chan string)
    ch2 := make(chan string)

    go func() {
        ch1 <- "I am from ch1"
    }()

    go func() {
        ch2 <- "I am from ch2"
    }()

    for i := 0; i < 2; i++ {
        select {
        case msg1 := <- ch1:
            fmt.Println("received-"+strconv.Itoa(i), msg1)
        case msg2 := <- ch2:
            fmt.Println("received-"+strconv.Itoa(i), msg2)
        }
    }
}

```

## 超時控制

`select` 是用來讓我們的程序監聽多個文件句柄的狀態變化的處理機制 。

當發起一些阻塞的請求後 ， 可以用 `select` 機制輪詢掃描文件句柄 ， 直到被監視的文件句柄有一個或多個發生了狀態改變。

`channel` 在系統層面來說也是個文件描述符 ， 在 Go 語言中我們可以用 `goroutine` 並發執行任務 ， 接著使用 `select` 來監視每個任務的 `channel` 情況 。

如果這幾個任務都長時間沒有回复 `channel` 信息 ， 並且我們又有超時的需求 ， 那麼我們可以使用一個 `goroutine` 來設置超時機制 ， 具體做法就是啟動 `sleep` 並且在 `sleep` 之後回复 `channel` 信號。

```go
package main

import (
    "fmt"
    "time"
)

func main()  {
    ch := make(chan string)
    timeout := make(chan bool)

    go func() {
        time.Sleep(5 * time.Second)
        timeout <- true
    }()

    go func() {
        time.Sleep(10 * time.Second)
        ch <- "Hello World"
    }()

    select {
    case msg := <- ch:
        fmt.Println(msg)
    case <- timeout:
        fmt.Println("task is timeout")
    }
}

```