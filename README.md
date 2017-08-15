## 写在最前
熟悉 React 技术栈的同学，想必对 Redux 数据流框架并不陌生。其倡导的单向数据流等思想独树一帜，虽然样板代码会有一定程度上的增多，但是对于开发效率和调试效率的提高是显著的。同时还带来了很多诸如 “时间旅行”，“ undo/redo ” 等黑魔法。

其实这还只是表象。如果你深入去了解 Redux 的设计理念，探索中间件奥秘，玩转高阶 reducer 等等，迎接你的就会是另一扇门。透过它，函数式编程思想之光倾斜如注。

## 思想背景
但是随着这个 web app 复杂度的提升，数据计算量压力徒增，你所设计的 Reducer 变得臃肿不堪。好吧，我们可以拆分 Reducer 使得代码看上去更加舒服。可是计算量呢？也许有一些“梦魇”，瓶颈般永远无法消除。

冥冥之中，“各种处理计算既然注定在同一时空，那么能否永远平行？”


**曾几何时，你是否听说过 JS 单线程异步？听说过浏览器卡顿或卡死？听说过 60 fps？**

其实一个很严峻的事实是：根据 60 fps 计算，每一帧留给我们 JS 执行的时间为 16ms（甚至更少）。那么一旦当 Reducer 计算时间过长，必然会影响浏览器渲染。

## 多线程思路
关于浏览器主线程、render queue、event loop、call stack 等内容，本文不再复述，因为里面的知识完全都够写一本书了。假定读者对其有一二认知，那么你也不难理解我们即将登场的救星—— Web Worker!

**我们先来简单认识一下 web worker：**

2008 年 W3C 制定出第一个 HTML5 草案开始，HTML5 承载了越来越多崭新的特性和功能。其中，最重要的一个便是对多线程的支持。在 HTML5 中提出了工作线程（Web Worker）的概念，并且规范出 Web Worker 的三大主要特征：

- 能够长时间运行（响应）；
- 理想的启动性能；
- 以及理想的内存消耗。

Work 线程可以执行任务而不干扰用户界面。

**于是，脑洞大开，能否将我们的 Redux Reducer 计算状态部分放进 Worker 线程中处理呢？**

**答案是肯定的。**
**那么要如何实施呢？**

我们先来看一下经典的 Redux workflow，如下图：


![redux 流程图](http://upload-images.jianshu.io/upload_images/4363003-9287ea86fb7f41d0.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

    
如果要接入 Web Work，那么我们改动流程图如下：


![redux + worker 流程图](http://upload-images.jianshu.io/upload_images/4363003-d0cfa1344915694e.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## 具体实现和一个demo

当然，有了思路，还需要在实战中演练。

我使用 [“N-皇后问题”](http://blog.csdn.net/hackbuteer1/article/details/6657109) 模拟大型计算，并且实现的 demo 中可以任意设置 n 值，增加计算耗时。
如果你不理解此算法也没有关系，只需要知道N-皇后问题这个算法的计算耗时很长，且和 n 值相关：n 越大，计算成本越大。

除了一个极其耗时的计算，页面中还运行这么几个模块，来实现复杂的渲染逻辑操作：

- 一个实时每16毫秒，显示计数（每秒增加1）的 blinker 模块；
- 一个定时每500毫秒，更新背景颜色的 counter 模块；
- 一个永久往复运动的 slider 模块；
- 一个每16毫秒翻转5度的 spinner 模块


![页面过程](http://upload-images.jianshu.io/upload_images/4363003-411a0716711aecd2.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这些模块都定时频繁地更新 dom 样式，进行大量复杂的渲染计算。正常情况下，由于 JS 主线程进行N-皇后计算，这些渲染过程都将被卡顿。

同时，我设置“N-皇后问题”的 n 值，来观察在计算时这些模块的表现（是否卡顿）。在不开启 Work 线程的情况下，n 设置为13时，有 gif 图，左半部分：


![off.gif](http://upload-images.jianshu.io/upload_images/4363003-cfde4fbadc85b4e5.gif?imageMogr2/auto-orient/strip)


我们非常清晰地看到：由于浏览器 call stack 进行 n=13 的皇后问题计算，而无法“按时”渲染，所以造成了这几个模块的卡顿，这些模块都无法更新状态。**在这个卡顿过程中，用户的任何事件（如点击，敲键盘等）都无法被浏览器响应。**这就是用户体会到的“慢”！

如果我把 n 值设置的大与13呢，比如24？
千万不要这么做！因为你的浏览器会被卡死！我使用 Mac Pro 8G 内存情况下，设置到14，浏览器就无法响应了。

在开启 Work 线程时，请参考上 gif 图右半部分，几个模块的渲染丝毫不受影响。完美达到了我们的目的。

因为 Reducer 的超级耗时计算被放入 Worker 线程当中，所以丝毫没有影响浏览器的渲染和响应。完全解决了用户觉得“电脑慢”的问题。



看到了如此完美的对比，也许你想问 Web Worker 的兼容性如何呢？


![兼容性](http://upload-images.jianshu.io/upload_images/4363003-c371ca5c1ac345a6.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 总结
其实，这篇文章的意义并不在于这个 demo 和应用。而是在启发一种新的想法的同时，review 了很多 JS 当中关键概念和基本知识。比如：单线程异步、宿主环境、60 fps、一个算法等等。

更值得一提的是，如果你去深入 demo 代码，你更会发现 Redux 设计精妙的思想，比如我们将 Web Worker 的应用抽象出一个公共库：**Redux-Worker，并包装为 Redux 的中间件（middleware），所有 React Redux 都可以无侵入，采用中间件的思想使用：**

    import { applyWorker } from 'redux-worker';
    const enhancerWithWorker = compose(
        applyMiddleware(thunk, logger),
        applyWorker(worker)
    );
    
    const store = createStore(rootReducer, {}, enhancerWithWorker);
    
当然，Redux-Worker 这个中间件的实现原理更是巧妙，这里不再展开。感兴趣的同学可以参考我的[此项目 Github 仓库。](https://github.com/HOUCe/redux-worker-demo)我 fork 了此[库源码](https://github.com/chikeichan/redux-worker)，并在核心逻辑加入了中文注释，感兴趣的同学可以关注。







 **我的其他关于 React 文章：**
- [通过实例，学习编写 React 组件的“最佳实践”](https://zhuanlan.zhihu.com/p/27825741)
 - [React 组件设计和分解思考](https://zhuanlan.zhihu.com/p/27727292)
 - [从 React 绑定 this，看 JS 语言发展和框架设计]()
- [React 服务端渲染如此轻松 从零开始构建前后端应用](https://zhuanlan.zhihu.com/p/28004982)
- [做出Uber移动网页版还不够 极致性能打造才见真章](http://www.jianshu.com/p/49029b49f2b4)
- [解析Twitter前端架构 学习复杂场景数据设计](http://www.jianshu.com/p/7a56ac1de2a8)
- [React Conf 2017 干货总结1: React + ES next = ♥](http://www.jianshu.com/p/83c86dd0802d)
- [React+Redux打造“NEWS EARLY”单页应用 一个项目理解最前沿技术栈真谛](http://www.jianshu.com/p/cde3cf7e2760)
- [一个react+redux工程实例](http://www.jianshu.com/p/8e28be0e7ab1)





Happy Coding!

PS: 
作者[Github仓库](https://github.com/HOUCe) 和 [知乎问答链接](https://www.zhihu.com/people/lucas-hc/answers)
欢迎各种形式交流。