# Understanding how the browser works by creating one

本文将介绍浏览器的工作原理，并据此实现一个简易的迷你浏览器。它不复杂，就像一个玩具，但足够能清晰地演示基础渲染原理。毕竟 learning by doing 是最有效的，let's go.

## 浏览器基础渲染流程
浏览器工作有以下五个步骤（时间？其他细节？）

* 浏览器首先使用 HTTP 协议或者 HTTPS 协议，向服务端请求页面；
* 把请求回来的 HTML 代码经过解析，构建成 DOM 树；
* 计算 DOM 树上的 CSS 属性；
* 最后根据 CSS 属性对元素逐个进行渲染，得到内存中的位图；
* 一个可选的步骤是对位图进行合成，这会极大地增加后续绘制的速度；
* 合成之后，再绘制到界面上。

这是一条流水线。

从 HTTP 请求回来，就产生了流式的数据，后续的 DOM 树构建、CSS 计算、渲染、合成、绘制，都是尽可能地流式处理前一步的产出。
即不需要等到上一步骤完全结束，就开始处理上一步的输出，这样我们在浏览网页时，才会看到逐步出现的页面。

[图]
URL =HTTP=> =parse=> DOM =CSScomputing=> DOM with CSS =layout=> DOM with position =render=> bitmap

## 第一步 网络请求部分

首先，我们起一个简易的服务器。它将返回一句简单的 ' Hello World\n'

./server/server.js
```
const http = require("http");

http.createServer((request, response) => {
  let body = [];

  request.on('error', error => {
    console.log(error);
  })
    .on('data', chunk => {
      body.push(chunk.toString());
      console.log(body)
    })
    .on('end', () => {
      body = body.join('');
      response.writeHead(200, { 'Content-Type': 'text/html' });
      response.end(' Hello World\n');
    });
}).listen(8080);

console.log('The server is running!');
```


运行 `node server.js`，服务器就跑起来了。
我更建议使用 [node monitor]()，每次修改 server 代码后，不需要手动重启。

`nodemon server.js`
![第一张截图]()


接着，我们从客户端发起 HTTP 请求。 To send a request, 我们需要
 * 为 IP 指定 host
 * 为 TCP 指定 port
 * 为 HTTP 指定 method，path，headers and body
 
like this。然后在立即执行函数中执行。

./client/index.js
```
void async function () {
  const request = new Request({
    host: '127.0.0.1',
    port: 8080,
    method: 'POST',
    path: '/',
    headers: {
      ['X-Jobpal']: 'chatbots',
    },
    body: {
      name: 'Jobpal',
    }
  });

  const reponse = await request.send();
  console.log(response);
}()
```

然后，我们根据这些需求，来实现 Request 类。

如以下代码所示，我们在构造函数中，为相应属性添加默认值。

在处理 headers 时，需要特别注意 'Content-Type'。（？？必需的 request header？？）It is required, otherwise the body can not be parsered correctly，因为不同格式的 body 解析方式不同。

一些常见的 body 格式是： These are four mosted used format:
* application/json
* application/x-www-form-urlencoded
* multipart/form-data
* text/xml

我们使用 HTML 的 form 标签提交产生的 HTML 请求，默认会产生 application/x-www-form-urlencoded 的数据格式，当有文件上传时，则会使用 multipart/form-data。

这里我们把默认值设为 application/x-www-form-urlencoded。这种格式下，提交的数据按照 key1=val1&key2=val2 的方式进行编码，key 和 val 都进行了 URL 转码

关于 encodeURIComponent 可以参考 https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/encodeURIComponent

在 send 方法中，我们把构建好的请求发送出去（这里需要用到'net' 包来创建 TCP 链接），如果没有出错，将

./client/index.js
```
const net = require('net');

class Request {
  constructor({
    method,
    host,
    port,
    path,
    headers,
    body
  }) {
    this.host = host;
    this.port = port || 80;
    this.method = method || 'GET';
    this.path = path || '/';
    this.headers = headers || {};
    this.body = body || {};

    if (!this.headers['Content-Type']) {
      this.headers['Content-Type'] = 'application/x-www-form-urlencoded';
    }
    // encode the http body.
    if (this.headers['Content-Type'] === 'application/json') {
      this.bodyText = JSON.stringify(this.body);
    } else if (this.headers['Content-Type'] === 'application/x-www-form-urlencoded') {
      this.bodyText = Object.keys(this.body).map(
        key => `${key}=${encodeURIComponent(this.body[key])}`
      ).join('&');
    }

    this.headers["Content-Length"] = this.bodyText.length;
  }


  // The parameter is a TCP connection.
  send(connection) {
    return new Promise((resolve, reject) => {
      const parser = new ResponseParser;

      if (connection) {
        connection.write(this.toString());
      } else {
        // create a TCP connection
        connection = net.createConnection(
          { host: this.host, port: this.port },
          () => connection.write(this.toString())
        );
      }

      connection.on('data', data => {
        console.log(data.toString());
        console.log(JSON.stringify(data.toString(), null, 4));
      });

      connection.on('error', error => {
        console.log(error);
        reject(error);
        connection.end();
      })
    })
  }

  toString() {
    // http messages https://developer.mozilla.org/en-US/docs/Web/HTTP/Messages
    return `${this.method} ${this.path} HTTP/1.1\r
${Object.keys(this.headers).map(key => `${key}: ${this.headers[key]}`).join('\r\n')}\r
\r
${this.bodyText}`;
  }
}
```

![](https://static001.geekbang.org/resource/image/3d/a1/3db5e0f362bc276b83c7564430ecb0a1.jpg)
