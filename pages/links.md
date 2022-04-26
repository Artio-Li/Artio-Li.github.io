---
layout: mypost
title: 友情链接
---

欢迎各位朋友与我建立友链，如需友链请到留言板留言，我看到留言后会添加上的，本站的友链信息如下

```
名称：{{ site.title }}
描述：{{ site.description }}
地址：{{ site.domainUrl }}{{ site.baseurl }}

名称：Mox的笔记库
链接：https://www.mocusez.site
备注：探索未曾设想的道路
```

<ul>
  {%- for link in site.links %}
  <li>
    <p><a href="{{ link.url }}" title="{{ link.desc }}" target="_blank" >{{ link.title }}</a></p>
  </li>
  {%- endfor %}
</ul>
