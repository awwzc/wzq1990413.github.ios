---
layout: page
title: About
description: 搬砖打杂
keywords: Zhiqiang Wu, 伍志强
comments: true
menu: 关于
permalink: /about/
---

我是伍志强，搬搬砖，打打杂。

写优雅的代码,过自己的生活。


## 联系

{% for website in site.data.social %}
* {{ website.sitename }}：[@{{ website.name }}]({{ website.url }})
{% endfor %}

## Skill Keywords

{% for category in site.data.skills %}
### {{ category.name }}
<div class="btn-inline">
{% for keyword in category.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
