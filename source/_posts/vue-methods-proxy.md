---
title: 代理模式在vue.js的应用
date: 2018-07-28 12:21:18
tags: vue.js、代理模式
---

## 说明
本文主要讲解vue.js中如何运用代理模式，让代码更纯粹。以及运用类似工厂模式的方法，将各种代理方法通过注册的方式添加到代理工厂里面去，这样各种代理的实现就能独立出来，实现代码的松耦合

## 问题引发的思考
这里选举最简单最常见的例子，在项目中每当涉及删除的时候，都需要先提示，确认删除，然后再删除。这是正常的逻辑。防止用户误操作，因为这些过程都是不可逆的。
列如（这里用了element-ui组件）：
用户点击删除按钮，按钮绑定deleteObject方法
```javascript
 export default {
 // 省略vue文件中的其他代码
 methods:{
   deleteObject (id) {
     this.$confirm('确定删除?', '提示', {
       confirmButtonText: '确定',
       cancelButtonText: '取消',
       type: 'warning',
       lockScroll: false
     }).then(() => {
       // 这里省略删除数据的代码...包括调用后端
       // this.$http.delete(url,{id:id})
     }).catch(() => {
     })
   }
 }
```
刚开始写看起来一切很正常，也没什么毛病。但是，写着写着就会发现，这种代码充斥这整个代码，和业务（例如删除）耦合在一起，而且都是重复的。我希望这个方法更纯粹一点，就是删除逻辑，没有其他不相关的代码。
可能有人觉得，现在怎么就不纯粹了，提示也是应该明确给出的。那我说另外一个需求，**阻止用户连续点击**，有些按钮点击之后是需要发送请求的，不能让用户卡卡卡不停的点击。
还有，比如，用户搜索过程中每次输入结束都调用后台，但是只要用户不停输入就不发送请求，这些方法是需要**debounce、throttle**化处理的。debounce、throttle不懂的话不过多介绍，自行搜索。
以前的做法是定义个变量
```javascript
var isDoing = false
function query {
   if(isDoing){
     return
   }
   isDoing = true
   // 之后的某个时机请求回来或者失败。或者固定时长之后isDoing =false
}
```
现在代码肯定不纯粹了，业务量大，很影响维护。看着也难受

## 如何解决呢
搞过后端（这里指Java）的人肯定知道，用代理模式，JDK、cglib，调用代理方法，代理方法去做删除的提示啊、控制执行、延迟执行等等。当符合条件时，再调用被代理的方法。
**在vuejs中如何实现呢？**
要代理的方法是methods定义的方法，不能影响到vue.js的绑定处理

先说明确需求，有两种使用方式：
1、所有以delete开头的方法，自动运用删除提示代理
2、通过明确的配置，实现代理

两个各有优缺点
1、优点：
   - 无需配置，使用简单
   缺点：
   - 不明确，毕竟JavaScript没有Java的注解
   - 需要代理实现遍历所有的methods方法，做相应名称匹配，很有可能会误操作。
   - 不利于扩展，无法传递给代理方法参数。以后增加各种代理需求，会很复杂
2、优点：
    - 配置明确、一目了然
    - 参数可以自定义
   缺点：
    - 需要配置，有点麻烦
    - 配置会污染vue.js的属性

各方权衡，选择配置的方式，自动遍历代理不利于代码阅读和理解，就像机关暗门藏在里面。当然，没有什么比一目了然的代码更让人喜欢。至于配置会污染vue.js的属性。这里可采用一个命名空间的方式。以公司名称（或缩写）做属性，就算以后和vue冲突（概率很低）,也方便修改

## vue-扩展
vue单文件组件，属性值有：props、data、methods...不列举了。
现在做扩展、自定义一个属性名：xxxConfig( xxx建议取公司名)，这样减少以后冲突的几率
```javascript
export default {
  // 这里是vue 其他属性 data、computed等等
   // xxxConfig下面所有的配置都是自定义的，不是vue的，不要误会，当然，配置需要相应的实现代码来支撑（后面讲）
   xxxConfig:{ // 自定义，vue模板的配置，统一采用xxConfig,采用一个命名空间namespace，以后的所有对vue的扩展，全部放入xxxConfig下面
     methodProxy: { // 对方法的扩展，即方法代理配置
       confirm: { // 代理的类型: 'confirm'（二次确认，多用于删除之类的）。（注意：该代理需要手动注册，这是因为，移动端和pc端实现代理方式有差异）
         methodNames: ['onDeleteObj'] // 需要进行删除代理的方法，每次调用onDeleteObj都会提示是否删除，让用户确认，但是onDeleteObj方法里面并没有提示代码
       },
        params: { // 定制化参数（默认可以不传）,该参数会传给代理函数
         onDeleteObj: { // 代理方法需要参数，名称'onDeleteObj'就是methodNames里面需要代理的方法名。
           title: '提示', // 这里举例是element-ui,$confirm的参数
           message: '确定删除',
           options: {}
        }
      }
    },
    .... // 省略其他对vue的扩展，为以后扩展留下空间
  },
  methods:{
   onDeleteObj (args){
     console.log('调用我之前会弹出提示，但是我并不关心')
     console.log('如果我的参数args是绑定在@click指令里面传入的，那么args依然能正确传入',args)
   }
  }
}
```
就像上面那样，onDeleteObj被代理了，但是它不知道，也无需关心。同时methodProxy下面的methodNames可以同时配置多个方法名称

