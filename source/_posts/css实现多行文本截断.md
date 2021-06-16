---
title: css实现多行文本截断
date: 2021-06-09 11:56:02
categories: 前端
tags:
  - IE
  - CSS
  - Javascript
---

单行文本的末尾的省略号很好实现，但是多行文本的末尾的省略号也是一个很常见的需求。这里主要探讨一下兼容 IE 下的一些意想不到的奇淫技巧。

<!-- more -->

## 主流浏览器实现

假设有这样的一个 html 结构

```html
<div class="text">
  浮动元素是如何定位的
  正如我们前面提到的那样，当一个元素浮动之后，它会被移出正常的文档流，然后向左或者向右平移，一直平移直到碰到了所处的容器的边框，或者碰到另外一个浮动的元素。
</div>
```

多行文本超出省略大家应该很熟悉这个了吧，主要用到用到 line-clamp，关键样式如下

```css
.text {
  display: -webkit-box;
  -webkit-line-clamp: 3;
  -webkit-box-orient: vertical;
  overflow: hidden;
}
```

![](/images/screenshot/640.png)

## IE 浏览器的实现

手动写一个“省略号”，放在文字的前面

```html
<p>
  <span class="ellipsis" v-show="ellipsis">...</span>
  eum veldoloremque eum veldoleum veldoloremque eum veldoleum veldoloremque eum
  veldoleum veldoloremque eum veldoleum veldoloremque eum veldoleum
  veldoloremque eum veldoleum veldoloremque eum veldoleum veldoloremque eum
  veldoleum veldoloremque eum veldoleum veldoloremque eum veldoleum
  veldoloremque eum veldoleum veldoloremque eum veldol
</p>
```

给 ellipsis 加上**右浮动 float**，这就是一个**文本环绕效果**,没错千万不要以为浮动已经是过去式了，具体的场景还是很有用的

```css
.ellipsis {
  float: right;
}
```

图中蓝色的就是
![](/images/screenshot/微信截图\_20210609142628.pn)

但是我们需要放在右下角，我们试一下 **margin-top**

![](/images/screenshot/微信截图\_20210609143252.pn)
可以看到，虽然省略号的蓝色方块到了右下角，但是文本却没有环绕按钮上方的空间，空出了一大截，无能为力了吗？

我们可以利用一下**clear**，[MDN](https://developer.mozilla.org/zh-CN/docs/Web/CSS/clear)上这么说的：

> **clear**
>
> 指定一个元素是否必须移动(清除浮动后)到在它之前的浮动元素下面。clear 属性适用于浮动和非浮动元素。

我们可以在 ellipsis 之前创建一个伪元素右浮动到最右边，给 ellipsis 加上**clear:both** 就可以到最下面了，只要这个伪元素高度合适

```css
p::before {
  float: right;
  width: 2px;
  height: 100%;
  margin-top: -20px; /* 预留出省略号的高度 */
  content: "";
  background: red;
}
.ellipsis {
  clear: both;
}
```

![](/images/screenshot/微信截图_20210609142628.png)

发现好像没有生效，可能是 height:100%并没有生效，我们可以给父盒子加上一个高度，或者父盒子打开弹性布局 display:flex，使子元素动态获取高度

```html
<div style="display:flex;">
    <p>
    <span class="ellipsis">...</span>
    eum veldoloremque eum veldoleum veldoloremque eum veldoleum veldoloremque eum
    veldoleum veldoloremque eum veldoleum veldoloremque eum veldoleum
    veldoloremque eum veldoleum veldoloremque eum veldoleum veldoloremque eum
    veldoleum veldoloremque eum veldoleum veldoloremque eum veldoleum
    veldoloremque eum veldoleum veldoloremque eum veldol
</div>
</p>

```

![](/images/screenshot/微信截图_20210609153310.png)

### 文本行数的判断

上面的交互已经基本满足要求了，但是还是会有问题。比如当**文本较少**时，此时是没有发生截断。

```js
if (el.scrollHeight > el.clientHeight) {
  // 文本超出了
  el.classList.add("trunk");
}
```

参考：[CSS 实现多行文本“展开收起”](https://juejin.cn/post/6963904955262435336)
