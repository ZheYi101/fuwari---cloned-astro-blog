---
title: elmUI真王朝了? 使用经验分享
description: 之前就用过elmUI 但这次单人写服外前端Vue项目用了大量elmUI组件 感觉甚好 分享下喜悦
published: 2025-04-28
draft: false
tags: [前端入门]
category: 前端
image: /source-of-blog/blog-3%20elmUI/elmLogo.png
---
# elmUI介绍
`(给没用过elmUI的小登写的 老登别看啦)`
先给伟大的ElmUI贴个链接
[elm](https://element-plus.org/zh-CN/)
[组件库](https://element-plus.org/zh-CN/component/overview.html)
最新版本全称Element Plus 套用官方说明就是个"基于 Vue 3，面向设计师和开发者的组件库"

也就是说比如这样一个登录按钮![alt text](/source-of-blog/blog-3%20elmUI/image.png)
你不用手搓 直接写入
````html
<button type="primary">登录</button>
````
就行啦 也就是说可以当场找到你想要的组件 当场写入你的前端项目

# 组件介绍
## 内容组件
elmUI中如 `Card`,`Button`, `input`, `icon` 这些 是作为`某一个`页面布局的一部分来使用的
用起来最大的感受就是省时间 舒服 `拆包即用`
组件的拓展性都不错 比如`input`组件
![alt text](/source-of-blog/blog-3%20elmUI/image-1.png)
如图 左边的几个input 像邮箱那里 可以直接在input里添加上右侧的**发送验证码**按钮
右上的是大型文本框 写学期总结的地方()
## 布局(功能)组件
如`el-dialog`(弹窗组件) `el-info`(提示信息功能) `el-message`
这些组件的泛用性个人认为会高于`Button`什么的 而且也更重要
毕竟搓个`button`谁不会 就是懒 搓个`弹窗组件`这种还是很头疼的

刚开始用的时候 特别是`el-dialog` 会让我感觉: 哦 原来这么轻松就能实现这样的布局结构
![一个el-dialog的使用](/source-of-blog/blog-3%20elmUI/image-2.png)
然后我就在我的项目里用了快10个el-dialog 比起只有只有页面的跳转 网页高级了个档次

## 缺点
因为是用来布局的 所以会**附带着elm自己的一套风格**
也就是说 这些组件拿来写写个人项目还是很不赖的，拿来多人开发的正式场景，就容易遇到风格问题
可能UI要求的风格和elm的不兼容 那就用不了了 所以还是别依赖过头了()

# 其他
## bug
### 手动添加elm内置的class名
elmUI有内置的class名以及对应属性 所以不要乱使用可能和`内置class名`**重叠**的`class名`
如下 我由于在一个`<el-upload/>`的最外层额外写了个**class="el-upload"** 导致如下属性侵入
![alt text](/source-of-blog/blog-3%20elmUI/image-class.png)
其中display:inline-flex; 导致此处变为flex布局 故导致如下bug

正常的![正常的](/source-of-blog/blog-3%20elmUI/image-4.png)
在组件最外层多写了`class="el-upload"`![套了层flex](/source-of-blog/blog-3%20elmUI/image-3.png)
### 样式渗透时多加了scoped
elm的组件源码正常我们不会去改 那这时候要是我们想稍微对这组件小改个色 或控个尺寸 咋办呢
这时候就可以在`<style>`里写 来操作
具体操作如下
先F12找到这个组件你要改的那一层的class名
`这里注意 elm的组件基本都好几层 别改错层了`
![alt text](/source-of-blog/blog-3%20elmUI/image-5.png)
如果你有在.vue的style里都加上`scoped`的习惯的话 就单独起一个`<style>` 不加scoped 或者在外层套个`:global()`
如下
![alt text](/source-of-blog/blog-3%20elmUI/image-6.png)

`scoped`这个属性 会让你写的`<style>`只对当前文件起效果 而显然elm组件在其他文件里 所以记得**别加scoped捏**
## 水项目小技巧
因为elmUI够出名 所以**AI**也认识
你用elm组件写项目的话 可以直接叫AI 比如`生成一个ElMessageBox 里面要有账号和密码....` 就能光速水完
哎 这是真无敌了
不过要这样用多了自己再改的不认真 大概率整个项目要变shi山了
## 感受
显然elmUI不是万能的 但是也算是非常实用全能的
用起来让我对一些已有规范的组件有所认识(比如switch组件)![alt text](/source-of-blog/blog-3%20elmUI/image-7.png)
这样的组件虽然html里没有 但是大家都这么叫这个名 都大概是这么个写法

没用过elmUI的 还是很推荐写个人项目时用一用的 既有效率 又能带来提升(使用方法直接看官网文档 链接在本博客最上方 或者直接浏览器搜索elmUI)
