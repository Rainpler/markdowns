omix2.0是腾讯发布的一个小程序原生框架，主要特性是特性实现小程序的全局状态管理，特具有无状态视图设计、对小程序零入侵、极简的 API的特点。

开源地址：<https://github.com/Tencent/omi/tree/master/packages/omix#%E7%89%B9%E6%80%A7>

## 快速使用
```javascript
npx omi-cli init-x my-app
```

如果你习惯使用 TypeScript，可以使用 TypeScript 的模板：

```javascript
npx omi-cli init-x-ts my-app
```
然后把小程序工作目录设置到 my-app 就可以开始愉快地使用 OMIX 了。

## 快速入门
#### API

- create.Page(store, option) 创建页面， store 从页面注入，可跨页面跨组件共享, 如果 option 定义了 data，store 的 data 会挂载在 this.data.$ 下面
- create.Component(option) 创建组件
- create.Component(store, option) 创建组件页面
- this.store.data 全局 store 和 data，页面和页面所有组件可以拿到， 操作 data 会自动更新视图
>不需要注入 store 的页面或组件用使用Page和Component 构造器, Component 通过 triggerEvent 与上层通讯或与上层的 store 交互

以官方demo提供代码为例，我们来看一下页面的结构。

**store.js**
```javascript
//定义全局 store:
export default {
  data: {
    motto: 'Hello World',
    userInfo: {},
    hasUserInfo: false,
    canIUse: wx.canIUse('button.open-type.getUserInfo'),
    logs: []
  },
  //无脑全部更新，组件或页面不需要声明 use
  //updateAll: true,当为 true 时，无脑全部更新，组件或页面不需要声明 use
  debug: true
}
```
**index.js**
```javascript
import create from '../../utils/create'
import store from '../../store/index'

//获取应用实例
const app = getApp()

create.Page(store, {
  use: [
    'motto',
    'userInfo',
    'newProp'
  ],
  computed: {

  },

  onLoad: function () {

    //store变化监听
    const handler = function (evt) {
      console.log(evt)
    }
    store.onChange(handler)

    //store.offChange(handler)

  },
})
```
我们首先使用的时候需要引入utils中create.js和store文件夹中index.js，然后通过``create.Page(store, option)`` 创建页面，在use中定义的变量，通过``this.store.data.xxx``去引用，并且这些变量是全局共享的，存储在store中。自定义在data中的变量，通过``this.data.$.xxx``去引用。在wxml中，可以通过``{{xxx}}``去引用在use中定义的变量，通过``{{$.xxx}}``去引用在data中定义的变量。

计算属性定义在页面或者组件的 computed 里，如这里的reverseMotto， 它可以直接绑定在 wxml 里，motto 更新会自动更新 reverseMotto 的值。
```javascript
computed: {
    reverseMotto() {
      return this.motto.split('').reverse().join('')
    }
  },
```

这样子，我们就引入了数据的全局管理，可以在跨界面，跨组件的时候，共享store中的数据。

#### 其他

这里需要注意，改变数组的 length 不会触发视图更新，需要使用 size 方法:
```javascript
this.store.data.arr.size(2) //会触发视图更新
this.store.data.arr.length = 2 //不会触发视图更新

this.store.data.arr.push(111) //会触发视图更新
//每个数组的方法都有对应的 pure 前缀方法，比如 purePush、pureShift、purePop 等
this.store.data.arr.purePush(111) //不会触发视图更新

this.store.set(this.store.data, 'newProp', 'newPropVal')  //会触发视图更新
this.store.data.newProp = 'newPropVal' //新增属性不会触发视图更新，必须使用 store.set
```
