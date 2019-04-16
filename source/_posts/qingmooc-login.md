---
layout: simple-article
title: 微信扫码登录(需先关注公众号)
date: 2018-05-07 21:15:37
categories:
    - 轻慕课
tags:
    - WebSocket
---

最近实验室项目遇到一个新需求。之前项目用的微信扫码登录是利用的微信实现的网站应用扫码功能，需要变成先关注公众号才能登录进去的扫码登录，这样用微信原本的扫码登录就不行了。
<!-- more -->
- 目前扫码后效果是这样的：

![data:image](http://qingmooc-v1.oss-cn-qingdao.aliyuncs.com/other/WechatIMG15.jpeg)

然后一确认登录 PC端就登录成功了。

但是这一流程的问题在于，用户不需要关注公众号，然后老师就不乐意了，咱们要做**先关注公众号才能登录进去的扫码登录**。

- 也就是说目标效果是这样的：

扫码之后显示公众号首页
![data:image](http://qingmooc-v1.oss-cn-qingdao.aliyuncs.com/other/WechatIMG16.jpeg)

点击关注之后，PC 端登录成功。

> 微信相关api：[常规扫码登录](https://open.weixin.qq.com/cgi-bin/showdocument?action=dir_list&t=resource/res_list&verify=1&id=open1419316505&token=&lang=zh_CN)、[生成公众号二维码](https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1443433542)、[关注公众号后回调](https://mp.weixin.qq.com/wiki?t=resource/res_main&id=mp1421140454)

### 一、方案

- 客户端向服务器请求二维码
- 服务器调用微信接口，生成公众号的二维码
- 用户扫码并关注公众号后，微信会回调后台接口，后台认证用户信息
- 后台通知前端用户登录成功，并将登录凭证给前端

以上流程基本上是符合我们的需求的，但是最后一步用到了 服务器推送技术。

说到服务器推送技术，那就不得不说 WebSocket 协议了。WebSocket 协议规避了Ajax 轮询和 Ajax 长轮询技术的缺点。我最后选择的是 WebSocket 协议来实现服务器推送。

参考这篇文章，[Websocket 研究 / Nodejs 模块选型对比](https://cloud.tencent.com/developer/article/1005550)，我决定使用 Node.js 的 ws 模块进行开发。

### 二、具体实现

#### 2.0 服务器起一个 WS 服务器
我平时常用的是 Koa 框架，所以当然也希望可以配合 Koa 框架使用 ws，我找了一下后发现了 koa-websocket 包

这个包做的大致是 new 了一个 ws.Server 对象，并将它挂在了 koa 的 app 上面，从而可以通过 app.ws 来访问 ws.Server 对象。当有 ws 请求时，可以用 ctx.websocket 得到客户端连接。

```js
import Koa from 'koa';
import websockify from 'koa-websocket';

const verifyClient = (info) => {
  const { origin, req, secure } = info;
  return true;
};

// ws 的选项
const wssOptions = {
  verifyClient,
  clientTracking: true,
};


const app = websockify(new Koa(), wssOptions);

// 路由中间件
app.ws.use(signUpRoute);

// process.env.NODE_APP_INSTANCE 是用 pm2 部署应用时会有的系统变量，从 0开始
process.env.NODE_APP_INSTANCE = process.env.NODE_APP_INSTANCE || 0;

// 如果 pm2 启动了两个 Worker，那就分别监听 3001 和 3002 端口
const port = 3001 + parseInt(process.env.NODE_APP_INSTANCE, 10);

app.listen(port, () => {
  console.log(`WebSocketServer listening on http://localhost:${port}`);
});
```

app/websocket/router.js

```js
// 路由中间件
import route from 'koa-route';

import { signUp } from 'app/modules/signup';

// signUp 函数中是具体的处理逻辑，也就是调用微信那些 api，最终生成 二维码
const signUpRoute = route.all('/ws/signup', signUp);

export {
  signUpRoute,
};
```

#### 2.1 客户端向服务器请求二维码

客户端在打开网站的登录页面后，页面会创建一个连接到后台服务器的 WebSocket 对象，并发送一个请求二维码链接的请求，后台在收到前端请求后，会调用微信的生成带参数的二维码接口，生成公众号的二维码链接，并传给前端。

前端：
```js
  // 前端使用的是 React 框架，将生成 WebSocket 对象的步骤放在 componentDidMount 中
  componentDidMount() {
    this.ws = new WebSocket('wss://www.xxx.com/ws/signup')
    
    // 客户端发送一个二维码请求
    this.ws.onopen = () => {
      this.ws.send(JSON.stringify({
        type: 'qrcode',
      }))
    }
    
    // 客户端监听服务器端的回复
    this.ws.onmessage = (event) => {
      const { data } = event
      const { type, message } = JSON.parse(data)
      // 
      if (type === 'qrcode') {
        this.setState({
          url: message
        })
      }
    }
  }
```
#### 2.2 服务器调用微信接口，生成公众号的二维码

```js
const signUp = async (ctx) => {
  const socket = ctx.websocket;
  socket.on('message', async (message) => {
    const messageJSON = JSON.parse(message);
    const { type } = messageJSON;
    const payload = {
      type: 'nothing',
    };
    switch (type) {
      // 收到前端的qrcode 请求
      case 'qrcode': {
        const ticket = await getQRCodeUrl();
        if (ticket) {
          const { qrcodeUrl } = login;
          // 给 socket 一个唯一标识
          socket.uuid = ticket;
          payload.type = 'qrcode';
          payload.message = `${qrcodeUrl}${encodeURIComponent(ticket)}`;
          socket.send(JSON.stringify(payload));
        }
      }
        break;
      default: payload.message = '';
    }
  });
};
```
可以看到，中间有一步是客户端连接后端后，后端会给这个ws socket 一个唯一标识，从而区分每个客户端。我这里是将一个生成二维码时的一个唯一字符串赋给了那个socket，这样在用户扫码之后，微信会回调后台的接口，并将那个唯一字符串传回来，然后通过那个唯一字符串就可以找到对应的 socket，从而发送登录成功的消息。

#### 2.3 用户扫码后，微信会回调后台接口，后台认证用户信息

当用户扫码并关注公众号后，微信会回调后台接口，其中会带有用户的openid 和用于标识一个 ws 的唯一字符串，后台会根据 openid 去查询用户的信息，并生成 jsonwebtoken。

#### 2.4 后台通知前端用户登录成功，并将登录凭证给前端

```js
const token = await postMessage(openid);
postClient(uuid, token);

// postClient 函数实现
const wsServer = app.ws.server;
const postClient = (uuid, message) => {
  const clients = wsServer.clients;
  let client;
  for (const cli of clients) {
    if (encodeURIComponent(cli.uuid) === encodeURIComponent(uuid) && Number(cli.readyState) === 1) {
      client = cli;
      break;
    }
  }
  if (client) {
    client.send(JSON.stringify({
      type: 'token',
      message,
    }));
    client.close(1000, JSON.stringify({
      type: 'end',
      message: 'connection will be terminated',
    }));
    return true;
  }
  return false;
};
```
其中 app.ws.server.clients 是一个 Set，里面存放了当前所有的 ws 连接。client.readyState 标识了当前连接的状态，为 1 时表示连接状态。具体看 ws 的[文档](https://github.com/websockets/ws/blob/HEAD/doc/ws.md)

uuid  就是微信传回来的唯一标识

可以通过 uuid 取到对应的ws 连接，并判断这个连接是否还是连接状态，如果还是连接状态，就将 jsonwebtoken 传过去，客户端也就登录成功了。

### 三. pm2 部署 + nginx 反向代理

代码编写测试好，就剩部署了。我这里是用 pm2 启动了两个 WebSocket Server 实例，每个实例监听不同端口。由于使用的是 WebSocket 协议，所以 nginx 要加一些配置。

```
upstream wss_nodes {
 ip_hash;
 server 127.0.0.1:3001;
 server 127.0.0.1:3002;
}

map $http_upgrade $connection_upgrade {
  default upgrade;
  ''      close;
}


server {
    ...
  location /ws {
    proxy_pass http://wss_nodes;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    proxy_set_header Upgrade $http_upgrade;
    # Connection 头部的值 取决于客户端请求头是否存在 Upgrade 字段 
    proxy_set_header Connection $connection_upgrade;
    proxy_read_timeout 300s;
  }
}
```
使用 ip_hash 是为了让同一客户端的连接每次都能代理到同一 wss 实例。

关于代理请求的头部设置，参考了 nginx 的[文档](http://nginx.org/en/docs/http/websocket.html)、nginx [map模块配置和使用](http://www.phperz.com/article/16/0507/212376.html)

> WebSocket 和 HTTP 协议不同，但是 WebSocket 中的握手和 HTTP 中的握手兼容，它使用 HTTP 中的 Upgrade 协议头将连接从 HTTP 升级到 WebSocket，当客户端发过来一个 Connection: Upgrade 请求头时，Nginx 是不知道的，所以，当 Nginx 代理服务器拦截到一个客户端发来的 Upgrade 请求时，需要显式来设置 Connection 、 Upgrade 头信息，并使用 101（交换协议）返回响应，在客户端和代理服务器、后端服务器之间建立隧道来支持 WebSocket。

> WebSocket 仍然受到 Nginx 缺省为60秒的 proxy_read_timeout 的影响。这意味着，如果你有一个程序使用了 WebSocket，但又可能超过60秒不发送任何数据的话，那你要么需要增加超时时间，要么实现一个 ping 的消息以保持联系。使用 ping 的解决方法有额外的好处，可以发现连接是否被意外关闭


### 四. pm2 主从进程通信

经过上述步骤之后，在线上测试时，发现用户扫码之后，有时候后端会推送给前端 jsonwebtoken，有时候不会。稍微想了一下，发现还是大意了。

假设现在两个 WS 服务器分别监听 3001、3002 端口，当一个客户端发起一个 WebSocket 连接到 3001 端口的 WS Server，这个 WS Server 就持有了这个客户端的连接。然后后台返回给前端二维码，之后用户扫码，关注公众号，这时微信会回调后台接口，然后后台应该根据 uuid 找到对应的客户端连接，问题就在于后台应该去 3001 端口的 WS Server 找，还是应该去 3002 端口的 WS Server 找呢？

又翻了翻 pm2 的文档，pm2 是支持主从进程之间通信的，所以说可以通过 pm2 来发送消息，通知别的 WS Server “ 这个客户端在我这里没找到，你找找 ”

```js
import pm2 from 'pm2';
import { postClient } from 'app/websocket';

const neighborIds = [];

const myId = process.env.NODE_APP_INSTANCE;

// 通知其他进程
function broadcastMessageToNeighbor(type, message) {
  pm2.connect(() => {
    pm2.list((err, processes) => {
      if (err) {
        console.log(err);
      }
      processes.forEach((process) => {
        const ortherId = process.pm_id;
        if (Number(myId) !== Number(ortherId)) {
          neighborIds.push(ortherId);
        }
      });
      neighborIds.forEach((neighborId) => {
        // topic 字段不能少，暂且设为与 type 一样
        pm2.sendDataToProcessId(neighborId, {
          type,
          data: message,
          topic: type,
        }, (error) => {
          console.log(error);
        });
      });
    });
  });
}

// 当前进程监听 pm2 传递来的消息
process.on('message', (packet) => {
  const { type, data } = packet;
  if (type === 'token') {
    const { uuid, message } = data;
    postClient(uuid, message);
  }
});

export {
  broadcastMessageToNeighbor,
};
```
经过上面的处理，基本已经实现了目标的效果。

### 四. 新的解决办法(2018/6/30)

上面的办法虽然看起来解决了问题，但是其实是一种 hack 的方式，知道我翻阅 pm2 文档时，看到了应用实现负载均衡应当[首先将应用变为无状态应用](https://pm2.io/doc/en/runtime/best-practices/stateless-application/)，也就是说不应当将数据(比如 session 或者 websocket 连接)存储在进程内部，而应当使用 redis 数据库之类的方式使多个进程可以共享数据，这么一说我茅塞顿开啊，这说的不就是我的情况。

经过了一番比较之后，我挑选了一个新的服务器推送库[socket.io](https://socket.io/)，这个库非常强大，支持命名空间的概念，支持 room 和广播，并且还支持使用 Redis 作为 adapter，从而实现我们的需求 :)。

```js
const Server = require('socket.io');
const redisAdapter = require('socket.io-redis');
// 将我们的 server 作为第一个参数传入
const io = new Server(server, {
  // 默认 path 是 /socket.io
  path: 'ws',
  adapter: redisAdapter({ host: 'localhost', port: 6379 })
});
```
这样就启动了一个 socket.io 服务器，并且使用 redis 作为 adapter 实现了 stateless 应用。

需要注意的是 path 指的是 socket.io 的通信路由，所以还得在 nginx 中加一个对 /ws (默认是 /socket.io) 路由的反向代理，配置与第三节一致

由于使用了 redis 作为 adapter，那么 websocket 连接对于多个进程就相当于是共享的，因此不需要考虑之前碰到的问题了。

```js
// 数据库存储客户端ticket 和 连接的 socket.id 的对应关系
const log = await WechatTicket.findOne({
  ticket: decodeURIComponent(ticket),
});
if (_.isEmpty(log)) { return; }
// 向对应的客户端发送 token
const socketId = log.socketId;
await io.to(socketId).emit('token', token);
// 断开连接
await io.of('/login').adapter.remoteDisconnect(socketId, true, (err) => {
  // 处理错误
});
```
这样就真正地实现了这个需求。

### 五. 总结

通过上面的过程基本上完成了可用性，并且使用 https 以及 wss 来进行通信。但是还是有许多可以[优化的地方](https://security.tencent.com/index.php/blog/msg/119)，比如对Origin头部进行验证，防止跨站点WebSocket劫持攻击；设置单IP可建立连接的最大连接数等等




