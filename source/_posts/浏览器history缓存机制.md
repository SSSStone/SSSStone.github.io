---
title: 浏览器History缓存机制
date: 2017-09-30 17:30:18
tags: 
- 浏览器
- History
- 缓存
categories: 
- 浏览器
---

用户点击浏览器工具栏中的后退按钮，或者移动设备中的返回键，或者js执行`history.back()`时，浏览器会再当前窗口‘打开’历史记录中的前一个页面。
那么‘打开’这个操作是如何实现的？在不同的浏览器中实现方式是否一样？

<!-- more -->

# History.back()标准

先从标准规范入手，浏览器history mechanisms标准如下：

https://www.w3.org/Protocols/rfc2616/rfc2616-sec13.html#sec13.13

>13.13 History Lists
User agents often have history mechanisms, such as "Back" buttons and history lists, which can be used to redisplay an entity retrieved earlier in a session.
History mechanisms and caches are different. **In particular history mechanisms SHOULD NOT try to show a semantically transparent view of the current state of a resource. Rather, a history mechanism is meant to show exactly what the user saw at the time when the resource was retrieved.** By default, an expiration time does not apply to history mechanisms. If the entity is still in storage, a history mechanism SHOULD display it even if the entity has expired, unless the user has specifically configured the agent to refresh expired history documents. This is not to be construed to prohibit the history mechanism from telling the user that a view might be stale.

>Note: if history list mechanisms unnecessarily prevent users from viewing stale resources, this will tend to force service authors to avoid using HTTP expiration controls and cache controls when they would otherwise like to. Service authors may consider it important that users not be presented with error messages or warning messages when they use navigation controls (such as BACK) to view previously fetched resources. Even though sometimes such resources ought not to cached, or ought to expire quickly, user interface considerations may force service authors to resort to **other means of preventing caching** (e.g. "once-only" URLs) in order not to suffer the effects of improperly functioning history mechanisms.

简单来说，就是浏览器的back()操作在打开历史页面时，不需要遵循HTTP缓存规范，会尽可能地还原用户离开页面时的页面状态。

浏览器的这个特性，能够让页面在浏览，特别是"返回/前进"切换的时候更加流畅；但是，因为它对回退页面的缓存，在实际开发中如果不注意的话，很容易踩坑。

chrome、Safari、firefox等各大浏览器厂商在此规范下作出了不同的尝试。

# 在各浏览器中的实现

## 在Safari/Firefox等浏览器中的表现

Safari中实现了Page Cache
https://webkit.org/blog/427/webkit-page-cache-i-the-basics/

Firefox中实现了Back/Forward Cache
https://developer.mozilla.org/en-US/Firefox/Releases/1.5/Using_Firefox_1.5_caching
https://developer.mozilla.org/en-US/docs/Working_with_BFCache

Opera中实现了Fast History Navigation

Pagecache完全保存了用户离开页面前的状态，包括HTML、JS、CSS文件和DOM、JS对象的状态。

用一个很简单的例子说明

```javascript
window.onload = function () {
	console.log(1);
};
var i = 0;
setInterval(function () {
	i++;
	console.log(i);
},2000)
```

用户从这个页面跳转到其它页面再返回时：
- 不会重新发送HTTP请求资源文件 ---> 静态资源文件被缓存
- 不会触发页面onload事件 ---> DOM状态被缓存，JS不会重新执行
- 变量i从离开前的最大值开始增加 ---> js对象状态被保存

可以形象地理解为：当用户在离开页面的时候，页面“暂停”，用户返回到该页面的时候，再“继续”。

## 在Chrome(Blink)浏览器中的表现

使用Blink内核的浏览器会缓存静态文件，在用户返回该页面时，从缓存中读取文件重新加载。

>Blink内核上移除了以前webkit内核有关Page Cache相关的功能，Google给的解释是这个模块不适合多进程的架构。

同样的例子运行结果：
- 不会重新发送HTTP请求资源文件 ---> 静态资源文件被缓存
- 会触发页面onload事件 --->JS重新执行
- 变量i从0开始增加

# 遇到问题

## History mechanisms本身带来的问题

