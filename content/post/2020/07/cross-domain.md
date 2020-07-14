---
title: '關於 Web 跨域相關紀錄'
date: 2020-07-14T08:51:19+08:00
lastmod: 2020-07-14T08:51:19+08:00
draft: false
keywords: ['CORS']
description: ''
tags: ['CORS', 'web']
categories: ['紀錄']
---

## 前言

前段時間公司同事出現了跨域問題 , 雖然我知道了大略的解決方案 , 但是說實在的真實真實的跨域知識及領域 ,

不是了解得很透徹 , 趁這個機會好好查一下文章記錄下來

<!--more-->

## 1. 什麼是跨域

跨域簡單的說是指一個網域下的網頁或腳本想要去請求另一個網域下的資源 ,

而我們的跨域 , 大都是指瀏覽器基於同源政策限制下產生問題

## 2. 瀏覽器的同源

同源策略/SOP（Same origin policy）是一種協定，由 `Netscape` 公司 `1995` 年引入瀏覽器，它是瀏覽器最核心也最基本的安全功能，如果缺少了同源策略，瀏覽器很容易受到 `XSS` 、`CSFR` 等攻擊

同源，什麼是源呢？源指的是 `協議`、`域名`、`端口`，那麼同源即三者相同，即便是不同的域名指向同一個 IP 位址，也不同源

以 `http://www.example.com:80/dir/page.html` 這一個網址來說

- `http://` --> `協議`
- `www` --> `子域名`
- `example.com` --> `主域名`
- `80` --> `端口`
- `/dir/page.html` --> 請求資源位置

如果像下面網址請求的同源狀況

> - http://www.example.com/dir2/other.html ---> `同源`
> - http://example.com/dir/other.html --> `不同源（域名不同）`
> - http://v2.www.example.com/dir/other.html --> `不同源（域名不同）`
> - https://www.example.com/dir/other.html --> `不同源（協議不同）`
> - http://www.example.com:81/dir/other.html --> `不同源（端口不同）`

如有以上情境 , 瀏覽器將會限制我們以下限制

> - Cookie、LocalStorage 和 IndexDB 無法讀取。
> - DOM 無法獲得。
> - AJAX 請求不能發送。

## 3. 如何解決

### 3.1. JSONP

#### 3.1.1. JSONP 原理

假如有兩的不同域名的資源 a.html 和 b.js

a.html

```html
<html>
  <head>
    <title>test</title>
    <script type="text/javascript" src="http://www.example2.com/b.js"></script>
  </head>
  <body>
    <script>
      console.log(b) // Hello~~
    </script>
  </body>
</html>
```

b.js

```javascript
var b = 'Hello~~'
```

由這一個例子來講 , 我們可以了解 `<script>` 這個標籤的 `scr` 的屬性是不被同源政策所限制 ,

所以這就是 `JSONP` 的核心原理

#### 3.1.2. 如何實現

