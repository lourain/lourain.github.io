---
title: Vue核心双向绑定原理解析
date: 2021-07-23 14:27:48
categories: 前端
tags:
  - Vue
  - Javascript
---

![](/images/screenshot/2772613-b61ffc1f678f6095.webp)

<!-- more -->

## Vue2 双向绑定

如上述示意图，实现这样的功能，主要分为两大块去考虑： 节点树中的代码字符串和 data 中的数据的劫持。

vue.js 是采用的数据劫持结合发布者-订阅者模式的方式,通过 object.defineProperty()来劫持各个属性的 setter/getter
在数据变动时,发布消息给订阅者,触发相应的监听回调

1. 需要**Observe 观察者**的数据对象进行遍历,包括子属性对象的属性,都加上 setter 和 getter,**进行数据拦截**这样的话,
   给这个对象的某个值赋值,就会触发 setter,那么就能监听到数据的变化
   ```js
    class Observe {
    constructor(data = {}) {
        this.data = data
        this.walk(this.data)
        
    }
    walk(data) {
        if(typeof data !== 'object'){
            return
        }
        let keys = Object.keys(data)
        keys.forEach(key => {
            if (typeof data[key] == 'object') {
                this.walk(data[key])
            } else {
                this.defineReactive(data, key, data[key])
            }
        })

    }
    defineReactive(data, key, value) {
        var that = this
       
        let dep = new Dep()
        Object.defineProperty(data, key, {
            configurable: true,
            enumerable: true,
            get() {
                //在compile中已经有被调用了，相当于载入就执行了
                console.log(`${key}被劫持了`);
                console.log(Dep.target);
                
                Dep.target && dep.addSub(Dep.target)

                return value
            },
            set(newVal) {
                value = newVal
                console.log(dep.subs);

                // that.walk(newVal)
                //触发通知、
                dep.notify()
                // window.watcher.update()
                
            }
        })
    }
}
   ```
2. **Compile 解析**解析模版指令,将模版中的**变量替换成数据**,然后初始化渲染页面视图,并将每个**指令对应的节点绑定更新函数**,
   添加**监听数据的订阅者**,一旦数据有变动,收到通知,更新视图
   ```js
        
function compile(node) {
        let childrens = this.toArray(node.childNodes)
        childrens.forEach(child => {
            if (this.isElementNode(child)) {
                this.compileElement(child)

            }
            if (this.isTextNode(child)) {
                this.compileText(child)
            }
            //如果元素节点下面还有子节点，则继续解析
            if (child.childNodes.length > 0) {
                this.compile(child)
            }
        })
    
    }

   ```

3. **Watcher 订阅者**是 observer 和 compile 之间通信的桥梁,主要做的事情是
   1. 在实例化时往属性订阅器(dep)里面添加自己
   2. 自身必须有一个 update()方法
   3. 待属性变动 dep.notice()通知时,能够调用自身的 update()方法,并触发 compile 中绑定的回调
   4. mvvm 作为数据绑定的入口,整合 observer,compile 和 watcher 来监听自己的 model 数据变化,通过 compile 来解析编译模版

```js
//watcher用于关联observer和compile
class Watcher {
    constructor(vm, expr, cb) {
        this.vm = vm
        this.expr = expr
        this.cb = cb
        //把Dep与watch关联起来  把watch实例添加进去
        Dep.target = this
        //存储老值
        this.oldValue = this.getVMvalue(vm, expr)

    }
    //获取vm内数据
    getVMvalue(vm, expr) {
        let data = vm.$data
        expr.split('.').forEach((key) => {
            data = data[key]
        })
        return data
    }
   
    update() {
        let oldValue = this.oldValue
        let newValue = this.getVMvalue(this.vm,this.expr)
        if(newValue !== oldValue){
            this.cb(newValue,oldValue)
        }
    }
}


//建立订阅者模式 便于管理
class Dep {
    constructor(){
        this.subs = []
    }
    
    addSub(sub) {
        this.subs.push(sub)
        Dep.target = null
    }
    //通知触发 告诉订阅者
    notify() {
        this.subs.forEach(sub => {
            //执行sub内的update函数
            sub.update()
        });
    }
}



```

**最终利用 watcher 搭起 observer 和 compile 之间的通信桥梁,达到数据变化->更新视图:视图交互变化->数据 model 变更的双向绑定效果**
最终的实现：[https://github1s.com/lourain/vue/blob/HEAD/vue.js](https://github1s.com/lourain/vue/blob/HEAD/vue.js)

## object.defineProperty 数据绑定的弊端

1. 无法劫持到对象属性的新增或删除
   由于 js 的动态性，可以为对象追加新的属性或者删除其中某个属性，但是这点对经过 Object.defineProperty 方法建立的响应式对象来说，只能**追踪对象已有数据是否被修改，无法追踪新增属性和删除属性**。
2. 不能监听数组的变化（对数组基于下标的修改、对于 .length 修改的监测）
   vue 在实现数组的响应式时，它使用了一些 hack，把无法监听数组的情况通过重写数组的部分方法来实现响应式，这也只限制在数组的 push/pop/shift/unshift/splice/sort/reverse 七个方法，其他数组方法及数组的使用则无法检测到，

## Vue3 双向绑定

了解到 Object.defineProperty 的一些不足，vue3 用[Proxy](https://developer.mozilla.org/zh-CN/docs/orphaned/Web/JavaScript/Reference/Global_Objects/Proxy)属性来替代之，配合[Reflect](https://developer.mozilla.org/zh-CN/docs/orphaned/Web/JavaScript/Reference/Global_Objects/Reflect)来对数据的劫持更改

Proxy 代理的特点:
1. proxy 直接代理的是整个对象而非对象属性,
2. proxy 的代理针对的是整个对象而不是像 object.defineProperty 针对某个属性,
3. 只需要做一层代理就可以监听同级结构下的所有属性变化,包括新增的属性和删除的属性

proxy 代理身上定义的方法共有 13 种,其中我们**最常用的就是 set 和 get,但是他本身还有其他的 13 种方法**
