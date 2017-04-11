---
title: 设置和移除disabled属性
date: 2017-03-31 19:34:18
tags: 
- disabled
categories: 
- CSS
---

```
//两种方法设置disabled属性 
$('#areaSelect').attr("disabled",true); 
$('#areaSelect').attr("disabled","disabled"); 

//三种方法移除disabled属性 
$('#areaSelect').attr("disabled",false); 
$('#areaSelect').removeAttr("disabled"); 
$('#areaSelect').attr("disabled",""); 
```