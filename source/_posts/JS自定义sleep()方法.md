---
title: JS自定义sleep()方法
date: 2017-03-31 19:34:18
tags: 
- sleep()
categories: 
- JavaScript应用
---

JS中不存在自带的sleep方法，要想休眠要自己定义sleep()方法。

```javascript
function sleep(milliSeconds){
    var startTime = new Date().getTime(); // get the current time
    while (new Date().getTime() < startTime + milliSeconds) {} // hog cpu
}
```
