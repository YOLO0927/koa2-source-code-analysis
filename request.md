### request.js(koa 的 ctx.req) => 封装于 http.IncomingMessage 类（即新建服务时回调内的 request）

首先我们要先清楚 request.js 中的 this.req 即为 application.js 内的 request.req，在 callback 函数中最终赋值给了 ctx.req => request.js 中的 this.req =>
```js
const request = context.request = Object.create(this.request)
context.req = request.req = response.req = req
```
从顶层看过来的同学应该都这知道

所以很多东西就非常好解释了，大家对着 node 的文档就可以找到这些怎么来的了，就是一些比较简单的封装，如 ctx.header => 在 context.js 中由 ctx.req.header 挂载而来 => node 文档中 htt.IncomingMessage 类下的 message.headers（即为 createServer 内回调的 request.headers），而 set 访问器也是对应着修改 request.headers，其他挂载在 context 下的也同理。

其他的好像就没什么好解析的了，都是很简单的过滤函数以及 set 时更改对应 request 下的 api 而已。