需求很明确，代码很迷茫。不能光说不做，实现上面的代理需求。同时运用工厂方式，统一收集各种代理

## 实现代理
### 以上面删除提示为例子,实现一个类型为confirm 的代理方法
新建confirm-proxy.js
```js
let defaultParams = {
  message: '确定删除?',
  options: {
    title: '提示',
    confirmButtonText: '确定',
    cancelButtonText: '取消',
    type: 'warning',
    lockScroll: false
  }
}
/**
 * @params fn 被代理的方法
 * @params options 参数
 * @return 代理方法
 */
function confirmProxy (fn, options) {
  let params = Object.assign(defaultParams, options) // 注:参数层级超过两层，需要merge方式
  let target = fn
  return function () { // **注:这里不要用箭头函数，因为你代理的是vue methods里面的方法，代理方法会被vue处理、无法绑定vue this **
    let args = arguments
    // window.app 是全局根实例
    /* window.app = new Vue({
         el: '#app',
         router,
         store,
         render: h => h(App)
      })*/
    // 你也可以用其他的提示方法，这里只是说明
    window.app.$confirm(params.message, params.options).then(() => {
      target.apply(this, args)
    }).catch(() => {
    })
  }
}

export default {
  name: 'confirm',
  fn: confirmProxy
}
```
每个代理都是一个单独的文件（这里 confirm-proxy.js），**代理名称-proxy为文件名**，默认导出代理类型名称和代理方法。
#### confirm代理的运用
自定义vue插件, 一般中大型项目的开发都会有自定义的vue插件，和公司业务、其他框架结合。
这里我只写有关代理的代码，其他省略、
customVuePlugin.js
```js
 import proxyFactory from './proxy-factory.js' // 代理工厂、所有代理注册在里面，稍后讲
 function _addMixin (Vue) {
   const namespace = 'xxxConfig' // 自定义vue扩展的命名属性
   Vue.mixin({
     beforeCreate () {
       // 从配置中读取是否有需要代理的方法,proxyConfig中的属性，参考上述的配置
       let proxyConfig = this.$options[namespace] ? this.$options[namespace].methodProxy : null
       if (!proxyConfig) {
         return
       }
       // 遍历代理配置，去代理需要代理的方法
       Object.keys(proxyConfig).forEach((proxyType) => {
         let configItem = proxyConfig[proxyType]
         let proxyMethod = proxyFactory.getProxyMethod(proxyType) // 从代理工厂中获取相应的代理方法
         let methodNames = configItem.methodNames // 需要代理的函数数组
         if (!Array.isArray(methodNames) || methodNames.length === 0) {
           console.warn(`[gx proxy warn]: methodNames must be array and length > 1`)
           return
         }
         methodNames.forEach((methodName) => {
           let fn = this.$options.methods[methodName]
           if (!fn) {
             console.warn(`[gx proxy warn]: method ${methodName} is not exist`)
             return
           }
           let options = {}
           if (configItem.params && configItem.params[methodName]) { // 配置参数
             options = configItem.params[methodName]
           }
           this.$options.methods[methodName] = proxyMethod(fn, options) // 覆盖原来的方法，因为是beforeCreate生命周期，vue还没有做相应的绑定
         })
       })
     }
   })
 }
 export default {
   install (Vue) {
     _addMixin(Vue)
   }
 }
```
**最为关键的一句proxyFactory.getProxyMethod(proxyType)**
参数proxyType就是confirm代理类型。这句话就是从代理工厂中获取代理类型为'confirm'的代理方法，这里有点绕口
proxy-factory.js
```js
class ProxyFactory {
  constructor () {
    this.proxyConfig = {} // 存放所有代理类型
  }
  /**
   * 注册 代理类型、方法
   * @param type
   * @param fn
   */
  register = (type, fn) => {
    if (this.proxyConfig[type]) {
      throw new Error(`代理类型[${type}]已经存在，不能重复注册，请检查注册代码`)
    }
    this.proxyConfig[type] = fn
  }
  getProxyMethod = (proxyType) => {
    if (!this.proxyConfig[proxyType]) {
      throw new Error(`代理类型配置有误，不存在该代理类型:${proxyType}`)
    }
    return this.proxyConfig[proxyType]
  }
}
export default new ProxyFactory()
```
接下来要做的就是将代理类型comfirm注册到ProxyFactory中去
先思考一个问题，这里的代理类型、代理方法是根据不同业务需要开发的，是可变的。而代理工厂、vue插件中的实现methods代理的代码是不变的，将不可变的代码从中分离出来，进行复用，这不正是软件复用的思想吗。
如果有一些代理类型、vue扩展是通用的那么也可以放在一起。这些固定不变的、通用的代码可以放到基础工程里面去，参考[基础工程](/2018/07/19/build-front-base-project)

