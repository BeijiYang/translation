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

我们将要实现以下步骤
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

什么是状态机（？？补充）参考这段改一下
>根据这样的分析，现在我们讲讲浏览器是如何用代码实现，我们设想，代码开始从 HTTP 协议收到的字符流读取字符。在接受第一个字符之前，我们完全无法判断这是哪一个词（token），不过，随着我们接受的字符越来越多，拼出其他的内容可能性就越来越少。比如，假设我们接受了一个字符“ < ” 我们一下子就知道这不是一个文本节点啦。之后我们再读一个字符，比如就是 x，那么我们一下子就知道这不是注释和 CDATA 了，接下来我们就一直读，直到遇到“>”或者空格，这样就得到了一个完整的词（token）了。实际上，我们每读入一个字符，其实都要做一次决策，而且这些决定是跟“当前状态”有关的。在这样的条件下，浏览器工程师要想实现把字符流解析成词（token），最常见的方案就是使用状态机。 

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


我们从服务端返回 HTML，保存修改后，node monitor 自动重启了服务端。客户端重新请求，成功获得新的返回值。

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
      response.end(
        `<html maaa=a >
    <head>
          <style>
    body div #myid{
      width:100px;
      background-color: #ff5000;
    }
    body div img{
      width:30px;
      background-color: #ff1111;
    }
      </style>
    </head>
    <body>
      <div>
          <img id="myid"/>
          <img />
      </div>
    </body>
    </html>`);
    });
}).listen(8080);

console.log('The server is running!');
```

![cm01 html 上+下]()


至此，我们已经成功地通过网络请求，获得了 HTML 文件。

下一步，我们解析 HTML ，获得 DOM tree。

## 第二步 HTML 解析

### 词法解析 tokenlization

[步骤总结](https://static001.geekbang.org/resource/image/34/5a/34231687752c11173b7776ba5f4a0e5a.png)

tokenlization 是什么 ？？
HTML 的结构不算太复杂，我们日常开发需要的 90% 的“词”（指编译原理的术语 token，表示最小的有意义的单元），种类大约只有标签开始、属性、标签结束、注释、CDATA 节点几种。

以 `<p class="a">text</p>` 为例，从最小有意义单元的定义来拆分，可以分为：
* <p“标签开始”的开始；
* class=“a” 属性；
* >  “标签开始”的结束；
* text 文本；
* </p> 标签结束。

与前文类似，我们同样采用状态机来进行解析。只不过，HTML 标准已经把状态机设计好了，参考 [标准](https://dev.w3.org/html5/spec-LC/tokenization.html)。
我们要做的，仅仅是用 JavaScript 翻译出来。
标准中规定了 80 多个状态，示例代码中仅取一小部分，并做一些简化，足够解析从后端收到的 HTML 即可。

我们把状态设计为函数，这样状态迁移代码就非常的简单：

```
let state = data; // initial state HTML 标准里把初始状态称为 data

for (const char of html) {
  state = state(char);
}
```

// 标签种类 开始 封闭 自封闭

据此，我们增加新的 parser.js

set up a empty state machine

./client/parser.js
```
const EOF = Symbol('EOF'); // end of file token

module.exports.parseHTML = function (html) {
  let state = data; // initial state             HTML 标准里把初始状态称为 data

  for (const char of html) {
    console.log(JSON.stringify(char), state.name)
    state = state(char);
  }
  state = state(EOF);
}

// There three kinds of HTML tags: opening tag <div>, closing tag </div>, self-colsing tag <div/>
// initial state             HTML 标准里把初始状态称为 data
function data(char) {
  if (char === '<') {
    return tagOpen;
  } else if (char === EOF) {
    return;
  } else {
    return data;
  }
}

// when the state is tagOpen, we don't know what kind of tag it is. 
// <
function tagOpen(char) {
  if (char === '/') {
    // </    </div>
    return endTagOpen;
  } else if (char.match(/^[a-zA-Z]$/)) {
    // the char is a letter, the tag could be a opening tag or a self-closing tag
    // <d      <div> or </div>
    return tagName(char); // reconsume
  } else {
    // Parse error
    return;
  }
}
// /
function endTagOpen(char) {
  if (char.match(/^[a-zA-Z]$/)) {
    return tagName(char);
  } else if (char === '>') {
    // error  />  It's html, not JSX
    // Parse error
  } else if (char === EOF) {
    // Parse error
  } else {
    // Parse error
  }
}

function tagName(char) {
  if (char.match(/^[\t\n\f ]$/)) {
    // tagname start from a '<', end with a ' '
    // <div prop
    return beforeAttributeName;
  } else if (char === '/') {
    return selfClosingStartTag;
  } else if (char.match(/^[a-zA-Z]$/)) {
    return tagName;
  } else if (char === '>') {
    // the current tag is over, go back to the initial state to parse the next tag
    return data;
  } else {
    return tagName;
  }
}

// 已经 <div/ 了，后面只有跟 > 是有效的，其他的都报错。
function selfClosingStartTag(char) {
  if (char === ">") {
    return data;
  } else if (char === EOF) {

  } else {

  }
}

