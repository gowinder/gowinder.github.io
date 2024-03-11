# 解决React开发模式服务端Cors问题


# "解决React开发模式服务端Cors问题

## 启因

最近在做`openai`的后端代理服务,一个后台池账号池,缓冲前端过来的`openai`的`completion`请求.

后端基本开发完成了,所以需要一个demo前端做测试及演示用,就用`react`做了一个简单的前端聊天程序,代码基本由`GPT-4`来生成,生成了多次,拼接了一下,由于我也没有写过`react`,所以还花了点时间选择拼接,因为`CHATGPT`也经常出些错误,懂的人一看就知道,不懂的人就要上当.

程序运行起来后,因为后端给前端的接口是一个`websocket`,启动后就报错: `Access to XMLHttpRequest at 'http://localhost:3014/socket.io/?EIO=4&transport=polling&t=OSYSDmI' from origin 'http://localhost:3000' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.`



## 解决

查了一些资料,`CORS`问题其实需要在`SERVER`端解决,首先我的`SERVER`端用的是`Nestjs`,`websocket`用的是`WebSocketGateway`,定义如下:



```typescript
@WebSocketGateway({
  cors: {
    origin: '*',
  },
  namespace: 'main',
})
export class WSGateway
  implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect {
  	...
  }
```

`websockeet`已经加了`CORS`了,应该不是这个问题,问题出在`localhost:3000`,这个是前端`react`用的`web`服务,报错也是这个`3000`,经过查询,开发模式启动`react-scripts`用的是`webpack`的开发服,根据[官方文档](https://create-react-app.dev/docs/proxying-api-requests-in-development/):

新建一个`scr/setupProxy.js`

```javascript
const { createProxyMiddleware } = require("http-proxy-middleware");

module.exports = function (app) {
  app.use(
    "/",
    createProxyMiddleware({
      target: "http://localhost:3014", // Replace with the API server URL
      //   changeOrigin: true,
      ws: true,
      logger: console,
      //   pathRewrite: {
      //     "^/api": "", // Remove the '/api' prefix from the request URL
      //   },
    })
  );
};

```

`target`填入`react`要连接的目标服务器地址, `ws`设置成`true`


