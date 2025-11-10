---
title: SSR篇1-从认识渲染方式到简单的SSR水合问题
description: 终于挤出时间写篇博客了, 后续大概率会更SSR-2 ,3 (吧)
published: 2025-11-09
draft: false
tags: [SSR]
category: 前端
image: /source-of-blog/blog-7 ssr1/137138579_p0_master1200.webp
---

### 唠叨

上一篇博客 (9月末) 的时候, 本人在写Nuxt 操作localstorage的时候遇到了一个问题, 起初以为是代码设计的问题, 后来发现是需要避免的 `水合(Hydration)问题`。
当时其实就挺想写一篇文章的, 不过无奈现实生活的繁忙 以及才疏学浅, ssr的水太深 得花大精力研究我才能写出一篇文章。就拖到现在才写一小篇。悲

一开始我研究 `水合问题` 我想着就解决水合问题是什么就行了, 然后随便总结几个可能触发水合问题的情况。 后来再看, 水合问题是 `SSR(Server Side Rendering)` 和 `SSG(Static Side Generation)` 两种 `渲染方式(Rendering Patterns)` 共有的一种问题, 那SSR和SSG是什么我也得理解是什么, 不然无法理解为什么要水合, 为什么水合会有问题, 水合到底是一个怎么样的操作。 那要理解SSR和SSG, 我又得知道渲染方式是什么 现在大概有哪些渲染方式......

所以那也只能原谅我了吧, 原谅我拖了这么久才写出这一小篇文章

不过虽然我好像关于SSR和水合问题没多少经验, 但回头一算我竟发现自己已经接触了三个带SSR的项目 (算上我的博客,它是Astro的孤岛渲染方式, 我对博客也改造过)。 那看来我应该还是有底气写好文章的吧。

# 概念引入
从知识体系的层层递进来说吧, 我应当先说什么是渲染方式再说下什么是SSR再说什么是水合问题...

不过这可真是条长路, 那就先非常笼统的讲下几个核心名次概念(`渲染方式`, `SSR`), 再容我用简单的例子来让大家基本的理解理解SSR和水合问题,能明白几个简单的水合问题情况, 之后再谈对SSR什么的慢慢理解吧。

## 从渲染方式说起吧
`SSR —— Server Side Rendering —— 服务端渲染`, 是一种渲染方式, 渲染方式描述的是: 浏览器是如何把你写的前端代码渲染成网页
### 静态渲染
比如你直接写个`Hello.html`文件 然后跑起来, 那么采用的就是`Static WebSites——静态渲染`, 就啥也不操作 纯纯把你的`Hello.html`搬到浏览器上让浏览器把他渲染出来就完成了。
那么如果你想要分页面, 你就应该写 `page1.html` ,`page2.html` 然后通过切换访问的文件来切换页面

### 单页应用
相对应的比如说我们直接用`npm create vue@latest` 创建一个新的Vue项目, 那么它的渲染方式默认就是 `SPA —— Single Page Application —— 单页应用`。

在一个SPA的前端项目里, 页面的概念不再是单纯使用page1 2 3文件来定义(虽然很可能对于开发者来说, 你还是分了个Page文件夹, 然后每一页对应一个vue文件); 在`vue`里 所有的页面都跑在一个 id="app" 的div里, 切换页面仅仅是通过用`js操作`, 把名为app的这个div里的内容进行改变罢了

# 服务端渲染概念
通过上面两种大家熟悉但是不知道名字的渲染方式, 相信大家已经对于浏览器`渲染`这一操作有了一定的理解了吧。 接下来来到重头戏SSR渲染方式了

## 第一步-获取静态文件
那么经常改造自己博客的朋友们都知道, 一般来首博客都是采用 ssr来渲染的; 它的具体渲染逻辑
简单来说就是 **在服务器上预先把页面渲染成完整的 HTML 字符串，然后返回给浏览器**
相比于`SPA`呢, 比如说如果你的博客采用SPA, 那么在用户进入博客的时候, 服务器传给用户的会是一个
`<div id="app"></div>` 的空壳, 然后浏览器再运行服务器传来的js代码, 把`<div id="app"></div>` 里填充上内容;

而如果用SSR, 那么服务器会直接把完整的**预先**渲染好的博客首页内容以**带内容的HTML文件**的形式传过来, 用户不用再去渲染; 也就是说
> **首屏加载速度** 相比起来会非常的快, 类比餐饮就相当于给了你**预制菜**, 而SPA是~~搞了半天还得自己煮~~;

## 第二步-水合
到这里你大抵会觉得 SSR 渲染真是太棒了吧;

但是呢, 你是否有察觉到, 我刚刚讲的SSR的渲染步骤并不足以让一个网站完整可用?

