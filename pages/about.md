---
layout: page
title: About
description: 
keywords: Jared Zhu
comments: true
menu: 关于
permalink: /about/
---

误打误撞走进了处理器行业，一路上磕磕绊绊，总归算是走过了十年...

经历过高光，经历过低谷，33岁的年纪，又将重新启航，继续征途在处理器的领域...

专注于**性能**，**理财**，以及分享一些**读书笔记**，**生活感悟**，欢迎大家一起探讨


## 联系
**邮箱：** freejared@163.com


## Skill Keywords

{% for skill in site.data.skills %}
### {{ skill.name }}
<div class="btn-inline">
{% for keyword in skill.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
