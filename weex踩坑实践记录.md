### 技术概述

weex是阿里开源的一套构建高性能移动界面的原生跨平台技术框架，它的上层由Vue，Rax（非常类似React的开发框架）实现数据驱动，底层由iOS，Android实现render engine来驱动界面的最终落地。类比React Native它的优势在于难得的一次编写，多端运行，是的，它也很好的支持着移动Web端。

### 构建-build

Native使用weex-loader，Web则需要使用vue-loader，在Web端上vue-loader目前仅支持^11.3.3版本，以及weex-vue-render需要>= 0.11.50，并且vue-loader的配置做如下修改：

+ webpack 1.x
```javascript
module: {
  loaders: [
    {
      test: /\.vue(\?[^?]+)?$/,
      loaders: ['vue-loader']
    }
  ]
},
vue: {
  /**
   * important! should use postTransformNode to add $processStyle for
   * inline style normalization.
   */
  compilerModules: [
    {
      postTransformNode: el => {
        el.staticStyle = `$processStyle(${el.staticStyle})`
        el.styleBinding = `$processStyle(${el.styleBinding})`
      }
    }
  ]
}
```

+ webpack 2.x
```javascript
module: {
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
          compilerModules: [
            {
              postTransformNode: el => {
                el.staticStyle = `$processStyle(${el.staticStyle})`
                el.styleBinding = `$processStyle(${el.styleBinding})`
              }
            }
          ]
        }
      }
    ]
}
```
最佳的实践是推荐你使用目前为止我们内部评价最高的一份脚手架工程（支持三端一致，意味着处理了降级。）：[dingtalk-templates/webpack](https://github.com/dingtalk-templates/webpack)，
你可以直接下载它，自行修改package.json文件中的{{}} 配置，或者安装 [open-dingtalk/weex-dingtalk-cli](https://github.com/open-dingtalk/weex-dingtalk-cli) 这个命令行工具来玩转脚手架，这个命令行工具就像你使用vue-cli一样的简单：

```bash
$ npm install -g weex-dingtalk-cli
```
### 样式-style

+ weex支持的样式属于css子集
+ 必须写完整，如background:#000需要写成background-color:#000
+ 样式不允许提取文件，必须写在Vue的单组件中
+ 原则上不推荐使用预处理器，因为无法预期转译出来的样式符合weex的css子集
+ 布局只能使用Flexbox
+ 如果要显示文本必须使用text组件，并且你想改变字体大小必须写在text组件上
+ 只支持class，不允许继承
+ 单位只支持px
+ 不支持背景图片
+ 基于750px进行缩放，会有浮点级别的误差
+ 样式需要声明 scoped 属性
+ Android上处理圆角，必须在外层div中设置border-radius
+ 如果你想动态的替换class，只能使用数组表达式，<div :class=['name', a? 'b': 'c']></div>

如果你想使用预处理器（只是不推荐），可以如下配置：

```javascript
{
    test: /\.vue$/,
    loader: 'vue-loader',
    options: {
        loaders: {
          scss: 'vue-style-loader!css-loader!sass-loader'
        }
    }
}
```

```html
<style lang="sass">
    @import './common.scss'
    // ...
</style>
```
如果你想使用更精准的适配（无法忍受浮点级别的误差），可以获取scale，deviceWidth自行进行适配，推荐在loader阶段去处理（自行开发转换工具）。

### JavaScript与内存管理-JavaScript and memory manage

> 由于JS运行在JavaScriptCore/V8中，此与Web有较大差异。

如下：

+ jquery，axios 之类的原来Web开发领域的库都不可以使用
+ 不支持DOM操作
+ 虽然提供了Native DOM可以操作界面的渲染，原则上不推荐使用，方法与DOM操作类似
+ 既然不支持DOM操作，更改界面的方式应该使用数据驱动
+ 仅支持部分事件
+ weex SDK >= 0.10.0 的才支持事件冒泡
+ 没有window，document，location，history等对象
+ runtime是一个“全局环境”，不允许往全局环境中挂载对象，因为无法释放且所有weex页面共享
+ 只有scroller和list组件可以滚动
+ 不允许在Vue中操作style，遍历是很耗性能的
+ Vue中的v-show等原来操作Dom的指令或Api都不可以使用
+ vue-router 只允许使用 abstract 模式
+ vuex必须在初始化之前使用Vue.use注入
+ native端只能使用网络图片，解决的方式是在最后上线时统一替换成CDN
+ + 热更新以及增量更新的方式都可以参考React Native目前成熟的方案
+ iOS由于使用了同一套URL System，UIWebView的cookie是会共享到weex中的，同理weex中的cookie也是会共享的，只有WKWebView不会。原则上，你不应该使用cookie来处理用户体系的问题

> weex native 与 weex web 之间的差异较大，那么怎么办？   
我们提供了一套抹平一些常见差异的库，你也可以在weex环境中使用，[https://github.com/open-dingtalk/weex-dingtalk-journey](https://github.com/open-dingtalk/weex-dingtalk-journey)。   

在说内存（memory）之前，大家先来看一副图，weex的内存分布：

![](https://segmentfault.com/img/remote/1460000010023505?w=916&h=958)

正常情况下，Native memory 业务开发人员是无法处理的，而运行在js core 中的内存，我们知道如果不断开引用，js是无法回收释放内存的。   

+ 不允许往 runtime 里去挂载对象
+ 业务代码中的一些引用在beforeDestroy 中断开设置为null
+ 学会使用工具分析内存泄漏的问题，https://webkit.org/downloads/
+ 不要随意的使用函数递归，缩短对象方法的执行路径（传统JS领域的内存管理最佳实践也适用一部分）
+ 由于界面的渲染需要依赖createInstance(id, code, config, data)，sendTasks(id, tasks)，receiveTasks(id, tasks)发送指令的方式进行通信，你应该减少通信的次数，在更新界面时，合并不必要的通信指令的发送。
+ 如果你使用vue-router的方式，尽量减少组件之间的共享。

### 转场方式-navigator

由于weex的特殊性，它的转场方式有几种构成。

+ weex to weex，如果你需要支持钉钉js-api，那么你应该使用openLink。（如果是你自己实现，使用weex自带的navigator模块）
+ weex to h5 依然使用openLink，（如果是你自己实现，那么可以通过module的方式来打开一个WebViewController| UIWebView or WKWebView）
+ native to weex 直接alloc weex 容器的Controller传入Url即可
如果你使用vue-router，那么配置好你的路由path，使用push，go方法即可，唯一可惜的是使用vue-router的方式较为生硬。   

### 页面级别的数据传输-Page level data transfer

> 页面级别的数据传输基本很少会发生，钉钉的开发者推荐统一使用domainStorage方案。
+ weex to weex 通过URL传参数（携带的数据量有限），通过weex storage module
+ weex to h5，h5 to weex 通过URL传参数
+ native to weex 通过alloc weex 容器中的option或者data传入，前者可以在weex.config中获取，后者可以在vm上下文中获取
+ weex to native 定义一个跳转native的module，使用native的属性或者init时传入

### 调试工具-Debug Kit used

weex的调试工具需要额外安装weex-toolkit，weex-devtool，以及在你的Native工程中集成对应的WXDevtool（iOS）。    

如果你运行weex debug遇到如下的错误：

```bash
Error: EACCES: permission denied, open '/Users/xxx/.xtoolkit/node_modules/weex-devtool/frontend/weex/weex-bundle.js'
    at Error (native)
  ```
  
（非Windows用户）使用sudo即可。

+ 不集成 WXDevtool SDK
首先，你需要安装Weex Playground，可自行在各大市场中下载安装。   

不需要指明文件路径，在终端输入：

```bash
$ weex debug
```

先使用 Weex Playground 扫码（启动成功后会弹出一个界面），然后将你的业务代码贴到 这里，注意：   

+ 不允许出现import等导入模块的语法
+ 安装了Weex Playground的设备和你的电脑必须在同一局域网内

最后用安装了Weex Playground的设备扫码（业务代码贴过去那里的右侧会出现的二维码）。   

+ 集成 WXDevtool SDK

```
[WXDevTool setDebug:YES];
[WXDevTool launchDevToolDebugWithUrl:@"ws://192.168.1.108:8088/debugProxy/native"];
```

ws:// xxx.xx..x 这个地址是在用weex debug 在终端里给你输出出来的，如果setDebug为YES会开启debugger模式，反之亦然。   

注意setDebug设置为YES。

### 原生开发-Native

请直接阅读 [Weex入坑之旅](https://zhuanlan.zhihu.com/p/25182677) ，这是用iOS视角写的一篇文章，大概在半年之前。

### 写在最后

希望大家可以用一个开放的心态来看待weex，它的设计，实现有很多是值得学习的地方，   
比如多framework支持，共享runtime，module，component，handler等等，非常的自由领域，相当于它设计好了一个render engine。   
理论上你可以学习它的几个关键接口，知道Native DOM指令后，也能开发出替代Vue的上层框架，不信？你看看Rax即明白了。   
  
   
weex也有一些不足的地方，开发者数量少，社区活跃度不高，很多问题并不一定能被google搜录到。   
文档确实有一点不完善，native的实现也有一定的bug数量，你看react这么多年了，依然有bug，只要在逐步改进迭代修复，我觉得它就是非常棒的，万事没有十全十美，美中不足的一点瑕疵，说不定才是完美呢。
