### application.js(koa 的顶部构造函数) => 设计核心
首先大家要看到通过 CMD 规范释出的类 Application 是继承于 Emitter 类的，这个 Emitter 类即是 Node 的 events 模块，直接使 Application 类变为异步事件驱动架构，比如 http 模块的 server 类。后续在代码中会有所体现，写过任何前端组件的同学都应该看看文档就懂了。

接下来我们直接从 listen 函数切入
```
listen(...args) {
  // 利用 debug 包监听 listen 过程中的信息
  debug('listen');

  // 创建 http 实例服务，此时可看出 this.callback() 应返回一个带 req 与 res 作为参数的函数
  // 此时 server 即返回新建的 http.Server 实例
  const server = http.createServer(this.callback())

  // args => [port][, host][, backlog][, callback]
  return server.listen(...args);
}
```
线索跳至 this.callback()，在 demo 中即 app.callback()
```
callback() {
  // 使用 koa-compose 将所有中间件集成返回为一个中间件函数，此时 fn 是一个闭包，我们在下面会解释 koa-compose 源码，因为这对中间件的顺序执行起到关键性作用，是 koa 整个中间件设计的核心。
  const fn = compose(this.middleware);

  // 监听整个 app 的 error 事件，继承于 eventsEmitter 后的用法
  if (!this.listenerCount('error')) this.on('error', this.onerror);

  // 返回 createServer 参数中所需的回调函数
  const handleRequest = (req, res) => {
    const ctx = this.createContext(req, res);
    return this.handleRequest(ctx, fn);
  };

  return handleRequest;
}
```
在讲解 compose 之前，我们知道在 koa 中使用中间件都是直接使用实例 api use 的，如果想知道中间件设计模式的原理，首先必须要知道 use 做了什么
```
use(fn) {
  // 两项校验很简单，一是校验引入的中间件一定要为函数，二是校验中间件函数是否使用了 generator，此特性在 V3 版本会被全面替换为 async await 希望读者可以自己替换
  if (typeof fn !== 'function') throw new TypeError('middleware must be a function!');
  if (isGeneratorFunction(fn)) {
    deprecate('Support for generators will be removed in v3. ' +
              'See the documentation for examples of how to convert old middleware ' +
              'https://github.com/koajs/koa/blob/master/docs/migration.md');
    fn = convert(fn);
  }
  debug('use %s', fn._name || fn.name || '-');

  // 核心在此，我们每次 use 的全局中间件都会被 push 入实例 app 的 middleware 栈中顺序保存，最后返回实例
  this.middleware.push(fn);
  return this;
}
```
现在我们知道了原来一直 use 就是将自己引入的中间件入栈保存，那么它会在哪执行呢？现在我们回到 this.callback 中的 `const fn = compose(this.middleware)` 去先分析 koa-compose 的源码，就会非常清楚了，以下为解析

```
function compose (middleware) {
  // 判断参数必须为数组
  if (!Array.isArray(middleware)) throw new TypeError('Middleware stack must be an array!')
  // 判断数组内每一个 middleware 必须都为 Function
  for (const fn of middleware) {
    if (typeof fn !== 'function') throw new TypeError('Middleware must be composed of functions!')
  }

  // 返回闭包，由此可知在 koa this.callback 中的 fn 后续一定会使用这个闭包传入过滤后的 context(ctx)
  return function (context, next) {
    // last called middleware #
    // 初始化中间件函数数组执行的下标值
    let index = -1

    // 返回递归执行的 Promise.resolve 去执行整个中间件数组，初始从第一个开始
    return dispatch(0)
    function dispatch (i) {
      // 校验上次执行的下标 index 不能大于本次执行的传入下标 i，如果大于可能是 next(下个中间件) 执行了多次导致
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))

      // 赋值当前执行的中间件函数下标值
      index = i

      // 取出当前要执行的中间件函数，如果当前执行的下标已经等于中间件数组长度，那么此时 fn = next 一定会是空，那么此时直接返回 Promise.resolve() 宣布中间件数组执行完毕
      let fn = middleware[i]
      if (i === middleware.length) fn = next
      if (!fn) return Promise.resolve()

      // 若数组下标并未到达最后一位，且存在当前中间件函数则执行当前函数并传入 dispatch(i + 1)，就可看出是数组中的下一个中间件了，此时作为 next 传入了中间件函数中；
      // 也就是说我们写中间件时，已经默认注入了 ctx 与 下次执行的封装函数 next，也是因为如此我们在 koa 的中间件中才可以非常方便的判断什么时候进入下一个中间件去执行的洋葱结构，并且一定要执行 next() 否则数组将在此中断，因为这里是 Function.prototype.bind(如果连 bind 都不知道的童鞋，唉，我不好意思说你了，别看了吧)
      // 需注意的是 bind 时指向 null 也是为了以防在执行过程中你有什么骚操作改变了指向，那就不好了
      // 在不断的 Promise.resolve 中去实现递归 dispatch 函数，最终实现顺序控制执行所有中间件函数
      try {
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
      } catch (err) {
        return Promise.reject(err)
      }
    }
  }
}
```
OK，现在我们已经知道 koa 设计模式的整个中间件体系是怎么运作的了，这对整个设计模式的理解非常关键，接下来我们继续回到 this.callback 中向下走，接下来我们看看返回的 handleRequest 到底干了什么，根据上面对 compose 的分析，我们应该已经可以猜到里面必然会使用 `const fn = compose(this.middleware)` 这个 fn 闭包了，好的，我们也分为两段讲

