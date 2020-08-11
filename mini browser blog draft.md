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

### 搭建服务器

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

### 构建并发送 HTTP 请求

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

如以下代码所示，我们在构造函数中，保存这些参数，并为相关属性添加默认值。

在处理 headers 时，需要特别注意 'Content-Type'。（？？必需的 request header？？）It is required, otherwise the body can not be parsered correctly，因为不同格式的 body 解析方式不同。

一些常见的 body 格式是： These are four mosted used format:
* application/json
* application/x-www-form-urlencoded
* multipart/form-data
* text/xml

我们使用 HTML 的 form 标签提交产生的 HTML 请求，默认会产生 application/x-www-form-urlencoded 的数据格式，当有文件上传时，则会使用 multipart/form-data。

这里我们把默认值设为 application/x-www-form-urlencoded。这种格式下，提交的数据按照 key1=val1&key2=val2 的方式进行编码，key 和 val 都进行了 URL 转码

关于 encodeURIComponent 可以参考 https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/encodeURIComponent

在构造函数中保存了参数后，就可以在 toString 方法中构建 HTTP 请求了。如下图所示，HTTP 请求分为 请求行，请求头和请求体。

![](https://static001.geekbang.org/resource/image/3d/a1/3db5e0f362bc276b83c7564430ecb0a1.jpg)

接着，在 send 方法中发出请求，并在请求成功时，把打印返回的 response 打印出来看看。

注意，这里需要用到'net' 包来创建 TCP 链接。


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
        console.log(JSON.stringify(data.toString()));
      });

      connection.on('error', error => {
        console.log(error);
        reject(error);
        connection.end();
      })
    })
  }

  toString() {
    // the HTTP request
    return `${this.method} ${this.path} HTTP/1.1\r
${Object.keys(this.headers).map(key => `${key}: ${this.headers[key]}`).join('\r\n')}\r
\r
${this.bodyText}`;
  }
}
```

![结果截图](cm01 client log request response)

可以看到，response 的结构也符合上图。有意思的是，response body 部分和我们在服务端发送的数据并不完全一样。

在服务端，我们发送的是 ` Hello World\n`
而现在，body 里的数据是 `d\r\n Hello World\n\r\n0\r\n\r\n`

造成这种差异的原因，就是 response headers 中的 `Transfer-Encoding: chunked` https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Transfer-Encoding

// Transfer-Encoding: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Transfer-Encoding

>chunked
>
>Data is sent in a series of chunks. The Content-Length header is omitted in this case and at the beginning of each chunk you need to add the length of the current chunk in hexadecimal format, followed by '\r\n' and then the chunk itself, followed by another '\r\n'. The terminating chunk is a regular chunk, with the exception that its length is zero. It is followed by the trailer, which consists of a (possibly empty) sequence of entity header fields.

所以，'d' 是十六进制数，表示 ` Hello World\n` 有十三个字符。后面的空 chunked 表示结尾。
// trunked body 的结构是，每个trunk，先是长度（那个十六进制数），后面跟着一个 trunk 的内容。
// 遇到长度为 0 的 trunk，整个 body 就结束。

接下来，我们需要解析 http response。

### 解析 HTTP response

#### 解析响应行 响应头

Response 是这样一个字符串，请求行，请求头部分之间用 `\r\n` 来分隔，最后一个请求头和请求体之间是两个 `\r\n`。

`HTTP/1.1 200 OK\r\nContent-Type: text/html\r\nDate: Sun, 02 Aug 2020 17:05:31 GMT\r\nConnection: keep-alive\r\nTransfer-Encoding: chunked\r\n\r\nd\r\n Hello World\n\r\n0\r\n\r\n`

我们可以用状态机来解析，根据该字符串的格式来设计出状态。
// we can design the states according to the format of the response

什么是状态机（？？补充）

增加 ResponseParser 类

./client/index.js
```
class ResponseParser {
  constructor() {
    this.WAITING_STATUS_LINE = 0;
    this.WAITING_STATUS_LINE_END = 1;

    this.WAITING_HEADER_NAME = 2;
    this.WAITING_HEADER_SPACE = 3;
    this.WAITING_HEADER_VALUE = 4;
    this.WAITING_HEADER_LINE_END = 5;
    this.WAITING_HEADER_BLOCK_END = 6;

    this.WAITING_BODY = 7;

    // initial state
    this.current = this.WAITING_STATUS_LINE;

    this.statusLine = '';
    this.headers = {};
    this.headerName = '';
    this.headerValue = '';
    this.bodyParser = null;
  }
  get isFinished() { }

  get response() { }

  receive(string) {
    for (const char of string) {
      this.receiveChar(char);
    }
    console.log(this.statusLine, '\n', this.headers)
  }
  