### 注册代理类型
最简单方式
```js
import proxyFactory from './proxy-factory.js'
import confirm from './confirm-proxy.js'

proxyFactory.register(confirm.name,confirm.fn)
```
这样上面vue插件中的
let proxyMethod = proxyFactory.getProxyMethod(proxyType) // 从代理工厂中获取相应的代理方法
上面这句话就有了着落
为了便于管理，应该新建一个目录假设叫vue-method-proxy（你可以任意取），专门用来存放各种代理的实现，并提供一个入口index.js

index.js
```js
import confirmProxy from './confirm-proxy.js'
// import xxxProxy from './xxx-proxy.js'

let list = [
  confirmProxy
]
/**
 * 将自定义的代理类型和方法注册到代理工厂
 * @param proxyFactory 定义在基础工程
 */
export function registerTo (proxyFactory) {
  list.forEach((item) => {
    proxyFactory.register(item.name, item.fn)
  })
}
export default {
  registerTo
}
```

**只要用index.js去注册一下就可以，以后再添加新的代理，只要关注新文件和index就可以了**。无需关注太多东西
```js
import methodProxyList from 'vue-method-proxy/index.js'
import proxyFactory from 'proxy-factory.js'
methodProxyList.registerTo(proxyFactory)
```

这样便于管理，以后有新的需求，比如要实现方法的debounce代理
在vue-method-proxy新建一个debounce-proxy.js
```js
/**
 * 对func做延迟触发，防止多次调用，多用于用于搜索请求，事件监听
 * @param func
 * @param delay
 * @returns {Function}
 */
export function debounce (func, delay = 500) {
  let timer
  return function (...args) {
    if (timer) {
      clearTimeout(timer)
    }
    timer = setTimeout(() => {
      func.apply(this, args)
    }, delay)
  }
}

function debounceProxy (fn, options) {
  let params = Object.assign(defaultParams, options)
  return debounce(fn, params.delay)
}
export default {
  name: 'debounce',
  fn: debounceProxy
}
```
将debounce-proxy.js注册到index中去
修改下index.js
```js
import confirmProxy from './confirm-proxy.js'
+import debounceProxy from './debounce-proxy.js'

let list = [
  confirmProxy,
+ debounceProxy
]
/**
 * 将自定义的代理类型和方法注册到代理工厂
 * @param proxyFactory 定义在基础工程
 */
export function registerTo (proxyFactory) {
  list.forEach((item) => {
    proxyFactory.register(item.name, item.fn)
  })
}
export default {
  registerTo
}
```
就可以在你的vue文件中直接使用debounce-proxy
```js
export default {
  gxConfig:{
     methodProxy: { // 故乡云代理配置
       debounce: { // 代理的类型: confirm（二次确认，多用于删除之类的）。（注意：该代理需要手动注册，这是因为，移动端和pc端实现代理方式有差异）
         methodNames: ['queryList'] // queryList方法需要被debounce化
       },
        params: { //
         queryList: {
           delay: 500 // queryList延迟500毫秒执行
        }
      }
    }
  },
  methods: {
    queryList () {
       console.log('我延迟调用了')
    }
  }
}
```
## 总结
对vue的methods代理的扩展就算完成了，这里vue的扩展仅仅对methods做了处理。还可以加入其他的扩展，建议使用单一的属性(我更喜欢叫命名空间)xxxConfig(xxx公司名称，防止以后冲突)。
这篇文章涉及JavaScript代理模式、工厂模式，主要是分离关注点，将不变的代码从可变的分离出去。提高代码复用。
对vue.js的扩展其实只是次要的，不要用什么框架都可以自定义扩展。思想先于代码，不要被固定模式给困住
