---
title: "Javscript 筆記"
date: 2020-10-15T15:43:43+08:00
lastmod: 2020-10-15T15:43:43+08:00
draft: true
keywords: []
description: ""
tags: ["javascript"]
categories: ["紀錄"]
author: ""
---
 
紀錄 Javscript 相關筆記

<!--more-->

## 1. 用以下原生的 Javascript 語法來取代 jQyery 語法

### Querying the DOM

```javascript
jQuery('div.html')
document.querySelector('div.home')
document.querySelectorAll('img')
```

### .html()

```javascript
// 獲取 html
var htmlContent = document.querySelector('.main-div').innerHtml;
// 塞入 html
document.querySelector('.main-div').innerHtml = '<p>Hello World</p>';
```

### .val()

```javascript
// 取值
var inputValue = document.querySelector('.input-box').value;
// 賦值
document.querySelector('.input-box').value = 'Demo String';
```

### .text()

```javascript
let textValue = document.querySelector('.main-div').innerText;
```

### Adding/Removing Classes

```javascript
document.querySelector('#testElement').classList.add('testClass');

document.querySelector('#testElement').classList.remove('testClass')
```

### Adding/Modifying CSS

```javascript
document.querySelector('#testElement').style.color = '#ffffff';

document.querySelector('#testElement').style.backgroundColor = '#000000';
```

### .attr()

```javascript
document.querySelector('img').src = '/images/image.png';
document.querySelector('img').getAttribute('data-title');
```

### .show() / .hide()

```javascript
document.querySelector('.test-element').style.display = 'none';

document.querySelector('.test-element').style.display = 'block';
```

## 2. debounce（防抖）

```javascript
const debounce = (fn, time) => {
  let timeout = null;
  return function() {
    clearTimeout(timeout)
    timeout = setTimeout(() => {
      fn.apply(this, arguments);
    }, time);
  }
};
```

## 3. throttle（節流）

```javascript
const throttle = (fn, time) => {
  let flag = true;
  return function() {
    if (!flag) return;
    flag = false;
    setTimeout(() => {
      fn.apply(this, arguments);
      flag = true;
    }, time);
  }
}
```