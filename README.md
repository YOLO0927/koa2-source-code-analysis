### koa2 源码解析
koa2 的源码整体上来说十分简单，总共也就几百行，很快就可以阅读完毕，这里是用于记录我阅读完后的笔记，阅读源码我们只需要看 lib 下的可执行文件即可，首先让我们来看看目录中的脚本分部情况。

lib<br>
├── application.js<br>
├── context.js<br>
├── request.js<br>
└── response.js

首先先放上最简单的新建服务实现

```js
  cosnt Koa = require('koa');
  const app = new Koa();
  app.listen(port)
```

我们就由这个最简单的 demo 来开始自顶向下分析源码的构成，我们将整体分成 4 大块，用文档的例子就是
1.  application => app(实例化的应用)
2.  context => ctx(实例上下文)
3.  request => req(由原生 request 事件的 http.IncomingMessage 类过滤而来，在 koa 中即为挂载在 ctx 下的 ctx.req 请求流)
4.  response => res(由原生 request 事件的 http.ServerResponse 类过滤而来，在 koa 中即为挂载在 ctx 下的 ctx.res 响应流)

由此我们最先就必须先从 koa 实例化与挂载端口的过程去分析，其中到底做了些什么封装，按照自顶向下的顺序我们的解析顺序是
1.  application.md（本质上看玩这个就ok了，后面随便你看不看= =...）
2.  context.md 或 request.md  或 response.md 此项没有严格顺序了，哪个先看都可以，因为都不是 koa 设计的核心重点～都算是辅助主设计的了。

**若在解析文档中有明显错误，还请童鞋们提醒我，我会立刻更改。**