- CallBack

  a.html : http://www.example.com/a.html

  ```html
  <script type="text/javascript" src="http://www.example2.com/b.js"></script>

  <script type="text/javascript">
    function cb(res) {
      console.log(res.data.b) // 我是b
    }
  </script>
  ```

  b.js : http://www.example2.com/b.js

  ```javascript
  var b = '我是b'
  // 調用 cb 函數，並以 json 數據形式做參數
  cb({
    code: 200,
    msg: 'success',
    data: {
      b: b,
    },
  })
  ```

  創建一個回調函數，然後在遠程服務上調用這個函數並且將 `JSON` 數據形式作為參數傳遞，完成回調，就是 `JSONP` 的簡單實現模式，或者說是 `JSONP` 的原型，是不是很簡單呢

  將 `JSON` 數據填充進回調函數，現在懂為什麼 `JSONP` 叫 `JSON with Padding` 了吧

  上面這種實現很簡單，通常情況下，我們希望這個 `script` 標籤能夠動態的調用，而不是像上面因為固定在 `HTML` 裡面加載時直接執行了，很不靈活，我們可以通過 `javascript` 動態的創建 `script` 標籤，這樣我們就可以靈活調用遠程服務了，那麼我們簡單改造下`頁面 a` 如下

  ```html
  <script type="text/javascript">
    function cb(res) {
      console.log(res.data.b) // 我是b
    }

    // 動態添加 <script>
    function addScriptTag(src) {
      let script = document.createElement('script')
      script.setAttribute('type', 'text/javascript')
      script.src = src
      document.body.appendChild(script)
    }

    window.onload = function () {
      addScriptTag('http://www.example2.com/b.js')
    }
  </script>
  ```

  如上所示，只是些基礎操作，就不解釋了，現在我們就可以優雅的控制執行了，再想調用一個遠程服務的話，只要添加 `addScriptTag` 方法，傳入遠程服務的 `src` 值就可以

  接下來我們就可以愉快的進行一次真正意義上的 `JSONP` 服務調取了

  我們使用 `jsonplaceholder` 的 `todos` 接口作為示例，接口地址如下

  > https://jsonplaceholder.typicode.com/todos?callback=?

  `callback=?` 這個接在網址後面表示回調函數的名稱，也就是將你自己在客戶端定義的回調函數的函數名傳送給服務端，

  服務端則會返回以你定義的回調函數名的方法，將獲取的 `JSON` 數據傳入這個方法完成回調，我們的回調函數名字叫 `cb`，那麼完整的接口地址就如下

  > https://jsonplaceholder.typicode.com/todos?callback=cb

  麼話不多說，我們來試下

  ```html
  <script type="text/javascript">
    function cb(res) {
      console.log(res)
    }

    function addScriptTag(src) {
      let script = document.createElement('script')
      script.setAttribute('type', 'text/javascript')
      script.src = src
      document.body.appendChild(script)
    }

    window.onload = function () {
      addScriptTag('https://jsonplaceholder.typicode.com/todos?callback=cb')
    }
  </script>
  ```

  可以看到，頁面在加載完成後，輸出了接口返回的數據，這個時候我們再來看 `jQuery` 中的 `JSONP` 實現

- jQuery

  還是用上面的接口，我們來看 JQ 怎麼拿數據

  ```html
  <script>
    $.ajax({
      url: 'https://jsonplaceholder.typicode.com/todos?callback=?',
      dataType: 'jsonp',
      jsonpCallback: 'cb',
      success: function (res) {
        console.log(res)
      },
    })
  </script>
  ```

  可以看到，為了讓 `jQuery` 按照 `JSONP` 的方式訪問，`dataType` 字段設置為 `jsonp` ， `jsonpCallback` 屬性的作用就是自定義我們的回調方法名，其實內部和我們上面寫的差不多

#### 3.1.3. JSONP VS AJAX

調用方式上

- `AJAX` 和 `JSONP` 很像，都是請求 `url`，然後把服務器返回的數據進行處理
- 所以類 `JQuery` 的庫只是把 `JSONP` 作為 `AJAX` 請求的一種形式進行封裝，不要搞混

核心原理上

- `AJAX` 的核心是通過 `xmlHttpRequest` 獲取非本頁內容
- `JSONP` 的核心是動態添加 `script` 標籤調用服務器提供的 `JS` 腳本，後綴 `.json`

兩者區別上，

- `AJAX` 不同域會報跨域錯誤，不過也可以通過服務端代理、`CORS` 等方式跨域，而 `JSONP` 沒有這個限制，同域不同域都可以
- `JSONP` 是一種方式或者說非強制性的協議，`AJAX` 也不一定非要用 `json` 格式來傳遞數據
- `JSONP` 只支持 `GET` 請求，`AJAX` 支持 `GET` 和 `POST`

### 3.2. CORS

`CORS` 需要瀏覽器和後端同時支持。 `IE 8` 和 `9` 需要通過 `XDomainRequest` 來實現。

瀏覽器會自動進行 `CORS` 通信，實現 `CORS` 通信的關鍵是後端 。 只要後端實現了 `CORS`，就實現了跨域。

