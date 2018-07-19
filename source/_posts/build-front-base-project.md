---
title: 搭建前端基础工程
date: 2018-07-19 11:06:07
tags: [前端、基础工程]
---
## 说明
随着移动端webview渲染能力越来越强,移动h5混合开发越来越流行，很多企业选择h5来开发移动端和pc web端。理所当然的是两个甚至更多的工程(mobile,pc,...)。
每个工程都有通用模块，用于存放公司自己封装的基础组件、通用函数。
#### 伴随的问题
1. 这些通用代码都是类似的，甚至相同的，当其中一处需要修改，另外几处也需要跟着修改。修改成本大了，而且容易出错（我经历的第一家公司就是通过复制，拷贝的方式将pc工程改名为mobile工程、pad工程。 ಠ╭╮ಠ）
2. 同事可以很随意修改你封装的代码，仅仅是为了满足他们的需求，而不考虑合理性，这是我不希望看到的，对于封装好的东西，不应该任何人都能轻易修改。所以我希望这些封装好的代码无法被他们修改，只能被使用


#### 本文就来探讨如何搭建前端基础工程，封装移动端、pc端工程通用代码、组件，及便捷的开发调试

### 项目说明
**移动端，和PC端两个项目。webpack打包、vue全家桶**
*这是移动端和pc端项目截图，vue-cli做的修改，大同小异*`
移动端:(pc端大同小异，基于vue-cli)
{% asset_img mobile-project.png 移动端项目截图 %}
src 目录下面的**base-components**、**js**封装的基础组件和常用方法，属于项目通用，移动端、pc端一式两份，很类似，每次修改都要修改两处
{% asset_img src-dir.png 移动端项目截图 %}

### 搭建基础工程，封装通用代码
#### 思路：
   现在由A、B两个工程对应公司的pc、移动端，现在新建一个工程C,将A、B通用的代码提出到C,然后把C发布到npm仓库(不想使用的，可以发布到公司的git上面，直接通过npm + 'git地址安装'，或者先配置到package.json中，然后npm install)

### 新建基础工程项目
   由于是基础工程，不要有复杂的配置、环境搭建。
   使用[rollup](https://rollupjs.org/guide/en "rollup"),[中文点这里](https://rollup.docschina.org/#introduction "rollup中文")打包（简单实用，可快速入门），配置很简单稍后说，[现成的rollup起步](https://github.com/rollup/rollup-starter-lib "rollup-starter-lib")

 ####  先给出基础项目结构
 这是刚刚搭建的基础工程，内部的封装正在迁移，内容不是特别多，适合学习
{% asset_img base-dir.png 移动端项目截图 %}
 #### 入口文件内容
 **index.js**
 ```javascript
    import util from './util/index.js'
    import ModuleControl from '../packages/business/module-control/index.js'

    const components = [
      ModuleControl
    ]
    const install = function (Vue, opts = {}) {
      components.map(component => {
        Vue.component(component.name, component)
      })
      Vue.prototype.$Util = util // 如果是Vue安装，直接挂在到vue原型上面，直接this.$util.xxx 使用库函数
    }

    // const gxyun = {
    //   version: '1.0.0',
    //   install,
    //   ModuleControl
    // }
    module.exports = {
      version: '1.0.0',
      install,
      util,
      ModuleControl
    }

    module.exports.default = module.exports
 ```
 其中 **util/index.js**
  ```javascript
  import cookie from './cookie.js'
  import * as device from './device.js'
  import * as dataFormat from './dataFormat.js'
  import * as date from './date.js'
  import * as dom from './dom.js'
  import * as func from './function.js'
  import * as obj from './object.js'
  import * as regular from './regular.js'
  export default {
    cookie,
    dataFormat,
    date,
    device,
    dom,
    func,
    obj,
    regular,
  }
  ```

### 打包
 **最简单的配置**
  ```javascript rollup.config.js
  const vue = require('rollup-plugin-vue2')
  const css = require('rollup-plugin-css-only')
  const babel = require('rollup-plugin-babel')
  // import { uglify } from 'rollup-plugin-uglify';

  export default [{
    input: 'src/index.js',
    plugins: [
      vue(),
      css(),
      babel({
        exclude: 'node_modules/**'
      })
    ],
    output: [{
      file: 'dist/gxyun-base.js',
      format: 'umd', // 通用模块
      name: 'xxxBase',
    }, {
      file: 'dist/xxx-base.esm.js',
      format: 'es', // es模块
      name: 'xxxBase',
    }, {
      file: 'dist/xxx-base.min.js',
      format: 'umd', // 通用模块压缩（这里未使用uglify）
      name: 'xxxBase',
    }]
  }]
  ```
  **需要用的的依赖**
  ```javascript  package.json
  "devDependencies": {
      "babel-core": "^6.26.3",
      "babel-plugin-external-helpers": "^6.22.0",
      "babel-preset-env": "^1.7.0",
      "babel-preset-es2015": "^6.24.1",
      "babel-preset-stage-2": "^6.24.1",
      "rollup": "^0.62.0",
      "rollup-plugin-babel": "^3.0.7",
      "rollup-plugin-buble": "^0.19.2",
      "rollup-plugin-commonjs": "^9.1.3",
      "rollup-plugin-css-only": "^0.4.0",
      "rollup-plugin-livereload": "^0.6.0",
      "rollup-plugin-node-resolve": "^3.3.0",
      "rollup-plugin-vue2": "^0.8.0",
      "uglify-js": "^3.4.4",
      "zlib": "^1.0.5",
      "gitbook-cli": "^2.3.2"
    },
  ```
  接下来执行
  ``` npm
   rollup -c  // 打包代码，dist目录下面会生成 gxyun-base.js，xxx-base.esm.js代码
   rollup -c -w // 启动监听、源码文件修改，会及时打包
  ```
  现在需要修改package.json文件
  将你的入口指向 打包后的文件
  ```javascript  package.json
    "main": "dist/gxyun-base.js", // 主入口 require('xxx.js')
    "module": "dist/gxyun-base.esm.js", // 支持es module
    "browser": "dist/gxyun-base.min.js", // 浏览器环境加载的入口
  ```


  多扯一句，这些命令配置到package.json scripts标签里面去，通过npm run执行相关命令
  ```javascript                                package.json
     "scripts": {
        "build": "node build/build.js",
        "build:config": "rollup -c",
        "dev": "rollup -c -w",
        "test": "node test/test.js",
        "pretest": "npm run build",
        "doc": "cd doc && gitbook install && gitbook serve",
        "build:doc": "sh build/doc.sh"
      },
  ```

### 至此，一个基础工程就建立好了，你只需要往。src、packages目录下面放置你的组件、方法
    其实模块化开发好处就是，模块之间引用，文件之间的放置没有固定成规，根据自己的需要即可，但是入口文件index.js是必须的


### 如何通过npm安装基础工程
    现在需要提到package.json文件，需要你对npm有了解，
   ```
    "name": "xxx-front-base", // 模块名称，很重要，最后通过npm xxx-front-base
     "repository": { // 仓库地址，如果是发布到npm，npm也是通过该地址去拉取你的蛋，不是，打错了，是代码
        "type": "git",
        "url": "此处是github项目地址，或者自己公司仓库地址，外网可以访问"
      },
   ```

#### 安装
  **如何引入该工程**
  回到mobile工程
  如果你的项目发布到了npm上面去（如果是公司业务项目，很多都是不会发布上去的）
  那么问题就简单了
  ``` npm
    npm install xxx-front-base
  ```
  **否则**
  在mobile项目中的package.json 文件中加入
  ```javascript
    "dependencies":{
      ... // 省略
      "gxyun-front-base": "git+http://xxxxxxx/taild/xxx-front-base.git", // 公司内部git私服地址
    }
    在执行
    npm install
  ```
  打开node_modules目录发现xxx-front-base模块已经安装上了

#### 使用
   ```javascript
   // 全局引入
   import xxx from 'xxx-front-base'
   Vue.use(xxx)
   // 部分引入
   import {util, ModuleControl} from 'xxx-front-base'
   Vue.use(ModuleControl)
   util.xxxx.xxx()
   ```

### 开发调试
   到此，项目搭建使用已经OK，但是有个问题，如何调试，基于base工程编写单元测试，集成现在的测试框架当然ok。巴特，如果是和业务相耦合的业务组件。依赖于公司环境的代码呢
   难道要我，每次修改完毕，打包基础工程，然后npm install一下吗？,而且很多代码都已经在项目中使用了，现成的测试环境，不高兴用测试框架，肿么办？？
   {% asset_img black-question.png %}

#### npm link
   本地垮工程，引入依赖，无需反复构建安装
   **第一步:**在base工程目录下，执行
   ```
      npm link
   ```
   会读取package.json 中的"name"（假如叫"xxx-front-base"）名称作为模块，安装到本地全局node_modules下面
  **第二步:** 在mobile工程下面执行
  ```
     npm link xxx-front-base(base工程的名称)
  ```
  执行完毕, 打开mobile工程的node_modules目录找到 xxx-front-base，你会发现 xxx-front-base与众不同。
  没错，这里的xxx-front-base不在指向npm install的代码库了，而是指向我本地的base工程
  {% asset_img npm-link-mark.png %}

#### 调试
   mobile是通过webpack构建的，有热部署。
   当我修改base工程里面的代码时候。base工程也有热部署，rollup自动构建。mobile工程webpack监听到xxx-front-base有所变化，也会重载，无需重新安装
   是不是很方便，hahhhhh


http://192.168.5.5:10080/taild/gxyun-front-base.git

