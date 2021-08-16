---
title: Vue3.0新特性
date: 2021-08-16 11:13:48
categories: 前端
tags:
  - Vue
  - Javascript
---

经过了漫长的迭代，Vue 3.0 终于在上 2020-09-18 发布了，带了翻天覆地的变化，使用了 Typescript 进行了大规模的重构，带来了 Composition API RFC 版本，类似 React Hook 一样的写 Vue，可以自定义自己的 hook ，让使用者更加的灵活，接下来总结一下 vue 3.0 带来的部分新特性。

<!-- more -->

1.setup()
2.ref()
3.reactive()
4.isRef()
5.toRefs()
6.computed()
7.watch()
8.LifeCycle Hooks(新的生命周期)
9.Template refs
10.globalProperties
11.Suspense


## Vue2 与 Vue3 的对比

- 用proxy代理替代了defineProperty来实现数据的双向绑定。[*剔除了defineProperty的一些诟病*](/2021/07/23/Vue核心双向绑定原理解析/)
- 对 TypeScript 支持不友好（所有属性都放在了 this 对象上，难以推倒组件的数据类型）
- 大量的 API 挂载在 Vue 对象的原型上，难以实现 TreeShaking。
- 架构层面对跨平台 dom 渲染开发支持不友好
- CompositionAPI。受 ReactHook 启发
- 更方便的支持了 jsx
- Vue 3 的 Template 支持多个根标签，Vue 2 不支持
- 对虚拟 DOM 进行了重写、对模板的编译进行了优化操作...

### 更快

在`Virtual Dom`方面，Vue3 重新审视了`Virtual Dom`，更改了自身对于 `Virtual Dom`的对比算法，在 Vue2 中的`Virtual Dom`每次更新，都会跟上一次缓存的虚拟节点进行全量的对比。**而在 Vue3 中，diff 算法会忽略所有的静态节点，只对有`patch Flag`的动态节点进行对比**，尽管 `JavaScript` 做 `Vdom` 的对比已经非常的快，但是 `patch Flag` 的出现还是让 Vue3 的 `Vdom` 的性能得到了很大的提升，尤其是在针对大组件的时候。

推荐一个网站: [https://vue-next-template-explorer.netlify.app/](https://vue-next-template-explorer.netlify.app/)

#### patch flag

可以看到图中左侧动态绑定 msg 的 span 有一处注释，官方叫它`patch flag`，在与上次虚拟节点进行对比时候，只对比带有`patch flag`的节点，并且可以通过`flag`的信息得知当前节点要对比的具体内容，不同的值对应不同的比对内容，例子中的是 1，代表`TEXT`。

![](/images/screenshot/patch-flag.png)

#### hoistStatic 静态提升

把静态的节点进行提升，可以看到所有的静态 span 都被拿到了渲染函数体外面，也就是说在应用第一次的启动被创建了一次后，之后这些虚拟节点会在每次渲染时候被不停的复用，这样就免去了重复的创建节点，大型应用会受益于这个改动，免去了重复的创建操作，优化了运行时候的内存占用。

![](/images/screenshot/hoistStatic.png)

#### cacheHandlers 事件侦听器缓存

第一张是不使用 `cacheHandlers`，每次渲染更新时，都会对事件进行一次绑定

![](/images/screenshot/cacheHandler1.png)

使用了`cacheHandler` ,第一次更新的时候，会将`onClick`函数的储存位置变成了缓存的形式。也就是说 当你的页面在不断的更新的时候，你的**事件侦听器并不会重复地销毁再创建**，而是以缓存的形式存在，这使 Vue3 在性能方面又有了一个出彩的地方。

> 另外，Vue3 在 @click 中，直接手写内联函数也会被缓存起来，这一点是 react 做不到的。再加上 在 Vue3 中父组件的更新并不会直接触发子组件的更新，使得事件侦听器缓存在组件的层面可以提现出来更高的价值。

#### ssr渲染
当有大量静态的内容时候，这些内容会被当做纯字符串推进一个buffer里面，即使存在动态的绑定，例如图中，会通过模板插值嵌入进去。这样会比通过虚拟dom来渲染的快上很多很多。

当静态内容大到一定量级时候，会用_createStaticVNode方法在客户端去生成一个static node，这些静态node，会被直接innerHtml，就不需要创建对象，然后根据对象渲染。

![](/images/screenshot/ssr.png)

### 更小

之前 vue 的代码，只有一个 vue 对象进来，所有的东西都在 vue 上，这样的话其实所有你没用到的东西也没有办法扔掉，因为它们全都已经被添加到 vue 这个全局对象上了。

vue3 的话，一些不是每个应用都需要的功能，我们就做成了按需引入。用 ES module imports 按需引入，举例来说，内置组件像 keep-alive、transition，指令的配合的运行时比如 v-model、v-for、帮助函数，各种工具函数。比如 async component、使用 mixins、或者是 memoize 都可以做成按需引入。**这配合tree shake使得Vue体积更小**
