---
title: 一次水合问题的修复经验 (于SSR篇1写完后大改了后半段)
description: 我在开发中遇到了一个涉及初始化异步的问题, 一开始我认为是链路设计问题; 在前辈的点拨下发现应当是SSR的水合问题
published: 2025-09-21
draft: false
tags: [前端入门,开发经验]
category: 前端
---

>本文一开始写的有问题, 但是因为思绪混乱, 短时间内没改好;
>后来在`2025-11-09`的[SSR篇1-从认识渲染方式到简单的SSR水合问题](https://zheyi.in/posts/%E5%89%8D%E7%AB%AF/ssr1/)发布后,
>
>重新大改了一次, 可以把本文视作一个`水合问题`的修复例子

# 场景描述
前端开发, 写的是登录页 (不过没有实际后端登录接口)
整个项目没有账号的概念, 是通过在请求的headers里放key来鉴权的

所以在登录页的时候只需要输入一行key就行, 如下图

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
// use-api-key.ts
import { useStorage } from "@vueuse/core";

export const API_KEY_STORAGE_KEY = "xxxxxxxxxxxxxxxxx";

export const useApiKey = () => {
  const apiKey = useStorage(API_KEY_STORAGE_KEY, "");

  return apiKey;
};
```
useStorage是vueuse的一个方法, 是创建一个`localStorage`名为 `API_KEY_STORAGE_KEY` 的对象
所以在login页代码里我只用这么写
```vue
<!-- login.vue -->
<template>
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
</template>

<script setup lang="ts">
const apiKey = useApiKey();
</script>
```

然后在要使用到`Key`的页面里, 也只需要用useApiKey() 这个 **Composable** 就获取到一个优雅的叫做apiKey的变量啦

## 问题
当然, 初版设计一定是有问题的, 不然就不会是初版设计了; 首先这次设计的优点当然是优雅——代码量也少, 使用方式也非常的舒服

问题在于`<el-button>`的 `:disabled`属性, 这个属性传入的是一个 **boolean** 值, 自动更新表现不理想, 出现了如下bug
![alt text](/source-of-blog/blog-6%20yibuDesign/image-2.png)
这里, 我在已有`apiKey`缓存的情况下进入页面(或者原地刷新), 会出现输入框里有字, 但是`<el-button>`的`:disable=true`的情况

# 解决方案
我一开始想了一个简单粗暴的解决方案, 如下

## 简单粗暴的方案
```vue
<!-- login.vue -->
<template>
  <el-input
      v-model="apiKey"
  />
  <el-button
      :disabled="isDisabled"
      @click="handleConfirmAPIKey"
  >
      登录
  </el-button>
</template>

<script setup lang="ts">
const apiKey = useApiKey();

const isDisabled = ref();

onMounted(() => {
  isDisabled.value = apiKey.value.length <= 0;
})

watch(apiKey, () => {
  isDisabled.value = apiKey.value.length <= 0;
});
</script>
```

也就是我手动把后续`:disabled`不会自动刷新的问题解决了, 我手动让它进行刷新(`onMounted`触发比apiKey初始化晚),
watch则是让 `isDisabled` 变量与`apiKey`动态同步

## 根本解决 (25-11-11撰写)
在前辈的提示下, 我打开了控制台发现了如下报错![alt text](/source-of-blog/blog-6%20yibuDesign/image.png)
让我知道这原来是个`水合问题`

问题为: `服务端渲染`时认为按钮是 disabled=true; (因为)
但客户端（浏览器）执行逻辑时认为按钮 disabled=false；
两边渲染结果不一致, 所以被认为是水合问题 就给我弹报错了

这里我给代码修改下进行测试
```ts
// use-api-key.ts
import { useStorage } from "@vueuse/core";

export const useApiKey = () => {
  const apiKey = useStorage<string>(API_KEY_STORAGE_KEY, "", undefined);
  console.log("渲染useApiKey", apiKey.value ? "apiKey有值" : "apiKey无值");
  return apiKey;
};
```
```vue
// login.vue
<!-- 省略template部分 -->
<script setup>
import { useApiKey } from "~/composables/use-api-key";
// 省略其他代码
const apiKey = useApiKey();

onMounted(() => {
  console.log("login页面的onMounted触发", apiKey.value ? "apiKey有值" : "apiKey无值");
});
</script>

```
![alt text](/source-of-blog/blog-6%20yibuDesign/image-3.png)
我们就可以直观的明白, 在SSR服务端渲染阶段没获取到localStorage里的apiKey, 而在客户端渲染时可以正常获取到apiKey;

另一方面, 我们可以知道, 时序来说是
>**服务端渲染 → 客户端渲染 → 页面挂载完成(onMounted触发)**

关于修复呢, 这里[VueUse](https://vueuse.org/core/useStorage/#usage)提供了一个字段, `initOnMounted`,

这里我们来测试下把useApiKey里的useStorage加上`initOnMounted: true`后, 会怎么样
```ts
// use-api-key.ts
import { useStorage } from "@vueuse/core";

export const useApiKey = () => {
  const apiKey = useStorage<string>(API_KEY_STORAGE_KEY, "", undefined, { initOnMounted: true });
  console.log("渲染useApiKey", apiKey.value ? "apiKey有值" : "apiKey无值");
  return apiKey;
};
```
![alt text](/source-of-blog/blog-6%20yibuDesign/image-4.png)

如图, useApiKey在客户端渲染的时候也没有拿到apiKey的值; 也就是说这个字段的作用是让 useStorage在页面挂载(Mounted)之后才获取值

这样, `服务端渲染`和`客户端渲染` 的apiKey就都为空字符串, `:disabled`也都是true了, 于是没有了水合问题, 程序正常执行;

这样按钮的是否disabled就变成了

> 初始一定是:disabled: true, 按钮无法点击; 在页面挂载后判断本地localStorage里是否有存储key, 有就让:disabled变为false

逻辑很清晰, 代码也很优雅 万事大吉咯