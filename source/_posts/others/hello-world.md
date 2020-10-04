---
title: Hello Hexo
date: 2019/3/24 19:00
categories:
# - [前端, 源码阅读, JavaScript]
tags:
# - ps3
---

使用hexo搭建博客的测试文章，测试常用markdown，链接配置，注释一些默认的页面信息。
<!--more-->

# 标题测试h1
## 标题测试h2
### 标题测试h3
---
## 下面来一些列表测试
无序列表
- 无序1
- 无序2
有序列表
1. 有序列表1
2. 有序列表2
---
## 来个引用测试吧
图片的引用也会居中显示
![图片](/blog/public/imgs/1.png)
[链接效果达到预期，返回主站](https://www.tangdingblog.cn)
[来个锚点跳转，跳转底部](#bottom)
> 事实上文字引用会居中，没有达到预期
---
## 接下来表格测试

header 1 | header 2
---|---
row 1 col 1 | row 1 col 2
row 2 col 1 | row 2 col 2

表格的效果比预期的要好很多

---
## code test
``` javascript
class Test extends React.Component {
    constructor(props) {
        super(props);
    }
}
// 效果相当的好
```

---
## 样式测试
==mark==
**加粗**
*倾斜*
++下划线++
~~中划线~~
- [ ] checkbox
- [x] checkedbox


<span id="bottom">底部</span>