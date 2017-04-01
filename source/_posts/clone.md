---
title: 复制JS的5中数据类型
date: 2017-03-31 19:34:18
tags: 
- JS数据类型
categories: 
- JavaScript应用
---

对JavaScript中的5种主要的数据类型（包括Number、String、Object、Array、Boolean）进行值复制

<!-- more -->

```javascript
Object.prototype.clone = function(){
    var o = this.constructor === Array ? [] : {};
    for(var e in this){
        o[e] = typeof this[e] === "object" ? this[e].clone() : this[e];
    }
    return o;
}
```