我们已经知道History mechanisms会~~无条件~~缓存页面，对于一些对时间有依赖性的页面，需要在用户返回时给到用户明确的过期提醒，或者通过一些手段让页面刷新。

"无条件"的表述并不准确，firefox给出说明，在以下情形下不会缓存页面。

>- the page uses an unload or beforeunload handler;
>- the page sets "cache-control: no-store".
>- the site is HTTPS and page sets at least one of:
>	- "Cache-Control: no-cache"
>	- "Pragma: no-cache"
>	- with "Expires: 0" or "Expires" with a date value in the past relative to the value of the "Date" header (unless "Cache-Control: max-age=" is also specified);
>- the page is not completely loaded when the user navigates away from it or has pending network requests for other reasons (e.g. XMLHttpRequest));
>- the page has running IndexedDB transactions;
>- the top-level page contains frames (e.g. &lt;iframe&gt;) that are not cacheable for any of the reasons listed here;
>- the page is in a frame and the user loads a new page within that frame (in this case, when the user navigates away from the page, the content that was last loaded into the frames is what is cached).

不管是提醒用户页面过期，还是重新加载页面，都需要监听到页面加载事件且能够判断页面是否来自pagecache，`pageshow()` 事件完全满足我们的需求。

### pageshow

>The pageshow event is fired when a session history entry is being traversed to.

在前文中我们已经了解到，如果返回页面来自PageCache(B/FCache类似)，页面onload事件不会执行，但是我们可以通过pageshow事件来监控页面加载。

- pageshow事件在页面显示时触发，无论页面是否来自PageCache
- pageshow事件在load事件后触发
- pageshow事件的event对象包含一个名为persisted的布尔值属性，如果页面保存在pagecache中，该值为true

```javascript
window.addEventListener('pageshow', function(event){
	if(event.persisted){
		alert('当前页面来自pagecache');
		location.replace(location.href);
	}
});
```

## 使用history.replacState()时的问题

在项目中，我们经常会用到history.replacState()来控制用户‘前进/后退’的体验，特别是在单页面应用中，我们为了保证用户返回时可以返回到指定页面，而不是首页，会用到history.replacState()。
当我们使用了history.replacState()或者history.pushState()以后，如果不对浏览器history.back()进行处理，会带来很多问题。

### chrome history mechanisms + history.replacState()

#### 症状

![](http://km.oa.com/files/photos/pictures/201709/1504946446_1_w649_h214.png)
(1)第一次：用户在A页面使用replaceState(null,null,c)到状态A'，跳转到B页面(活动页之类)，此时用户点击返回，会加载A'，历史缓存中没有C页面URL的缓存，所以加载C页面并缓存C页面。
(2)第二次：两天后，用户重复此操作，点击返回，会加载A'，历史缓存中有C页面缓存，直接读取缓存，展示给用户的是两天前缓存的C页面！
此时，C页面上的信息的时效性完全不能保证和预测。

#### 规避

1. Cache-Control: no-cache
2. Cache-Control: no-store
3. 随机数后缀

三种策略在规避history mechanisms中的表现：
![](http://km.oa.com/files/photos/pictures/201709/1504947926_93_w614_h73.png)
我们可以使用2、3两种策略来规避chrome history mechanisms对页面返回的影响。

### Page cache + history.replacState()

#### 症状

history.replacState()失效。
![](http://km.oa.com/files/photos/pictures/201709/1504948033_72_w645_h219.png)
我们想让用户从B页面返回时，能加载C页面，然而因为Pagecache的存在，用户返回A'后，虽然URL是C页面的URL，可是页面从cache中读取，并不会重新加载，也就不会展现给用户C页面。

#### 规避

`replaceState()` 已生效，我们只需要在用户返回到A'时，重新刷新页面，即可的到C页面，所以同样可以监听 `pageshow()` 事件来处理。

```javascript
window.addEventListener('pageshow', function(event){
	if(event.persisted){
		location.replace(location.href);
	}
});
```

# 总结回顾

history mechanisms 的存在是为了让用户在进行“前进/后退”操作时更流畅，很大程度上提升了用户体验，但是，这种优化并不一定和我们的项目兼容。作为前端开发，我们要深入了解这些优化措施背后的原理与规范，才能避免踩坑。

及时总结，同一个坑不能踩第二次。