首先是 `cosnt ctx = this.createContext(req, res)`

其次是 `return this.handleRequest(ctx, fn)`，看吧这里立刻就传入了！

需要注意其中 createContext 函数参数 req 与 res 即是在 listen 函数中 createServer(this.callback()) 时注入的，不知道的童鞋请自行查看 node 文档中的 http.createServer
```
createContext(req, res) {
  // 将过滤的 context.js 赋值
  const context = Object.create(this.context);
  const request = context.request = Object.create(this.request);
  const response = context.response = Object.create(this.response);

  // 将实例挂载到 context.app 中
  context.app = request.app = response.app = this;

  // 将 request 事件的http.IncomingMessage 类挂载到 context.req 中
  context.req = request.req = response.req = req;

  // 将 request 事件的http.ServerResponse 类挂载到 context.res 中
  context.res = request.res = response.res = res;

  // 将 context 上下文挂载到 request.ctx 中，以下同理，都是互相挂载的过程，使用户可以在 koa 中通过 ctx 就可以拿到所有需要的东西，方便在中间件中的各类操作
  request.ctx = response.ctx = context;
  request.response = response;
  response.request = request;
  context.originalUrl = request.originalUrl = req.url;
  context.state = {};
  return context;
}
```
```
handleRequest(ctx, fnMiddleware) {
  // 取出响应类
  const res = ctx.res;
  res.statusCode = 404;

  // 使用 context 中的 onerror 方法（其中使用了 eventsEmitter 的 emit 触发实例在 app.callback 时监听的 error 事件）
  const onerror = err => ctx.onerror(err);

  // 处理中间件全部执行完后的响应参数过滤，如进行请求 method 的过滤，处理 ctx.body 为 buffer，string，json，Stream 时的响应结果
  const handleResponse = () => respond(ctx);

  // 声明响应结束后的回调事件(一般一定要确保关闭连接并关闭文件描述符)，这里使用了 ctx.onerror 作为回调，大家可在内部找到 res.end() 去关闭
  onFinished(res, onerror);
  // 使用 compose 执行的闭包并在所有中间件执行完毕后改变响应的各类参数，并 catch 去监听全局错误
  return fnMiddleware(ctx).then(handleResponse).catch(onerror);
}
```
在此整个顶部设计就此完成，从实例化 koa 对象 => 新建服务 => 存取并处理中间件 => 创建并关联上下文 => 顺序控制执行中间件 => 处理最后响应参数 => 全局监听整个中间件的执行过程及 debbug 整个 listen 的过程

换成函数就是 Application.listen => Application.callback => koa-compose 源码分析 => Application.createContext  => Application.handleRequest

**在这个过程中不要忘记，所有的接口函数本质上也是使用 use 引入的，所以其实它们也算是中间件，只是我们在编写时会将它们放在最后才 push 入 app.middleware 中，所以它们会在我们之前的中间件执行后被执行，故在这之前我们可以先根据业务写好过滤中间件及相关通用中间件，所以整个过程 koa 都会监听到，并在最后才处理响应返回**