  // a state machine
  receiveChar(char) {
    switch (this.current) {
      case this.WAITING_STATUS_LINE:
        if (char === '\r') {
          this.current = this.WAITING_STATUS_LINE_END;
        } else {
          this.statusLine += char;
        }
        break;

      case this.WAITING_STATUS_LINE_END:
        if (char === '\n') {
          this.current = this.WAITING_HEADER_NAME;
        }
        break;

      case this.WAITING_HEADER_NAME:
        if (char === ':') {
          this.current = this.WAITING_HEADER_SPACE;
        } else if (char === '\r') {
          // no key-value pairs at all or the end of k-v paris (no more)
          this.current = this.WAITING_HEADER_BLOCK_END;
        } else {
          this.headerName += char;
        }
        break;

      case this.WAITING_HEADER_SPACE:
        if (char === ' ') {
          this.current = this.WAITING_HEADER_VALUE;
        }
        break;
      case this.WAITING_HEADER_VALUE:
        if (char === '\r') {
          this.current = this.WAITING_HEADER_LINE_END;
        } else {
          this.headerValue += char;
        }
        break;

      case this.WAITING_HEADER_LINE_END:
        if (char === '\n') {
          this.current = this.WAITING_HEADER_NAME;
          this.headers = {
            ...this.headers,
            [this.headerName]: this.headerValue
          }
          this.headerName = '';
          this.headerValue = '';
        }
        break;

      case this.WAITING_HEADER_BLOCK_END:
        if (char === '\n') {
          this.current = this.WAITING_BODY;
        }
        break;

      case this.WAITING_BODY:
        console.log(JSON.stringify(char));
        break;

      default:
        break;
    }
  }
}
```

并在 Request 的 send 方法中，做把拿到的数据传给 parser:
```
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
        console.log(JSON.stringify(data.toString()))
        // reveive the data and pass it to the parser
        parser.receive(data.toString());
        if (parser.isFinished) {
          resolve(parser.response);
          connection.end();
        }
      });

      connection.on('error', error => {
        console.log(error);
        reject(error);
        connection.end();
      })
    })
  }
```

执行文件，可以看到，响应行和响应头已经解析成功，响应体的内容也打印了出来。

![](cm01 response state machine)

不同类型的响应体，解析方式各有不同。接下来，我们仅以 chunked 为例，进行解析。


#### 解析响应体

在新增的 ChunkedBodyParser 类中，我们同样使用状态机来解析响应体。

`d\r\n Hello World\n\r\n0\r\n\r\n`

`d     \r\n         Hello World\n       \r\n      0    \r\n     \r\n`

十六进制数字和内容之间都是用 `\r\n` 分隔的，我们据此设计出状态。

./client/index.js
```
class ChunkedBodyParser {
  constructor() {
    this.WAITING_LENGTH = 0;
    this.WAITING_LENGTH_LINE_END = 1;

    this.READING_CHUNK = 2;
    this.WAITING_BLANK_LINE = 3;
    this.WAITING_BLANK_LINE_END = 4;

    this.current = this.WAITING_LENGTH;

    this.content = [];
    this.length = 0;
    this.isFinished = false;
  }

  receiveChar(char) {
    switch (this.current) {
      case this.WAITING_LENGTH:
        if (char === '\r') {
          this.current = this.WAITING_LENGTH_LINE_END;
          if (this.length === 0) {
            this.isFinished = true;
          }
        } else {
          // 十六进制数，一位一位往过读取。如，2AF5，先读 2，再进位，读A，依次类推...
          this.length *= 16;
          this.length += parseInt(char, 16);
        }
        break;

      case this.WAITING_LENGTH_LINE_END:
        if (char === '\n' && !this.isFinished) {
          this.current = this.READING_CHUNK;
        }
        break;

      case this.READING_CHUNK:
        this.content.push(char);
        this.length -= 1;
        if (this.length === 0) {
          this.current = this.WAITING_BLANK_LINE;
        }
        break;

      case this.WAITING_BLANK_LINE:
        if (char === '\r') {
          this.current = this.WAITING_BLANK_LINE_END;
        }
        break;

      case this.WAITING_BLANK_LINE_END:
        if (char === '\n') {
          this.current = this.WAITING_LENGTH;
        }
        break;

      default:
        break;
    }
  }
}
```

然后，在 ResponseParser 的 receiveChar 方法中使用该响应体解析器：

```
receiveChar(char) {
  ...
  case this.WAITING_HEADER_NAME:
        if (char === ':') {
          this.current = this.WAITING_HEADER_SPACE;
        } else if (char === '\r') {
          // no key-value pairs at all or the end of k-v paris (no more)
          this.current = this.WAITING_HEADER_BLOCK_END;
          if (this.headers['Transfer-Encoding'] === 'chunked') {
            this.bodyParser = new ChunkedBodyParser();
          }
        } else {
          this.headerName += char;
        }
        break;
  ...
  case this.WAITING_BODY:
        this.bodyParser.receiveChar(char);
        break;
  ...
}
```

至此，响应行、响应头、响应体全部解析完成，补全 ResponseParser 中的 isFinished 和 response

```
get isFinished() {
  return this.bodyParser && this.bodyParser.isFinished;
}


get response() {
  this.statusLine.match(/HTTP\/1.1 ([0-9]+) ([\s\S]+)/);
  return {
    statusCode: RegExp.$1,
    statusText: RegExp.$2,
    headers: this.headers,
    body: this.bodyParser.content.join('')
  }
}
```

可以看到打印出的解析结果

![](cm01 response body state machine)