function beforeAttributeName(char) {
  if (char.match(/^[\t\n\f ]$/)) { // 当标签结束
    return beforeAttributeName;
  } else if (char === "/" || char === ">" || char === EOF) {
    // 属性结束
    return afterAttributeName(char);
  } else if (char === "=") {
    // 属性开始的时候，不会直接就是等号，报错
    // return
  } else {
    return attributeName(char);
  }
}

function attributeName(char) {
  if (char.match(/^[\t\n\f ]$/) || char === "/" || char === EOF) { // 一个完整的属性结束 "<div class='abc' "
    return afterAttributeName(char);
  } else if (char === "=") { // class= 可以进入获取value的状态
    return beforeAttributeValue;
  } else if (char === "\u0000") { // null

  } else if (char === "\"" || char === "'" || char === "<") { // 双引号 单引号 <

  } else {
    return attributeName;
  }
}


function attributeName(char) {
  if (char.match(/^[\t\n\f ]$/) || char === "/" || char === EOF) { // 一个完整的属性结束 "<div class='abc' "
    return afterAttributeName(char);
  } else if (char === "=") { // class= 可以进入获取value的状态
    return beforeAttributeValue;
  } else if (char === "\u0000") { // null

  } else if (char === "\"" || char === "'" || char === "<") { // 双引号 单引号 <

  } else {
    return attributeName;
  }
}

function beforeAttributeValue(char) {
  if (char.match(/^[\t\n\f ]$/) || char === "/" || char === ">" || char === EOF) {
    return beforeAttributeValue; //?
  } else if (char === "\"") {
    return doubleQuotedAttributeValue; // <html attribute="
  } else if (char === "\'") {
    return singleQuotedAttributeValue; // <html attribute='
  } else if (char === ">") {
    // return data
  } else {
    return UnquotedAttributeValue(char); // <html attribute=
  }
}

function doubleQuotedAttributeValue(char) {
  if (char === "\"") { // 第二个双引号，结束
    return afterQuotedAttributeValue;
  } else if (char === "\u0000") {

  } else if (char === EOF) {

  } else {
    return doubleQuotedAttributeValue;
  }
}

function singleQuotedAttributeValue(char) {
  if (char === "\'") {
    return afterQuotedAttributeValue;
  } else if (char === "\u0000") {

  } else if (char === EOF) {

  } else {
    return singleQuotedAttributeValue;
  }
}

// 所有的 属性结束时，把其 Attribute name、value 写到 current token，即当前的标签上
function UnquotedAttributeValue(char) {
  if (char.match(/^[\t\n\f ]$/)) { // Unquoted Attribute value 以空白符结束
    return beforeAttributeName; // 因为空白符是结束的标志， “<html maaa=a ” 把相关值挂到token上后，接下的状态可能又是一个新的 attribute name
  } else if (char === "/") {
    return selfClosingStartTag; // 同上，自封闭标签的结束
  } else if (char === ">") {
    return data;
  } else if (char === "\u0000") {

  } else if (char === "\"" || char === "'" || char === "<" || char === "=" || char === "`") {

  } else if (char === EOF) {

  } else {
    return UnquotedAttributeValue;
  }
}

// afterQuotedAttributeValue 状态只能在 double quoted 和 single quoted 之后进入。
// 不能直接接收一个字符 如： "<div id='a'"" 这之后至少得有一个空格才可以，紧挨着如 "<div id='a'class=""" 是不合法的 "<div id='a' class=""" 才行
function afterQuotedAttributeValue(char) {
  if (char.match(/^[\t\n\f ]$/)) {
    return beforeAttributeName;
  } else if (char === "/") {
    return selfClosingStartTag;
  } else if (char === ">") { // 标签结束，emit token
    return data;
  } else if (char === EOF) {

  } else {
    return doubleQuotedAttributeValue;
  }
}

function afterAttributeName(char) {
  if (char.match(/^[\t\n\f ]$/)) {
    return afterAttributeName;
  } else if (char === "/") {
    return selfClosingStartTag;
  } else if (char === "=") {
    return beforeAttributeValue;
  } else if (char === ">") {
    return data;
  } else if (char === EOF) {

  } else {
    return attributeName(char);
  }
}
```

然后在 index.js 中使用它：
./client/index.js
```
const parser = require('./parser');
...

void async function () {
  const request = new Request({
   ...
  });

  const response = await request.send();
  const dom = parser.parseHTML(response.body);
  console.log(JSON.stringify(dom, null, 2));
}()
```
运行 index.js，可以看到状态机能跑起来了，在 console 里可以看出每一步 字符和状态的对应关系 （错一位，图上画箭头，↘）

不过由于是空的架子，它啥都没干，所以最后的 dom 是 undefined。

而状态机的意义就是，可以在各状态中添加相应的逻辑代码，达到我们的目的。我们这一步的目的是，得到一颗 DOM 树。

所以，接下来，我们开始创建元素 elements。