服務端設置 `Access-Control-Allow-Origin` 就可以開啟 `CORS` 。

該屬性表示哪些域名可以訪問資源，如果設置通配符則表示所有網站都可以訪問資源。

雖然設置 `CORS` 和前端沒什麼關係，但是通過這種方式解決跨域問題的話，會在發送請求時出現兩種情況，分別為簡單請求和復雜請求。

#### 3.2.1. 間單請求

只要同時滿足以下兩大條件，就屬於簡單請求

- 條件 1：使用下列方法之一：

  - `GET`
  - `HEAD`
  - `POST`

- 條件 2：`Content-Type` 的值僅限於下列三者之一：

  - `text/plain`
  - `multipart/form-data`
  - `application/x-www-form-urlencoded`

請求中的任意 `XMLHttpRequestUpload` 對象均沒有註冊任何事件監聽器；

`XMLHttpRequestUpload` 對象可以使用 `XMLHttpRequest.upload` 屬性訪問。

#### 3.2.2. 複雜請求

      不符合以上條件的請求就肯定是複雜請求了。

      複雜請求的 `CORS` 請求，會在正式通信之前，增加一次HTTP查詢請求，稱為 "預檢" 請求, 該請求是 `option` 方法的，通過該請求來知道服務端是否允許跨域請求。

      我們用 `PUT` 向後台請求時，屬於復雜請求，後台需做如下配置：

      ```javascript
          // 允許哪個方法訪問我
          res.setHeader('Access-Control-Allow-Methods', 'PUT')
          // 預檢的存活時間
          res.setHeader('Access-Control-Max-Age', 6)

          // OPTIONS 請求不做任何處理
          if (req.method === 'OPTIONS') {
              res.end()
          }
          // 定義後台返回的內容
          app.put('/getData', function(req, res) {
              console.log(req.headers)
              res.end('Hello')
          })
      ```

      接下來我們看下一個完整復雜請求的例子，並且介紹下CORS請求相關的字段

      ```html
          // index.html
          let xhr = new XMLHttpRequest()
          document.cookie = 'name=xiamen' // cookie 不能跨域
          xhr.withCredentials = true // 前端設置是否帶 cookie
          xhr.open('PUT', 'http://localhost:4000/getData', true)
          xhr.setRequestHeader('name', 'xiamen')
          xhr.onreadystatechange = function() {
              if (xhr.readyState === 4) {
                  if ((xhr.status >= 200 && xhr.status < 300) || xhr.status === 304) {
                      console.log(xhr.response)
                      //得到響應頭，後台需設置 Access-Control-Expose-Headers
                      console.log(xhr.getResponseHeader('name'))
                  }
              }
          }
          xhr.send()

      ```

      ```javascript
      //server1.js
      let express = require('express')
      let app = express()
      app.use(express.static(__dirname))
      app.listen(3000)
      複製代碼 //server2.js
      let express = require('express')
      let app = express()
      let whitList = ['http://localhost:3000'] //設置白名單
      app.use(function (req, res, next) {
        let origin = req.headers.origin
        if (whitList.includes(origin)) {
          // 設置哪個源可以訪問我
          res.setHeader('Access-Control-Allow-Origin', origin)
          // 允許攜帶哪個頭訪問我
          res.setHeader('Access-Control-Allow-Headers', 'name')
          // 允許哪個方法訪問我
          res.setHeader('Access-Control-Allow-Methods', 'PUT')
          // 允許攜帶cookie
          res.setHeader('Access-Control-Allow-Credentials', true)
          // 預檢的存活時間
          res.setHeader('Access-Control-Max-Age', 6)
          // 允許返回的頭
          res.setHeader('Access-Control-Expose-Headers', 'name')
          if (req.method === 'OPTIONS') {
            res.end() // OPTIONS請求不做任何處理
          }
        }
        next()
      })
      app.put('/getData', function (req, res) {
        console.log(req.headers)
        res.setHeader('name', 'jw') //返回一個響應頭，後台需設置
        res.end('Hello')
      })
      app.get('/getData', function (req, res) {
        console.log(req.headers)
        res.end('Hello')
      })
      app.use(express.static(__dirname))
      app.listen(4000)
      ```

