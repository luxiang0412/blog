---
title: idea js minimal
date: 2021-01-14 12:47:36
tags: ["npm", "idea"]
---

## 安装

npm安装uglifyjs
```bash
npm install uglifyjs -g
```

idea创建exteral tools

- Settings
- Tools
- External Tools
- Create Tool

|属性|值|
| --- | --- |
|Name|`uglifyjs`|
|Program|`C:\Users\luxiang\AppData\Roaming\npm\uglifyjs.cmd`|
|Arguments|`$FilePath$ -o $FileDir$\$FileNameWithoutExtension$.min.$FileExt$`|
|Working directory|`$ProjectFileDir$`|

如图
<img src="config.jpg">
---

## 使用

<img src="use.jpg">
