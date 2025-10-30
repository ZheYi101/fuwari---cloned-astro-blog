---
title: 自动保存apiKey功能遇初始化异步, 最终确诊水合问题
description: 我在开发中遇到了一个涉及初始化异步的问题, 一开始我认为是链路设计问题; 在前辈的点拨下发现应当是SSR的水合问题
published: 2025-09-21
draft: false
tags: [前端入门,开发经验]
category: 前端
---
# 场景描述
前端开发, 写的是登录页 (不过没有实际后端登录接口)
整个项目没有账号的概念, 是通过在请求的headers里放key来鉴权的
 
所以在登录页的时候只需要输入一行key就行

![alt text](/source-of-blog/blog-6%20yibuDesign/image-1.png)

## 功能设计
因而 我自发的设计了两个功能

1. 缓存key, 并且在每使用时自动调用缓存里的key补全输入框(这样之后要是你还是手动来到login页,就可以直接点击登录不用再输一遍key了)

2. 检测输入框有无内容,无内容时登录按钮 disabled 无法点击

按钮我使用的是`<el-button>` 因而disabled功能就是使用它的 `:disabled` 属性来实现的

# 初版设计
既然要进缓存, 那正常来说就得直接调`localStorage`, 不过这有点不是很优雅, 所以项目里创建了一个
`useApiKey.ts`来把这份缓存单独管理(这样就不用手动调取`localStorage`了) 具体代码如下
```ts
import { useStorage } from "@vueuse/core";

export const API_KEY_STORAGE_KEY = "xxxxxxxxxxxxxxxxx";

export const useApiKey = () => {
  const apiKey = useStorage(API_KEY_STORAGE_KEY, "");

  return apiKey;
};
```
useStorage是vueuse的一个方法, 是创建一个`localStorage`名为 `API_KEY_STORAGE_KEY` 的对象
所以在login页代码里我只用这么写
```html
<el-input
    v-model="apiKey"
/>
<!-- handleConfirmApiKey相当于登录函数(校验下key对不对) -->
<el-button
    :disabled="apiKey.length <= 0"
    @click="handleConfirmAPIKey"
>
    登录
</el-button>
```
```ts
const apiKey = useApiKey();
```
然后在要使用到`Key`的页面里, 也只需要用useApiKey() 这个 **Composable** 就获取到一个优雅的叫做apiKey的变量啦

## 问题
当然, 初版设计一定是有问题的, 不然就不会是初版设计了; 首先这次设计的优点当然是优雅——代码量也少, 使用方式也非常的舒服

问题在于`<el-button>`的 `:disabled`属性, 这个属性传入的是一个 **boolean** 值, 自动更新非常的烂, 于是就导致了一个bug
![alt text](/source-of-blog/blog-6%20yibuDesign/image-2.png)
这里, 我在已有`apiKey`缓存的情况下进入页面(或者原地刷新), 会出现输入框里有字, 但是`<el-button>`的`:disable=true`的情况
大概链路我猜测如下
![alt text](/source-of-blog/blog-6%20yibuDesign/image.png)
大概是这样:
>> 初始化`const apiKey`时`useApiKey`没初始化好, 这一小段真空期里`apiKey`就是个空字符串,

>> 而`<el-button>`的`:disable` 又刚好挑这时候调用了`apiKey`的值，所以就错了;

>> `v-model`没问题则是因为它本身的一个响应式做的太硬了, 同步性太强了;

当然, 这问题是有解决方案的, 如下
```html
<el-input
    v-model="apiKey"
/>
<el-button
    :disabled="isDisabled"
    @click="handleConfirmAPIKey"
>
    登录
</el-button>
```
```ts
const apiKey = useApiKey();

const isDisabled = ref();

onMounted(() => {
  isDisabled.value = apiKey.value.length <= 0;
})

watch(apiKey, () => {
  isDisabled.value = apiKey.value.length <= 0;
});
```

也就是我手动把后续`:disabled`不会自动刷新的问题解决了, 我手动让它进行刷新(`onMounted`触发比apiKey初始化晚),
watch则是让 `isDisabled` 变量与`apiKey`动态同步

效果大概如图描述(蓝色笔画部分)
![alt text](/source-of-blog/blog-6%20yibuDesign/image-3.png)

# 误解问题 | 挖坑
当然, 其实我直接把修改后的初版设计拿来用也是很不赖的, 但是我就是很不爽, 因为太多此一举了, 这解决方案不就是打补丁吗, 并非优秀的设计口牙;

> 这里容我先戛然而止一下(以下非本文章最初的段落, 是我后来修改过的)

初版里的问题我思考后觉得, 其实是链路设计的问题. 一开始我自己也是这么认为的 本篇文章里一开始也是这么写的; 但是文章发出来给别人一看 
被提示了真正的问题所在————ssr渲染模式的水合问题

回头想想, 包括博客在内的话, 我已经接触过三个ssr项目了 所以我觉得我应该是有能力写一篇相关文章的.  
接下来1~2篇博客里, 我会试着去讲述我使用`ssr`的经验, 以及整理 `水合问题` 的相关原理, 解决方案