刚刚讲的部分 仅仅包括了 **HTML**的渲染, 还没提到 **JS** 应该怎么办呢;
仅仅用着服务端传来的一个静态HTML文件, 那怎么使用前端五花八门的api。 肯定得想办法把javascript和对应的HTML绑定上;

比如说, 我写了一个登录页, 通过一开始服务端传来的HTML我当然可以把登录按钮什么的渲染出来, 但是显然没有js的话, 这个按钮就是一个空壳, 没法实际登录的;
我们得 给它绑定上 **@click=login()** 以及 login函数的具体实现, 两段js代码, 这样页面才能正常运行; 这个把js绑定到对应html元素的操作 就叫做 `水合(hydration)`

## 第三步-客户端执行

最后就是在客户端正常运行了 同SPA

# 实例讲解及水合问题
上面我讲归讲了, 但是实际上肯定还是难以理解的, 以及有很多细节只是看概念也根本懂不了, 下面我来实例讲解下。

虽然普遍来说大家通过博客接触到ssr会多一点, 不过博客的ssr用的框架各不相同, 也不方便我进行实例的书写; 所以这里我用`Nuxt`框架来讲解

> 除了Nuxt外, 还有React的Next.js, hexo(搭博客的那个), astro框架(群岛渲染) 比较常见且知名

这里你也不用具体了解Nuxt是什么, 你只要知道他是基于Vue的, 同时又支持SSR渲染就ok了; 不过处于尊重还是贴个中文官网链接

[Nuxt中文官网](https://nuxtjs.org.cn/docs/4.x/getting-started/introduction)

```cmd
npm create nuxt@latest ssrNuxtProject
```
这样我们构建一个Nuxt新项目, 你不手动改配置的话那就是ssr渲染

我们来模拟一个key验证的场景

首先对于`第一步-获取静态文件`, 在自行书写一定的html模拟一个简单的页面后, 你应该也无法直观的感受到性能的提升(不过你估计可以感受到SSR项目构建的是真的慢)
想要体验性能提升的话还是去找找一些常规SSR网站, 或者听听一些框架吹的牛吧

以下来讲下SSR使用过程中的水合问题
## localStorage无法调用
首先我们假设已经在localStorage(即浏览器缓存) 中存好了一个 key为**API_KEY**的键值对
```ts
<script setup lang="ts">
const key = localStorage.getItem("API_KEY")
</script>
```
那么请问, 这么写在ssr渲染的情况下能正常运行吗?
并不能 我们会收获如下报错
![alt text](/source-of-blog/blog-7%20ssr1/image.png)

这里我们会报错; 原因是在 **第二步-服务端水合阶段出了问题**
因为在**服务端上并没有localStorage这个api** , localStorage是浏览器相关的操作，是用来调用浏览器缓存的;

那么这样会报错吗
```ts
import { useKey } from "~/composables/use-key";
const key = useKey()
//下为use-key.ts
export const useKey = () => {
  const key = localStorage.getItem("API_KEY")
  return key;
};

```
也是会的, 也就是即使是import操作, 也会在服务端水合的时候执行

正确的不报错操作应该是如下
```ts
function getKey() {
	return ...
}
onMounted(()=> {
	getKey();
})
```
如上报错的原因均为服务端水合时没有localStorage这个api导致的
而正确写法中, onMounted是会在`第三步-客户端正常运行` 中触发的(这里是`onMounted`的性质, 会在页面挂载后运行)

## 无法打请求
这个就同理了, 在服务端水合的时候是不能打请求的,
我们写前端的时候很经常直接把请求变量裸丢着, 比如
```ts
const res = fetch('https://api.example.com/login', { method: 'post' });
//下略对res一通操作
```
那这在ssr里就不能直接写了

那咋办呢? 哎, 像Nuxt框架里就有一个东西叫`<client-only>` 如下
```html
<client-only>
    <!-- 内容物 -->
</client-only>
```
你可以把要打请求的部分封装成一个组件, 然后填到`<client-only>`里, 这个元素的意思是只会在 第三步-客户端使用 时渲染相关的玩意, 那自然就不会有水合问题了

# 尾声
其实我还有好多想写的, 像是在我的博客astro项目里见到的如下神秘写法
```astro
---

import { siteConfig } from "../config";
---

<div id="config-carrier" data-hue={siteConfig.themeColor.hue} data-lightDarkMode={siteConfig.lightDarkMode.defaultMode}>
</div>
```
一开始我觉得有点酷, 仔细想了想又觉得没必要 知晓实际作用后只剩感慨(卖关子.jpg)

碍于时间以及篇幅, 这里就先停笔了 希望你们很快就能看到我的`SSR篇2`文章捏
### 参考
[10 Rendering Patterns for Web Apps](https://www.youtube.com/watch?v=Dkx5ydvtpCA)

Bing搜索

ChatGpt

我的项目代码

折乙先生的大脑