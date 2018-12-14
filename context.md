### context.js(koa 的上下文) => 辅助设计

在使用 koa 时，我们会在中间件中直接使用被 koa-compose 处理后注入的 context，而 context 本身其实大部分是代理了 koa 中 request.js 与 response.js 封装的功能，下面我们来简单分析一下 context.js 里面都有什么

`toJson` 与 `inspect` 函数司空见惯，不多说了，大家自己看，这个没什么好说的，一般你也用不到。

`ctx.assert` 引入的断言工具，让你可以像 ctx.throw 一样，断言指定变量而返回你定义的 statusCode 与 errMsg，这个自己看看 `htt-assert` 的文档就会用了。

`ctx.onerror` 触发此错误事件时会同时触发 koa 实例中创建服务时默认 callback 中定义的 error 事件做出相应处理。

 针对 cookie 定义了访问器用于获取与设定，在 koa 的文档中我们看到需要设定 cookie 所对应的 key 数组，这是 cookies 包定义的
```js
get cookies() {
  if (!this[COOKIES]) {
    // 在实例 req 与 res 上定义 cookie 的 key 值，每次获取都会默认创建 cookie 的一个新 key 并为空（在没有 set 的情况下），直接返回 cookies 包实例。
    this[COOKIES] = new Cookies(this.req, this.res, {
      keys: this.app.keys,
      secure: this.request.secure
    });
  }
  return this[COOKIES];
},

set cookies(_cookies) {
  this[COOKIES] = _cookies;
}
```

所以我们可通过 get 访问器看出当获取 `ctx.cookies` 时机会直接返回新建 cookies 包的实例，此时我们按照 cookies 包的文档 API 操作即可，直接去 `cookies.get || cookies.set` => `ctx.cookies.get || ctx.cookies.set`。

最后则是通过 delegates 包将 `ctx.request` 与 `ctx.response` 内的函数，access(包括 getter 与 setter 访问器)与单独的 getter 直接挂载到 ctx 下，此时我们直接操作 ctx 下对应 api 将指向 `ctx.request` 或 `ctx.response`

到这里就都分析完毕了，context.js 只是一个方便访问各个 api 而挂载的上下文，所以内部本质上十分简洁，没有什么可圈可点的。
