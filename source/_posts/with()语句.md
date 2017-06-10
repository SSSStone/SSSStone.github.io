__---
title: with()用法
date: 2017-03-31 19:34:18
tags: 
- with()
categories: 
- JavaScript原理
---

with语句主要用来临时扩展作用域链，将语句中的对象添加到作用域的头部。
with语句结束后，作用域链恢复正常。

<!-- more -->

```javascript
name="1";
person={
	name:"2"
};
with(person){
	console.log(name);
};
console.log(name);
//结果
2
1
```