上述代碼由 `http://localhost:3000/index.html` 向 `http://localhost:4000/` 跨域請求，正如我們上面所說的，後端是實現 `CORS` 通信的關鍵。

### 3.3. postMessage

`postMessage` 是 `HTML5` `XMLHttpRequest Level 2` 中的 API，且是為數不多可以跨域操作的 window 屬性之一，它可用於解決以下方面的問題：

- 頁面和其打開的新窗口的數據傳遞
- 多窗口之間消息傳遞
- 頁面與嵌套的 iframe 消息傳遞
- 上面三個場景的跨域數據傳遞

`postMessage()`方法允許來自不同源的腳本採用異步方式進行有限的通信，可以實現跨文本檔、多窗口、跨域消息傳遞。

> otherWindow.postMessage(message, targetOrigin, [transfer]);

- message: 將要發送到其他 window 的數據。
- targetOrigin:通過窗口的 origin 屬性來指定哪些窗口能接收到消息事件，其值可以是字符串"\*"（表示無限制）或者一個 URI。在發送消息的時候，如果目標窗口的協議、主機地址或端口這三者的任意一項不匹配 targetOrigin 提供的值，那麼消息就不會被發送；只有三者完全匹配，消息才會被發送。
- transfer(可選)：是一串和 message 同時傳遞的 Transferable 對象. 這些對象的所有權將被轉移給消息的接收方，而發送一方將不再保有所有權。

  接下來我們看個例子：

  `http://localhost:3000/a.html` 頁面向 `http://localhost:4000/b.html` 傳遞 'Hello' , 然後後者傳回"Hi"。

  ```html
  // a.html
  <iframe
    src="http://localhost:4000/b.html"
    frameborder="0"
    id="frame"
    onload="load()"
  ></iframe>
  //等它加載完觸發一個事件, 內嵌在http://localhost:3000/a.html
  <script>
    function load() {
      let frame = document.getElementById('frame')
      frame.contentWindow.postMessage('Hello', 'http://localhost:4000') //發送數據
      window.onmessage = function (e) {
        //接受返回數據
        console.log(e.data) //Hi
      }
    }
  </script>
  ```

  ```javascript
  // b.html
  window.onmessage = function (e) {
    console.log(e.data) //Hello
    e.source.postMessage('Hi', e.origin)
  }
  ```

### 3.4. WebScoket

客戶端：http://www.example.com/a.html

```html
<script src="/socket.io/socket.io.js"></script>

<script>
  let socket = io.connect('ws://www.example1.com:3000')

  socket.on('my event', (data) => {
    console.log(data) // { hello: 'world' }
    socket.emit('my other event', { my: 'data' })
  })
</script>
```

服務端：ws://www.example1.com:3000

```javascript
const app = require('express').createServer()
const io = require('socket.io')(app)

app.listen(3000)

io.on('connection', (socket) => {
  socket.emit('my event', { hello: 'world' })

  socket.on('my other event', (data) => {
    console.log(data) // { my: 'data' }
  })
})
```

### 3.5. Nginx 反向代理

```shell
#proxy
    server {
        listen       80;
        server_name  www.example.com;

        location / {
            proxy_pass   http://www.example2.com:8080;  #反向代理
            proxy_cookie_domain www.example2.com www.example.com; #修改cookie里域名
            index  index.html index.htm;
            add_header Access-Control-Allow-Origin http://www.example.com;  #當前端只跨域不帶cookie時，可為*
            add_header Access-Control-Allow-Credentials true;
        }
    }
```

### 3.6. window.name + iframe

`window.name` 屬性的獨特之處：`name` 值在不同的頁面（甚至不同域名）加載後依舊存在，並且可以支持非常長的 name 值（2MB）。

其中 `a.html` 和 `b.html` 是同域的，都是 `http://localhost:3000` , 而 `c.html`是 `http://localhost:4000`

