---
title: 苦战之登录页的自动保存apiKey功能略有设计感
description: 我在开发中遇到了一个涉及初始化异步的问题, 略有设计感, 拿出来聊聊
published: 2025-09-21
draft: false
tags: [前端入门,开发经验]
category: 前端
image: /source-of-blog/blog-6%20yibuDesign/fengmian.webp
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

# 终版设计
当然, 其实我直接把修改后的初版设计拿来用也是很不赖的, 但是我就是很不爽, 因为太多此一举了, 这解决方案不就是打补丁吗, 并非优秀的设计口牙;

初版里的问题我思考后觉得, 其实是链路设计的问题
初版里我是

>> 初始化时: 
>>1. useApiKey → apiKey → 输入框;
>>2. useApiKey → apiKey → 按钮:disabled

>>后续: 
>>1. 输入框 → apiKey   
>>2. apiKey → useApiKey
>>3. apiKey → 按钮:disabled

后续的链路是没问题的, 但初始化的这两条链路; 有点奇怪不是么? 为什么我的起点一定得是这个useApiKey().

我应当在`apiKey`初始化时就让它带上正确的值, 而不是调用useApiKey()给到`apiKey`,
 这样子会让`apiKey`在刚刚初始化好,而`useApiKey()`没初始化好的一段**真空期**里, 值是错的.

`useApiKey()`应当只接受从 `apiKey`到它的**单链路**
所以我设计了新的链路

>> 初始化时: 
>>1. apiKey(直接调取**localStorage**) → 输入框;
>>2. apiKey → 按钮:disabled

>>后续: 
>>1. 输入框 → apiKey   
>>2. apiKey → useApiKey
>>3. apiKey → 按钮:disabled

不过其实写到最后我已经懒得维护这个`useApiKey()`了, 主要在其他地方也用处不大, 最后索性删了(当然按我这个设计来维护它是没问题的)

最后再展示下我的最终代码

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
// 写成如下的话, 用pnpm run dev跑会500 localStorage is not defined; 具体好像是ssr和ssg相关的原因, 但是背后的道理我并不是很理解, 此处不讲
// const apiKey = ref<string>(localStorage.getItem(API_KEY_STORAGE_KEY) || "");

const apiKey = ref<string>("");

onMounted(() => {
  apiKey.value = localStorage.getItem(API_KEY_STORAGE_KEY) || "";
})

watch(apiKey, (newVal) => {
  if(newVal) {
    localStorage.setItem(API_KEY_STORAGE_KEY, newVal);//这里还想维护useApiKey的化, 就改成给useApiKey赋值
  }
})
```

# 结语
以上, 大概是对`初始化异步`的一次思考;
说实话其实感觉要不是`<el-button>`的:disabled属性传入参数不能是ref, 同步性不够强, 也不会有这个bug我也不会整半天了

element-plus怎么这么坏啊!