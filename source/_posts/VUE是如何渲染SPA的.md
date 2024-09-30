---
title: VUE是如何渲染SPA的
date: 2024-09-28 13:35:52
tags:   
  - Vue
  - SPA
  - 渲染原理
  - 前端开发
---




## 涉及到的知识面

正则表达式，数据劫持，虚拟 DOM



## 简易流程加虚拟 DOM

为了可以介绍清楚整个转化流程，我将拿一个简单地 `.vue` 文件举例子，并把整个渲染流程代码逐个拆分到每个步骤中去。这个例子会涉及响应式和虚拟 DOM。


```vue
<template>
    <h1>{{ title }}</h1>
    <p>{{ content }}</p>
</template>

<script>
export default {
    data() {
        return {
            title: "this is title",
            content: "this is Content",
        }
    }
}
</script>
```

### SFC转为JS 

由于浏览器只能识别 `http | js | css`  文件，所以 `vite` 官方 提供一个`@vitejs/plugin-vue` 组件，可以把这个插件想象成编译器，把 `.vue` 文件转化为 `.js`文件。为了更好的探究实现过程，我决定手写，这时候就涉及到读取vue文件，把内容转为字符串输出。

早期，浏览器是一个沙盒，它不允许我们操作本地文件，通常都是后端处理，前端使用 `fetch` API 或 `XMLHttpRequest` 来发送请求到后端拿到数据。浏览器只允许同源的Ajax操作，如果跨域，就必须使用CORS权限。\
除此之外，还有纯前端操作文件的方法。参考[# 使用File System Access API让浏览器拥有操作本地文件的能力](https://blog.csdn.net/xgangzai/article/details/129605068)。
纯前端方法不是很流行，所以我不做研究。


利用 `webpack` ，相当于是后端的处理方式，由于 ` webpack ` 运行在 ` Nodejs ` 上，所以webpack内部在编译的过程帮我们处理了读取文件的步骤，我们只需配置一个loader，参数是从目标文件拿到的字符串信息，返回一个字符串webpack会生成对应的js文件。在loader中编写我们的处理逻辑。

以下手写webpack-loader ，模拟了`@vitejs/plugin-vue`的编译功能，把vue转化为了js。

```js
const regTemplate = /\<template\>(.+?)\<\/template\>/
const regScript = /\<script\>(.+?)\<\/script\>/
const regFirstSign = /({)/

module.exports = function (source) {
      const _source = source.replace(/[\r\n]/g,'')  
      const template = _source.match(regTemplate)[1]
      const script = _source.match(regScript)[1]
      const finalScript = script.replace(regFirstSign, '$1 template:' + '`' + template + '`' + ',')
      console.log(finalScript)
      return finalScript
}

//finalScript like this 
export default { template:`    <h1>{{ title }}</h1>    <p>{{ content }}</p>`,    data() {        return {            title: "this is 
title",            content: "this is Content",        }    }}
```


### template模板分析，编译

template分为标签，属性，内容。
- 标签有可能是原生的html，也有可能是组件
- 属性也有可能是vue框架属性，自定义属性
vue会对这些模板做过滤，生成AST树。

首先把文件里的template和data解构出来

```js
import App from './App.vue'
console.log(App)

//解构出被loader处理过的App.js
const {template,data: generate } = App

let data  = reactive(generate())
```

然后对模板进行分析编译，编译包括匹配每个标签和内容，然后生成虚拟的DOM。

>这里是一个简易版本，只针对原生标签的innerHtml做了分析。这个 innerHtml 也并非表达式。除此之外，也没有 vue 的特殊的事件属性写法

```js
const regHtml = /\<(.+?)\>\{\{(.+?)\}\}\<\/.+?\>/

const compileTemplate = (template,data)=>{
    //vDOM原来是对象，这里用数组只是为了展示虚拟节点的思想
    const vDOM = []
    const matched = template.match(new RegExp(regHtml,"ig"))
    matched.forEach((tag,index)=>{
        const [,tagName,key] = tag.match(new RegExp(regHtml,"i"))
        
        vDOM[index] = {
            tag:tagName,
            children: data[key.trim()]
        }
    })

    return vDOM
}
```


### render虚拟DOM

初次渲染只需要把虚拟DOM渲染到真实DOM上

```js
const render = (app,template,data)=>{
    state._vDOM = compileTemplate(template,data)

    const fragment  =  document.createDocumentFragment()
    //只做一层的渲染
    state._vDOM.forEach(vnode =>{
        const {tag,children} = vnode

        //并且只做innerText
        const node = document.createElement(tag)
        node.innerText = children

        fragment.appendChild(node)
    })

    app.appendChild(fragment)
}
```


### 数据更新，重新渲染

当数据更新，我们如何追踪这种变化？es6的proxy给了我们答案，通过一种数据劫持的方案，我们可以检测到数据变化，并且做一些额外的（更新视图）的动作。
以下是proxy ，set的handler的实现。
当数据变更时，我们会根据新的data重新编译模板，获取新的虚拟DOM，通过比较新老虚拟DOM，获取发生变化的虚拟DOM，并在真实DOM上重新渲染这部分的值。

> 这种比较涉及了diff算法，这里没有去实现

```js
const update = (template,vDOM,data,value)=>{
    const newVDOM = compileTemplate(template,data)

    newVDOM.forEach( (vnode,index) =>{
        if(vnode.children !== vDOM[index].children){
            patch(value,index)
            vDOM.splice(0, vDOM.length, ...newVDOM); // 直接用新内容替换原有内容
        }
    })
}

const patch = (value,index)=>{
    const childNodes = state._app.children
    childNodes[index].innerText = value
}

const createSetter = ()=>{
    const set = (target,key,value,receiver) =>{
        const oldValue = target[key]
        //一旦执行这一句，那么target的值立马会发生变化，也就是说，下面的代码的target会立马变成新的值。所以update的参数将会是更新后的data
        const res = Reflect.set(target,key,value,receiver)
        if(!Object.prototype.hasOwnProperty.call(target,key)){
            console.log("响应式新增", + value)
            update(state._template,state._vDOM,state._data,value)
        }else{
            if(value !== oldValue){
                console.log("响应式修改", key + " = " +  value)
                update(state._template,state._vDOM,state._data,value)
            }
        }
        return res
    }
    return set
}
const get = createGetter()
const set = createSetter()


const mutableHandler = {
    get,
    set
}
```

### 流程小结

 我们需要拆分一下概念，整个实现涉及到了 vue 的响应式，和虚拟 DOM。
- 响应式，使得开发者不需要手动管理 DOM 的更新，Vue 会根据数据的变化自动重新渲染相关部分，减少了出错的可能性。
- 而虚拟 DOM ，避免了 DOM 的频繁更新，提高性能。
Vue 的整个流程是可以绕过虚拟 DOM 实现的，接下来我将实现一个更加复杂的 vue 渲染 `.vue` 文件的流程。


## 完整流程

以上是 vue 的一个简易版本，由于我认识有限，接下来的内容将会不断更新，以便还原整个 vue 文件运行的流程。

### .Vue 文件转为. Js 文件



### CreateApp 入口函数


### 函件加载




## 虚拟 DOM


## 参考资料

-  [响应式原理](https://jonny-wei.github.io/blog/vue/vue/vue-observer.html#%E5%A6%82%E4%BD%95%E4%BE%A6%E6%B5%8B%E6%95%B0%E6%8D%AE%E7%9A%84%E5%8F%98%E5%8C%96)
- [什么是虚拟 DOM](https://blog.poetries.top/FE-Interview-Questions/principle-docs/comprehensive/07-%E8%99%9A%E6%8B%9FDOM%EF%BC%88%E4%B8%80%EF%BC%89.html#%E4%B8%80%E3%80%81%E4%BB%80%E4%B9%88%E6%98%AF-vdom)
- [# JS实现『从工程化到Vue』【上机题】](https://www.bilibili.com/video/BV1L94y1U73p/?spm_id_from=333.788&vd_source=115cedcdb1996c6483fb453252e441e6)
- [# 【小野森森】虚拟DOM怎么了？【前端基础】](https://www.bilibili.com/video/BV1mK421v7PH?p=4&vd_source=115cedcdb1996c6483fb453252e441e6)

