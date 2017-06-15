---
title: hexo-NEXT主题配置Algolia
date: 2017-06-13 23:30:27
tags:
- hexo
- NEXT
categories:
- 杂记
- 工具
---

Next 主题集成 Algolia 搜索功能，丰富我们的博客。

Algolia 的基础操作参考 Next 主题大腿的使用文档 - http://theme-next.iissnan.com/third-party-services.html#algolia-search

本文主要记录使用过程中需要注意的两点。

<!-- more -->

# `hexo-algolia` 工具更新。

以前把 `adminApiKey` 直接配置到配置文件中，容易造成密钥泄露，得到 `adminApiKey` 就可以拿到你账户下所有的权限，安全性很低。

Never use your Admin API Key to index content — you might leak it by mistake and it gives full control of your to others.

`hexo-algolia` 工具更新到1.1.0版本后，对 algolia 的 API-KEY 加强了保护。

具体更新内容参考 https://github.com/oncletom/hexo-algolia

1 禁用 `adminApiKey` 使用自定义 `ApiKey`

添加自定义 `ApiKey` ，并设定其权限。

![](/images/algolia-write-key.png)

2 配置文件改为配置环境变量

临时修改

```text
export HEXO_ALGOLIA_INDEXING_KEY='1341232*********41341'
```

开机生效

```text
$ vim ~/.bash_profile

# 添加全局变量
export HEXO_ALGOLIA_INDEXING_KEY='1341232*********41341'

$ source ~/.bash_profile
```

# 修改 `hexo-algolia` 代码。

修改 `hexo-algolia` 代码，是其不会上传 `content` 字段。

因为 `content` 字段为博客全文，过长影响查询效率。

修改位置： `node_modules/hexo-algolia/lib/command.js`  文件 37-44 行。

```javascript
var storedPost = _.pick(data, [
    'title',
    'date',
    'slug',
    //'content',
    'excerpt',
    'permalink'
]);

// 注释掉 content 即可
```