```html
// a.html(http://localhost:3000/b.html)
<iframe
  src="http://localhost:4000/c.html"
  frameborder="0"
  onload="load()"
  id="iframe"
></iframe>

<script>
  let first = true
  // onload事件會觸發2次，第1次加載跨域頁，並留存數據於window.name
  function load() {
    if (first) {
      // 第1次onload(跨域頁)成功後，切換到同域代理頁面
      let iframe = document.getElementById('iframe')
      iframe.src = 'http://localhost:3000/b.html'
      first = false
    } else {
      // 第2次onload(同域b.html頁)成功後，讀取同域window.name中數據
      console.log(iframe.contentWindow.name)
    }
  }
</script>
```

b.html 為中間代理頁，與 a.html 同域，內容為空。

```html
// c.html(http://localhost:4000/c.html)
<script>
  window.name = 'Hello'
</script>
```

總結：通過 `iframe` 的 `src` 屬性由外域轉向本地域，跨域數據即由 `iframe` 的 `window.name` 從外域傳遞到本地域。

個就巧妙地繞過了瀏覽器的跨域訪問限制，但同時它又是安全操作。

### 3.7. location.hash + iframe

實現原理： `a.html` 欲與 `c.html` 跨域相互通信，通過中間頁 `b.html` 來實現。

三個頁面，不同域之間利用 `iframe` 的 `location.hash` 傳值，相同域之間直接 js 訪問來通信。

具體實現步驟：一開始 `a.html` 給 `c.html` 傳一個 hash 值，然後 `c.html` 收到 `hash` 值後，再把 `hash` 值傳遞給 `b.html`，最後 `b.html` 將結果放到 `a.html` 的 `hash` 值中。

同樣的，`a.html` 和 `b.html` 是同域的，都是 `http://localhost:3000` 而 `c.html`是 `http://localhost:4000`

```html
// a.html
<iframe src="http://localhost:4000/c.html#iloveyou"></iframe>
<script>
  window.onhashchange = function () {
    //檢測hash的變化
    console.log(location.hash)
  }
</script>
```

```html
// b.html
<script>
  window.parent.parent.location.hash = location.hash
  //b.html將結果放到a.html的hash值中，b.html可通過parent.parent訪問a.html頁面
</script>
```

```html
// c.html
<script>
  console.log(location.hash)
  let iframe = document.createElement('iframe')
  iframe.src = 'http://localhost:3000/b.html#idontloveyou'
  document.body.appendChild(iframe)
  document.domain + iframe
</script>
```

### 3.8. document.domain + iframe

該方式只能用於二級域名相同的情況下，比如 `a.test.com` 和 `b.test.com` 適用於該方式。

只需要給頁面添加 `document.domain ='test.com'` 表示二級域名都相同就可以實現跨域。

實現原理：兩個頁面都通過 `js` 強制設置 `document.domain` 為基礎主域，就實現了同域。

我們看個例子：頁面 `a.test.com:3000/a.html` 獲取頁面 `b.test.com:3000/b.html` 中 `a` 的值

```html
// a.html
<body>
  helloa
  <iframe
    src="http://b.test.com:3000/b.html"
    frameborder="0"
    onload="load()"
    id="frame"
  ></iframe>
  <script>
    document.domain = 'test.com'
    function load() {
      console.log(frame.contentWindow.a)
    }
  </script>
</body>
```

```html
// b.html
<body>
  hellob
  <script>
    document.domain = 'test.com'
    var a = 100
  </script>
</body>
```

## 4. 参考文章

- [浏览器同源政策及其规避方法 - 阮一峰](http://www.ruanyifeng.com/blog/2016/04/same-origin-policy.html)
- [跨域资源共享 CORS 详解 - 阮一峰](http://www.ruanyifeng.com/blog/2016/04/cors.html)
- [你真的了解跨域吗 - 掘金](https://juejin.im/post/5f0b3e136fb9a07eaa40bc0d#heading-0)
