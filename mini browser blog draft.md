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

### 词法解析 tokenization

![步骤总结](https://static001.geekbang.org/resource/image/34/5a/34231687752c11173b7776ba5f4a0e5a.png)

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

#### setup the state machine

我们把状态设计为函数，这样状态迁移代码就非常的简单：

```
let state = data; // initial state HTML 标准里把初始状态称为 data

for (const char of html) {
  state = state(char);
}
```

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
    // text node
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

![](cm02 empty state machine)

不过由于是空的架子，它啥都没干，所以最后的 dom 是 undefined。

而状态机的意义就是，可以在各状态中添加相应的逻辑代码，达到我们的目的。

(我们这一步的目的是，得到一颗 DOM 树。
所以，接下来，我们开始创建元素 elements。)

#### get token

标签种类不外乎 开始 封闭 自封闭，我们在状态机中，**遇到标签结束状态时提交标签token。**

然后创建提交token的 emit 函数。the emit function takes the token generated from the state machine

这一步中，先在 emit 函数中什么都不做，仅仅打印 token。

补充后，parser 代码如下:

./client/parser.js
```
const EOF = Symbol('EOF'); // end of file token
let currentToken = null;

function emit(token) {
  console.log(token);
}

module.exports.parseHTML = function (html) {
  let state = data; // initial state             HTML 标准里把初始状态称为 data

  for (const char of html) {
    // console.log(JSON.stringify(char), state.name)
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
    emit({ type: 'EOF' });
    return;
  } else {
    // emit the text node one by one, we can join them later
    emit({
      type: 'text',
      content: char,
    });
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
    currentToken = {
      type: 'startTag',
      tagName: '',
    }
    return tagName(char); // reconsume
  } else {
    // Parse error
    return;
  }
}
// /
function endTagOpen(char) {
  if (char.match(/^[a-zA-Z]$/)) {
    currentToken = {
      type: 'endTag',
      tagName: ''
    }
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
    currentToken.tagName += char;
    return tagName;
  } else if (char === '>') {
    // the current tag is over, go back to the initial state to parse the next tag
    emit(currentToken);
    return data;
  } else {
    return tagName;
  }
}

// 已经 <div/ 了，后面只有跟 > 是有效的，其他的都报错。
function selfClosingStartTag(char) {
  if (char === ">") {
    currentToken.isSelfClosing = true;
    emit(currentToken) // 补?
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
    emit(currentToken); // 结束
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
    emit(currentToken);
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
    emit(currentToken);
    return data;
  } else if (char === EOF) {

  } else {
    return attributeName(char);
  }
}
```

![](cm2 token 上+下)

我们已经可以获取这些 token 了，不过现在属性还是缺失的。如 HTML 中带有 id 的 img 标签 `<img id="myid"/>`，其 token 是 `{ type: 'startTag', tagName: 'img', isSelfClosing: true }`

添加属性的方法与上一步类似，添加 currentAttribute 变量，并在状态机中补全逻辑

./client/parser.js
```
const EOF = Symbol('EOF'); // end of file token
let currentToken = null;
let currentAttribute = null;

...

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
    emit(currentToken); // 结束
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
    emit(currentToken);
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
    emit(currentToken);
    return data;
  } else if (char === EOF) {

  } else {
    return attributeName(char);
  }
}
```

再次执行，可以看到缺失的属性已经出现了

![](cm02 token img-id attr)

至此，我们就把字符流拆成了词（token）了，接下来我们要把这些简单的词变成 DOM 树。

### 语法分析 SyntaticalParser

#### 构建 DOM 树

在完全符合标准的浏览器中，不一样的 HTML 节点对应了不同的 Node 的子类，我们为了简化，就不完整实现这个继承体系了。我们仅仅把 Node 分为 Element 和 Text。

对于 element，我们通过用栈来匹配开始和结束标签的方案，实现 DOM 树的构建。

具体来说，在 emit 函数在接收 token 的同时，就开始构建 DOM 树，遇到开始标签就入栈，遇到匹配的结束标签就出栈。自封闭标签不需要入栈。

值得一提的是，当发生标签不匹配时，真正的浏览器会做容错处理，我们此处简化为报错。

当接收完所有输入，栈顶就是最后的根节点，我们 DOM 树的产出，就是这个 stack 的第一项。

而对于 Text 节点，我们则需要把相邻的 Text 节点合并起来，我们的做法是当词（token）入栈时，检查栈顶是否是 Text 节点，如果是的话就合并 Text 节点，然后把文本节点添加进 DOM 树。

./client/parser.js
```
const EOF = Symbol('EOF'); // end of file token
let currentToken = null;
let currentAttribute = null;
let currentTextNode = null;

const stack = [{ type: 'document', children: [] }];

module.exports.parseHTML = function (html) {
  let state = data; // initial state

  for (const char of html) {
    state = state(char);
  }
  state = state(EOF);
  // return the DOM tree
  return stack[0];
}

...

function emit(token) {
  let top = stack[stack.length - 1];

  if (token.type === 'startTag') {
    // create the element
    let element = {
      type: 'element',
      children: [],
      attributes: [],
      tagName: token.tagName
    }

    for (const prop in token) {
      if (prop !== "type" && prop !== "tagName") {
        element.attributes.push({
          name: prop,
          value: token[prop],
        })
      }
    }
    // 创造树型关系
    top.children.push(element);
    // 为了能打印出来，暂时注释掉这行，防止循环结构的错误 circular structure
    // element.parent = top;

    if (!token.isSelfClosing) {
      stack.push(element);
    }

    currentTextNode = null;
  } else if (token.type === 'endTag') {
    if (top.tagName !== token.tagName) {
      throw new Error('Tag does not match');
    } else {
      stack.pop();
    }
    currentTextNode = null;
  } else if (token.type === 'text') {
    if (currentTextNode === null) {
      currentTextNode = {
        type: "text",
        content: "",
      }
      top.children.push(currentTextNode);
    }
    currentTextNode.content += token.content;
  }
}
```

再次执行 index.js，就可以看到打印出的 DOM 树啦！不再是 undefined

可以看到，document 根节点下面是 html，在下面是 head 和 body ...

```
{
  "type": "document",
  "children": [
    {
      "type": "element",
      "children": [
        {
          "type": "text",
          "content": "\n      "
        },
        {
          "type": "element",
          "children": [
            {
              "type": "text",
              "content": "\n            "
            },
            {
              "type": "element",
              "children": [
                {
                  "type": "text",
                  "content": "\n      body div #myid{\n        width:100px;\n        background-color: #ff5000;\n      }\n      body div img{\n        width:30px;\n        background-color: #ff1111;\n      }\n        "
                }
              ],
              "attributes": [],
              "tagName": "style"
            },
            {
              "type": "text",
              "content": "\n      "
            }
          ],
          "attributes": [],
          "tagName": "head"
        },
        {
          "type": "text",
          "content": "\n      "
        },
        {
          "type": "element",
          "children": [
            {
              "type": "text",
              "content": "\n        "
            },
            {
              "type": "element",
              "children": [
                {
                  "type": "text",
                  "content": "\n            "
                },
                {
                  "type": "element",
                  "children": [],
                  "attributes": [
                    {
                      "name": "id",
                      "value": "myid"
                    },
                    {
                      "name": "isSelfClosing",
                      "value": true
                    }
                  ],
                  "tagName": "img"
                },
                {
                  "type": "text",
                  "content": "\n            "
                },
                {
                  "type": "element",
                  "children": [],
                  "attributes": [
                    {
                      "name": "isSelfClosing",
                      "value": true
                    }
                  ],
                  "tagName": "img"
                },
                {
                  "type": "text",
                  "content": "\n        "
                }
              ],
              "attributes": [],
              "tagName": "div"
            },
            {
              "type": "text",
              "content": "\n      "
            }
          ],
          "attributes": [],
          "tagName": "body"
        },
        {
          "type": "text",
          "content": "\n      "
        }
      ],
      "attributes": [
        {
          "name": "maaa",
          "value": "a"
        }
      ],
      "tagName": "html"
    }
  ]
}
```


## 第三步 CSS 计算
可以看到，DOM 树中的元素中有 content 有 attr，就是没有 CSS。它们在裸奔！ 下一步，穿上 CSS 漂亮的衣服

注意，CSS computing 也是发生在 DOM 树构建的过程中的。

这一步，我们需要安装 css 库，这是一个 CSS parser，可以把 CSS 解析为 CSS AST 抽象语法树。

`npm i css`

### 收集 CSS 规则

首先，我们需要从 CSS AST 中收集全部的 CSS rules

./client/parser.js
```
const css = require("css");
...

// 用来存放 CSS rules
const rules = []; 

// gather all the CSS rules
function addCSSRules(text) {
  const ast = css.parse(text);
  rules.push(...ast.stylesheet.rules);
}
```

在什么时候执行它呢？在 style 标签结束时，即遇到 </style> 时执行。（忽略其他位置的CSS）

所以，在 emit 方法中

```
...
else if (token.type === 'endTag') {
  if (top.tagName !== token.tagName) {
    throw new Error('Tag does not match');
  } else {
    if (top.tagName === 'style') {
      addCSSRules(top.children[0].content); // 栈顶元素 top 的 children 是当前 element
    }
    stack.pop();
  }
  currentTextNode = null;
}
...
```

此时，如果打印 rules ，就会发现 CSS 规则已经被收集了。

![](cm03 rules)

### 匹配 CSS 规则

CSS computing 即把CSS属性挂载到相匹配的DOM节点上去。

什么时候进行匹配呢，一般来说，CSS 设计会尽量保证所有的选择器都能在 startTag 进入时就能被判断。随着后来高级选择器的加入，该规则有所松动，但大部分规则仍然遵循。即 DOM tree 构建到 startTag 步骤时，就已经可以判断该元素匹配哪些 CSS 规则了。

篇幅所限，我们的示例代码中仅使用简单的选择器。

// 创建一个元素后，立即计算其 CSS

// 真实浏览器中，可能出现 body 中有 style 标签，需要重新计算 CSS 的情况，忽略。

我们把 CSS computing 的逻辑放进 computeCSS 函数里，在 emit 函数中调用:

parser
```
function emit(token) {
  let top = stack[stack.length - 1];

  if (token.type === 'startTag') {
    let element = {...}

    for (const prop in token) {
      ...
    }
    // CSS computing happens during the DOM tree construction 
    computeCSS(element);
    
    top.children.push(element);
    ...
```

在 computeCSS 函数中，为了正确匹配选择器与相应元素，我们需要获取当前元素的父元素序列。如 'div #myid'

`const elements = stack.slice().reverse();`

栈中会存储当前元素所有的父元素。
因为栈是不断变化的，所以用 slice 方法保存当前状态的副本。
reverse 原因：CSS selector 和元素匹配时，先从当前元素开始 逐级往外匹配。如，一个后代选择器 div #myid 前面的 div 不一定是哪个祖先元素，但后面的 #id 一定是当前元素。所以以 子 => 父 ，从内到外的顺序匹配。

另外，我们还需要一个计算 选择器 与 元素 是否匹配的方法。
同样，为了简化，我们此处仅处理这种由简单选择器和空格构成的后代选择器的情况。'div #myid'
然后在 computeCSS 中，取规则与元素进行匹配

parser
```
...

// 假设 selector 是简单选择器
// 简单选择器：.class选择器  #id选择器  tagname选择器
function match(element, selector) {
  if (!selector || !element.attributes) return false;

  if (selector.charAt(0) === '#') { // id selector
    const attr = element.attributes.filter(
      ({ name }) => (name === 'id')
    )[0];
    if (attr && attr.value === selector.replace('#', '')) {
      return true;
    }
  } else if (selector.charAt(0) === '.') { // class selector
    const attr = element.attributes.filter(
      ({ name }) => (name === 'class')
    )[0];
    if (attr && attr.value === selector.replace(".", "")) {
      return true;
    }
  } else { // type selector
    if (element.tagName === selector) return true;
  }
}


function computeCSS(element) {
  const elements = stack.slice().reverse();

  for (const rule of rules) {
    const selectors = rule.selectors[0].split(' ').reverse(); // 见 ast 结构，注意 reverse 对应
    // console.log(selectors) // [ '#myid', 'div', 'body' ]

    if (!match(element, selectors[0])) continue;

    let matched = false;

    let selectorIndex = 1; // elements 是父元素们，所以从1开始
    for (let elementIndex = 0; elementIndex < elements.length; elementIndex++) {
      if (match(elements[elementIndex], selectors[selectorIndex])) {
        selectorIndex++;
      }
    }
    if (selectorIndex >= selectors.length) {
      // all selectors are matched
      matched = true;
    }
    if (matched) {
      console.log(`Selector "${rule.selectors[0]}" has matched element ${JSON.stringify(element)}`)
    }
  }
}

...
```

从打印出的结果可以看出，元素与选择器已经正确匹配了。

>Selector "body div #myid" has matched element {"type":"element","children":[],"attributes":[{"name":"id","value":"myid"},{"name":"isSelfClosing","value":true}],"tagName":"img"}


在成功匹配后，我们下一步应该把 CSS 规则添加到相应的元素上。

这一步很简单，在 computeCSS 函数中，我们为 元素 添加 computedStyle 属性。然后在匹配时，把相应规则挂上去。

```
function computeCSS(element) {
  if (!element.computedStyle) {
    element.computedStyle = {};
  }

  const elements = stack.slice().reverse();

  for (const rule of rules) {
    ...
    
    if (matched) {
      const { computedStyle } = element;
      for (const declaration of rule.declarations) {
        const { property, value } = declaration;
        if (!computedStyle[property]) {
          computedStyle[property] = {};
        }
        computedStyle[property] = value;
      }
    }
  }
  console.log(element.computedStyle)
}
```

元素上已经有了相匹配的 CSS 规则，但这样就足够了吗？显然还不够，我们还得考虑 CSS 优先级的问题，高优先级的规则总应该被优先显示。目前打印出的结果是错的，后来的低优先覆盖了高优先级。

两个 img 元素的的 computedstyle 都是
>{ width: '30px', 'background-color': '#ff1111' }


CSS 是如何计算优先级的？

优先级 specificity
四元组，[inline, id, class/attribute, tagname] e.g. [0, 2, 1, 1] sp = 0 * N3️⃣ + 2 * N² + 1 * N + 1 N 是一个很大的数。 IE 老版本中，为了节省内存，N 取值255，不够大，导致了 256 个 class 相当于一个 id 的bug。 当然，正常人不会写 256 个选择器。 目前大部分浏览器取 65536。

据此，我们实现出计算、比较 CSS 优先级的函数。同样的，这也是简化版本。

```
function getSpecificity(selector) {
  const specificity = [0, 0, 0, 0];
  // 同样，假设没有 selector combinator，都是由简单选择器构成的符合选择器
  const selectors = selector.split(' ');

  for (const item of selectors) {
    if (item.charAt(0) === '#') {
      specificity[1] += 1;
    } else if (item.charAt(0) === '.') {
      specificity[2] += 1;
    } else {
      specificity[3] += 1;
    }
  }
  return specificity;
}

function compareSpecificity(sp1, sp2) {
  if (sp1[0] - sp2[0]) {
    return sp1[0] - sp2[0];
  }
  if (sp1[1] - sp2[1]) {
    return sp1[1] - sp2[1];
  }
  if (sp1[2] - sp2[2]) {
    return sp1[2] - sp2[2];
  }
  return sp1[3] - sp2[3];
}
```

在 computeCSS 函数中，当 match 的时候，不再仅仅简单给 element 挂上样式，而是结合优先级再进行判断。

```
if (matched) {
      const { computedStyle } = element;
      const specificity = getSpecificity(rule.selectors[0]);

      for (const declaration of rule.declarations) {
        const { property, value } = declaration;
        if (!computedStyle[property]) {
          computedStyle[property] = {};
        }
        // computedStyle[property] = value;
        // CSS specificity
        if (!computedStyle[property].specificity) {
          computedStyle[property] = {
            value,
            specificity,
            ...computedStyle[property],
          }
        } else if (compareSpecificity(computedStyle[property].specificity, specificity) < 0) {
          // current CSS selector have higher specificity than the previous, cover the previous rules
          computedStyle[property] = {
            value,
            specificity,
            ...computedStyle[property],
          }
        }
      }
    }
```

执行修改后的代码，我们发现，ID为 myid 的 img 元素的 背景色属性正确显示为 #ff5000

现在，再次打印 DOM 树，就会发现它不再是光秃秃的了，而是加上了 CSS 规则及其优先级。圣诞树。

接下来，浏览器的工作就是确定每一个元素的位置了，目的是得到一颗带位置的 DOM 树


## 第四步 排版

三代排版技术：
* 正常流 position display float
* flex 以此为例实现 https://css-tricks.com/snippets/css/a-guide-to-flexbox/
* grid

我们选择目前如日中天的 flex 排版，进行实现。

### 主轴与交叉轴

![](主轴与交叉轴)

相关概念

举例，当，主轴就是...

好处：简化大量编码工作

### 执行时机
layout 的相关准备工作

首先是确定 layout 的执行时机。计算 flex 布局，需要先获得当前元素的子元素。所以发生在标签结束时


parser
```
const css = require("css");
const layout = require("./layout.js");

...

function emit(token) {
  ...
   else if (token.type === 'endTag') {
    if (top.tagName !== token.tagName) {
      throw new Error('Tag does not match');
    } else {
      if (top.tagName === 'style') {
        addCSSRules(top.children[0].content); // 栈顶元素 top 的 children 是当前 element
      }
      // right here!
      layout(top);
      stack.pop();
    }
    currentTextNode = null;
  }
  ...
}

```

### 准备工作

然后，创建新的 layout 文件

px标识的字符串 => 数字类型 等类型转换

过滤非flex，过滤文本节点


./client/layout.js
```
// pre-processing
function getStyle(element) {
  if (!element.style) element.style = {};

  const { computedStyle } = element;

  for (const prop in computedStyle) {
    element.style[prop] = computedStyle[prop].value;

    // 把 px 单位的转为数字
    if (element.style[prop].toString().match(/px$/)) {
      element.style[prop] = parseInt(element.style[prop]);
    }
    // 把数字字符串转换为数字类型
    if (element.style[prop].toString().match(/^[0-9\.]+$/)) {
      element.style[prop] = parseInt(element.style[prop]);
    }
  }
  return element.style;
}

function layout(element) {
  if (!element.computedStyle) return;

  const elementStyle = getStyle(element);
  // 仅以 flex 布局为例实现
  if (elementStyle.display !== 'flex') return;

  // 过滤文本节点等
  const elementItems = element.children.filter(
    el => el.type === 'element'
  );
}

module.exports = layout; 
```

### 设置默认值

在 layout 函数里

设置flex属性的默认值

不同情况下 主轴 交叉轴 等的赋值

flex-direction wrap

```
function layout(element) {
  if (!element.computedStyle) return;

  const elementStyle = getStyle(element);
  // 仅以 flex 布局为例实现
  if (elementStyle.display !== 'flex') return;

  // 过滤文本节点等
  const elementItems = element.children.filter(
    el => el.type === 'element'
  );

  // to support the order property
  elementItems.sort((a, b) => (a.order || 0) - (b.order || 0));

  const style = elementStyle;

  ['width', 'height'].forEach(size => {
    if (style[size] === 'auto' || style[size] === '') {
      // 把空的变为 null，方便后续代码中进行判断
      style[size] = null;
    }
  })

  // set default value 设置 flex 相关属性的默认值，确保不空
  if (!style.flexDirection || style.flexDirection === 'auto') {
    style.flexDirection = 'row';
  }
  if (!style.alignItems || style.alignItems === 'auto') {
    style.alignItems = 'stretch';
  }
  if (!style.justifyContent || style.justifyContent === 'auto') {
    style.justifyContent = 'flex-start';
  }
  if (!style.flexWrap || style.flexWrap === 'auto') {
    style.flexWrap = 'nowrap';
  }
  if (!style.alignContent || style.alignContent === 'auto') {
    style.alignContent = 'stretch';
  }

  let mainSize,
    mainStart,
    mainEnd,
    mainSign,
    mainBase,
    crossSize,
    crossStart,
    crossEnd,
    crossSign,
    crossBase;

  if (style.flexDirection === 'row') {
    mainSize = 'width';
    mainStart = 'left';
    mainEnd = 'right';
    mainSign = +1;
    mainBase = 0;

    crossSize = 'height';
    crossStart = 'top';
    crossEnd = 'bottom';
  } else if (style.flexDirection === 'row-reverse') {
    mainSize = 'width';
    mainStart = 'right';
    mainEnd = 'left';
    mainSign = -1;
    mainBase = style.width;

    crossSize = 'height';
    crossStart = 'top';
    crossEnd = 'bottom';
  } else if (style.flexDirection === 'column') {
    mainSize = 'height';
    mainStart = 'top';
    mainEnd = 'bottom';
    mainSign = +1;
    mainBase = 0;

    crossSize = 'width';
    crossStart = 'left';
    crossEnd = 'right';
  } else if (style.flexDirection === 'column-reverse') {
    mainSize = 'height';
    mainStart = 'bottom';
    mainEnd = 'top';
    mainSign = -1;
    mainBase = style.height;

    crossSize = 'width';
    crossStart = 'left';
    crossEnd = 'right';
  }

  if (style.flexWrap === 'wrap-reverse') {
    const [crossEnd, crossStart] = [crossStart, crossEnd];
    crossSign = -1;
  } else {
    crossBase = 0;
    crossSign = +1;
  }

}

```

### 收集flex item元素进行内
强制或放得下就放一行；放不下就分行

// 分行：若设置了 no-wrap，则强行分配进第一行；若不是，即大部分情况下，根据主轴尺寸，把元素分进行内；

一个特殊情况：
如果父元素没有设置主轴尺寸(row时为width)，即由子元素把父元素撑开。此处称该模式为 auto main size。该情况下，都是一行

（接着写）
```
  let isAutoMainSize = false;
  if (!style[mainSize]) {
    // auto sizing
    elementStyle[mainSize] = 0;
    for (let i = 0; i < elementItems.length; i++) {
      const itemStyle = getStyle(elementItems[i]);
      if (itemStyle[mainSize] !== null || itemStyle[mainSize] !== (void 0)) {
        elementStyle[mainSize] = elementStyle[mainSize] + itemStyle[mainSize];
      }
    }
    isAutoMainSize = true;
  }
```

正常情况下：

（接着写）
```
// 把元素放进行里 flex 行
  let flexLine = [];
  const flexLines = [flexLine];

  // mainSpace 是剩余空间, 将其设为父元素的主轴尺寸 mainsize ？？？？
  let mainSpace = elementStyle[mainSize];
  let crossSpace = 0;

  // 循环所有的 flex items
  for (let i = 0; i < elementItems.length; i++) {
    const item = elementItems[i];
    const itemStyle = getStyle(item);
    // console.log("🔥", elementStyle)
    // 为单个元素的 空的 主轴尺寸 设置默认值 0 
    if (itemStyle[mainSize] === null) {
      itemStyle[mainSize] = 0;
    }
    // 单行 与 换行 的逻辑
    // 若有属性 flex (不是 display: flex)，说明该元素可伸缩，即一定可以放进 flexLine 里
    if (itemStyle.flex) {
      flexLine.push(item);
    } else if (style.flexWrap === 'nowrap' && isAutoMainSize) {
      mainSpace -= itemStyle[mainSize];
      if (itemStyle[crossSize] !== null && itemStyle[crossSize] !== (void 0)) {
        // e.g. 算行高(当flex direction 为 row时)
        crossSpace = Math.max(crossSpace, itemStyle[crossSize]);
      }
      // 因为 nowrap
      flexLine.push(item);
    } else { // 开始换行的逻辑
      // 若有元素主轴尺寸比父元素还大，则压缩到跟父元素一样大。(举例思考容易理解，如宽度)
      if (itemStyle[mainSize] > style[mainSize]) {
        itemStyle[mainSize] = style[mainSize];
      }
      // 若主轴内剩下的空间不足以容纳每一个元素 ?? 则换行
      if (mainSpace < itemStyle[mainSize]) {
        flexLine.mainSpace = mainSpace // 存储主轴剩余的空间，之后要用
        flexLine.crossSpace = crossSpace; // 存储交叉轴剩余空间
        // 前两行都是处理旧的 flexline  得出该行实际剩余的尺寸，和实际占的尺寸 ？
        // 接下来创建新行。当前的 item 已经放不进旧的行了。
        flexLine = [item]; // 创建一个新的 flex line
        flexLines.push(flexLine);
        // 重置两个属性
        mainSpace = style[mainSize];
        crossSpace = 0;
      } else { // 如果主轴内还能方向该元素
        flexLine.push(item);
      }
      // 接下来 还是算主轴和交叉轴的尺寸
      if (itemStyle[crossSize] !== null && itemStyle[crossSize] !== (void 0)) {
        crossSpace = Math.max(crossSpace, itemStyle[crossSize]);
      }
      mainSpace -= itemStyle[mainSize];
    }
  }
  // set mainSpace and crossSpace
  flexLine.mainSpace = mainSpace;

```



主轴方向的计算：
1. 找出所有 flex items
  . 把主轴方向的剩余空间 mainspace 按比例分配给这些 flex items
  . 特情处理：若剩余空间为负数（如no-wrap,不能自由换行），把所有flex items元素主轴方向尺寸置位为0，等比例压缩剩余元素
2. 如果没有 flex items
  根据 justify-content 来计算每个元素的位置

（接着写）
```
  if (style.flexWrap === "nowrap" || isAutoMainSize) {
    flexLine.crossSpace = (style[crossSize] !== undefined)
      ? style[crossSize]
      : crossSpace;
  } else {
    flexLine.crossSpace = crossSpace;
  }

  if (mainSpace < 0) {
    // overflow (happens only if container is single line), scale every item 
    // 只会发生在单行，算特情
    // 等比压缩
    let scale = style[mainSize] / (style[mainSize] - mainSpace) // style[mainSize] 是容器的主轴尺寸，它减去 mainSpace 是期望的尺寸
    let currentMain = mainBase;

    // 循环每一个元素，找出样式
    for (let i = 0; i < elementItems.length; i++) {
      const itemStyle = getStyle(elementItems[i]);
      if (itemStyle.flex) {
        // flex 元素不参与等比压缩，故而尺寸设为 0
        itemStyle[mainSize] = 0;
      }

      itemStyle[mainSize] = itemStyle[mainSize] * scale;

      // 以 row 的情况为例，计算压缩之后的 left 和 right。
      itemStyle[mainStart] = currentMain; // currentMain 当前排到哪儿了
      itemStyle[mainEnd] = itemStyle[mainStart] + mainSign * itemStyle[mainSize]; // left（start） 加上宽度，就是 right（end） 的值
      currentMain = itemStyle[mainEnd];
    }
  } else {
    // 多行
    // process each flex line
    flexLines.forEach(flexLine => {
      const mainSpace = flexLine.mainSpace;
      let flexTotal = 0;
      for (let i = 0; i < flexLine.length; i++) {
        let itemStyle = getStyle(flexLine[i]);
        // 在循环中找出 flex 元素，把 flex 加到 flexTotal 上去
        if ((itemStyle.flex !== null) && (itemStyle.flex !== (void 0))) {
          flexTotal += itemStyle.flex;
          continue;
        }
      }

      // 如果flex 元素存在，就把 mianSpace 均匀地分布给每一个 flex 元素 (如果有 flex 元素，永远是占满整个行，justifyContent 属性用不上)
      if (flexTotal > 0) {
        let currentMain = mainBase;
        for (let i = 0; i < flexLine.length; i++) {
          const itemStyle = getStyle(flexLine[i]);
          // 如果是 flex 元素，根据收集元素进行的时候计算得出的 mainSpace（每行的主轴方向的剩余空间），按比例（除以总值，乘以自己的flex）划分，得出这些 flex 元素各自的主轴尺寸
          if (itemStyle.flex) {
            itemStyle[mainSize] = (mainSpace / flexTotal) * itemStyle.flex;
          }
          // 跟前面如出一辙，先给一个 currentMain，它一开始等于 mainBase，每排一个元素，currentMain 就加一个（主轴方向的正负符号*主轴方向的尺寸），算得 mainEnd
          itemStyle[mainStart] = currentMain;
          itemStyle[mainEnd] = itemStyle[mainStart] + mainSign * itemStyle[mainSize];
          currentMain = itemStyle[mainEnd];
        }
      } else {
        // 如果没有 flex 元素，就把主轴方向的剩余空间，根据 justifyContent的规则分配
        let currentMain, gap;
        if (style.justifyContent === 'flex-start') {
          currentMain = mainBase; // 以 row 为例，从左向右排。currentMain 就是 mainBase
          gap = 0; // 每个元素之间没有间隔
        }
        if (style.justifyContent === 'flex-end') {
          currentMain = mainBase + mainSpace * mainSign; // 以 row 为例，从右向左排。currentMain 是 mainBase + mainSpace 剩余空间
          gap = 0; // 每个元素之间没有间隔
        }
        if (style.justifyContent === 'center') {
          currentMain = mainBase + mainSpace / 2 * mainSign;
          gap = 0; // 每个元素之间没有间隔
        }
        if (style.justifyContent === 'space-between') {
          currentMain = mainBase;
          gap = mainSpace / (elementItems.length - 1) * mainSign; // 每个元素直接有间隔，总共有 elementItems.length - 1 个间隔
        }
        if (style.justifyContent === 'space-around') {
          currentMain = gap / 2 + mainBase;
          gap = mainSpace / elementItems.length * mainSign; // 每个元素直接有间隔，总共有 elementItems.length 个间隔
        }
        if (style.justifyContent === 'space-evenly') {
          gap = mainSpace / (elementItems.length + 1) * mainSign
          currentMain = gap + mainBase
        }
        // 所有的元素都是 根据 mainstart 和  mainsize 算 mainend
        for (let i = 0; i < flexLine.length; i++) {
          itemStyle[mainStart] = currentMain;
          itemStyle[mainEnd] = itemStyle[mainStart] + mainSign * itemStyle[mainSize];
          currentMain = itemStyle[mainEnd] + gap;
        }
        // 至此，计算出所有的主轴尺寸。以 row 为例，是 宽width  左left  右right
      }
    })
  }
```


目前已经算出
（row）width left right

下一步计算交叉轴上的属性：height top bottom 

当这六个属性确定，元素的位置也就确定了

交叉轴方向的计算
* 根据每行中最大元素的尺寸确定行高
* 根据 flex-align and  item-align 确定元素具体位置

(接着写)
```
// 计算交叉轴位置的代码
  if (!style[crossSize]) { // 若父元素没有 crossSize， crossSpace 永远为零
    crossSpace = 0;
    elementStyle[crossSize] = 0;
    // 还需要把撑开的高度加上去
    for (let i = 0; i < flexLines.length; i++) {
      elementStyle[crossSize] = elementStyle[crossSize] + flexLines[i].crossSpace;
    }
  } else { // 如果有行高
    // 计算出最终的crossSpace 为crossSpace 减去每行最大crossSpace 剩余空间，用作分配
    crossSpace = style[crossSize];
    for (let i = 0; i < flexLines.length; i++) {
      crossSpace -= flexLines[i].crossSpace; // 剩余的行高
    }
  }

  // wrap-reverse 从尾到头 影响 crossBase
  if (style.flexWrap === 'wrap-reverse') {
    crossBase = style[crossSize];
  } else {
    crossBase = 0;
  }

  // 每行的 size 行高 等于 总体的（多行）交叉轴尺寸 除以 行数
  let lineSize = style[crossSize] / flexLines.length;
  let gap;
  // 根据 alignContent 的属性分配行高，矫正 crossSpace
  if (style.alignContent === 'flex-start') {
    crossBase += 0; // crossBase 增量为零
    gap = 0;
  }
  if (style.alignContent === 'flex-end') {
    crossBase += crossSpace * crossSign; // 增量把 crossspace 放在尾巴上
    gap = 0;
  }
  if (style.alignContent === 'center') {
    crossBase += crossSpace * crossSign / 2; // 剩余空间除以二
    gap = 0;
  }
  if (style.alignContent === 'space-between') {
    crossBase += 0;
    gap = crossSpace / (flexLines.length - 1);
  }
  if (style.alignContent === 'space-around') {
    crossBase += crossSign * gap / 2;
    gap = crossSpace / (flexLines.length);
  }
  if (style.alignContent === 'stretch') {
    crossBase += 0;
    gap = 0;
  }

  flexLines.forEach(flexLine => {
    let lineCrossSize = style.alignContent === 'stretch'
      ? flexLine.crossSpace + crossSpace / flexLines.length // 给剩余空间做分配
      : flexLine.crossSpace; // 填满
    // 计算每个元素的交叉轴尺寸
    for (let i = 0; i < flexLine.length; i++) {
      // const item = flexLine[i];
      let itemStyle = getStyle(flexLine[i]);
      // console.log('🌲', itemStyle)
      let align = itemStyle.alignSelf || style.alignItems; // 元素本身的 alignSelf  优先于 父元素的 align Items

      // 未指定交叉轴尺寸
      if (itemStyle[crossSize] === null) {
        itemStyle[crossSize] = align === 'stretch'
          ? lineCrossSize // 满属性
          : 0;
      }

      if (align === 'flex-start') {
        itemStyle[crossStart] = crossBase;
        itemStyle[crossEnd] = itemStyle[crossStart] + crossSign * itemStyle[crossSize];
      }
      if (align === 'flex-end') {
        itemStyle[crossStart] = crossBase + crossSign * lineCrossSize;
        itemStyle[crossEnd] = itemStyle[crossEnd] - crossSign * itemStyle[crossSize];
      }
      if (align === 'center') {
        itemStyle[crossStart] = crossBase + crossSign * lineCrossSize[crossSize] / 2;
        itemStyle[crossEnd] = itemStyle[crossStart] + crossSign * itemStyle[crossSize];
      }
      if (align === 'stretch') {
        console.log('🌲', itemStyle)
        itemStyle[crossStart] = crossBase;
        itemStyle[crossEnd] = crossBase + crossSign * ((itemStyle[crossSize] !== null && itemStyle[crossSize] !== (void 0)) ? itemStyle[crossSize] : lineCrossSize)
        console.log('\n', '✨', lineCrossSize, '\n', itemStyle)
        itemStyle[crossSize] = crossSign * (itemStyle[crossEnd] - itemStyle[crossStart]);
        // console.log('🌲', crossSize)
        // console.log(itemStyle[crossSize], crossSign, itemStyle[crossEnd], itemStyle[crossStart])
        // console.log(itemStyle)
      }
    }
    crossBase += crossSign * (lineCrossSize + gap);
  })
```


至此，我们得到了一棵带位置的 DOM 树。


### 服务端返回应用 flex 布局的 HTML

./server/server.js
```
     ...
     
     response.end(
        `<html maaa=a >
      <head>
            <style>
      #container {
        width:500px;
        height:300px;
        display:flex;
        background-color:rgb(139,195,74);
      }
      #container #myid {
        width:200px;
        height:100px;
        background-color:rgb(255,235,59);
      }
      #container .c1 {
        flex:1;
        background-color:rgb(56,142,60);
      }
        </style>
      </head>
      <body>
        <div id="container">
            <div id="myid"/>
            <div class="c1" />
        </div>
      </body>
      </html>`);
      ...
      
```

此时，可以在新的 DOM 树中观察到，元素的位置已经计算好了

![](cm04 dom element position)


下一步，把它渲染出来吧！



## 第五步 渲染绘制

在计算机图形学领域里，英文 render 这个词是一个简写，它是特指把模型变成位图的过程。我们把 render 翻译成“渲染”，我们现在的一些框架，也会把“从数据变成 HTML 代码的过程”称为 render，其实我觉得这是非常具有误导性的

这里的位图就是在内存里建立一张二维表格，把一张图片的每个像素对应的颜色保存进去（位图信息也是 DOM 树中占据浏览器内存最多的信息，我们在做内存占用优化时，主要就是考虑这一部分）


绘制依赖图形环境，用了 images  `npm i images`
绘制在 viewport 上进行
与绘制有关的属性：background-color, border, background-image etc.
gradient 需要 webGL 来做，images 做不出

### 渲染单个 flex item

./client/render.js
```
const images = require("images");

function render(viewport, element) {
  if (element.style) { // 检测元素是否有样式
    let img = images(element.style.width, element.style.height); // 根据其宽高创建新的 img 对象
    // 简化，只处理背景色
    if (element.style["background-color"]) {
      let color = element.style["background-color"] || "rgb(0, 0, 0)";
      color.match(/rgb\((\d+),(\d+),(\d+)\)/);
      img.fill(Number(RegExp.$1), Number(RegExp.$2), Number(RegExp.$3), ) // 又尼玛不全
      viewport.draw(
        img,
        element.style.left || 0,
        element.style.top || 0
      );
    }
  }
}

module.exports = render; 
```

./client/index.js
```
const net = require('net');
const images = require("images");
const parser = require('./parser');
const render = require("./render");

...

void async function () {
  const request = new Request({...});

  const response = await request.send();
  const dom = parser.parseHTML(response.body);

  const viewport = images(800, 600);
  render(viewport, dom.children[0].children[3].children[1].children[3]); // 传入视口 和 想要绘制的dom (class="c1" 的 div)
  viewport.save('viewport.jpg');
}()
```

可以看到，成功生成了图片

![](viewport-0.jpg)

### 完整渲染

递归调用子元素的绘制方法，可以完成 DOM 树的绘制
忽略一些不需要绘制的节点 ？
实际的浏览器中，文字绘制是难点，依赖字体库，把字体变成图片再渲染。忽略。
实际浏览器中，还会对图层进行 compositing，忽略。



./client/render.js
```
function render(viewport, element) {
  if (element.style) {
    ...
  }

  if (element.children) {
    for (const child of element.children) {
      render(viewport, child);
    }
  }
}
```

在 index.js 中，替换

`render(viewport, dom.children[0].children[3].children[1].children[3]);` => `render(viewport, dom);`

BANG ✿✿ヽ(°▽°)ノ✿

![](viewport-1.jpg)







<!-----

In this article, I'm going to introduce to you how the browser works by creating a simple mini-browser. It's not completed, just like a toy, but it can show the basic rendering principles. Learning by doing is always effective. Let's start the journey.

(浏览器进程)

# Browser's basic work process

The browser works with the following steps:
Requesting the page from the server using HTTP or HTTPS protocol
Parsing the HTML to build a DOM tree
Computing the CSS of the DOM tree
Rendering the elements according to the CSS properties to get the bitmaps in memory
Compositing bitmap for better performance(optional)
Painting to display the content on the screen
It is a gradual process, like an assembly line in the factory. For example, the rendering engine will not wait until all HTML is parsed before starting to build and layout the DOM tree. In this way, the contents can be displayed on the screen gradually as soon as possible.

In our mini-browser project, we will implement these steps as below.

PICTURE 1

# Step one: network request

## Set up a server

First of all, let's start a simple HTTP server which can send back a simple sentence " Hello World\n".

CODE PIECE 1

After installing the 'http' library and run the index.js file, the server is running!

CODE PIECE 1

PICTURE 2

I would recommend using the node monitor, so you don't have to restart the server by hand every time you change something.


## Construct and send an HTTP request

To send an HTTP request, we need to 
specify the host for IP
specify the port for TCP
specify the method, path, headers and body for HTTP
Like this. We put them in An IIFE (Immediately Invoked Function Expression).

CODE PIECE 2

According to these requirements, we can implement the request class as following.

CODE PIECE 3

In the construction function, we save the parameters and set the default values. When dealing with headers, pay attention to `content-type`. It is required, otherwise, the body can not be parsed correctly, because different formats require different parsing methods.

These are the four most used format:
application/json
application/x-www-form-urlencoded  (when submitting form)
multipart/form-data (when uploading files)
text/xml

In the project, we set the default value as `application/x-www-form-urlencoded`. In this format, the submitted data will be encoded like `key1=value1&key2=value2`.

In `toString()` method, the HTTP request is build. It is composed of a request line, request head, and request body.

PICTURE HTTP FORMAT

Then,  in the `send()` method, we send this request and print the response data when success.

Don't forget to install the 'net' library, we need it to create a TCP connection.

PICTURE

As we can see, the structure of response data conforms to the above figure.

Maybe you have noticed that the response body is different from what we sent on the server-side.
what we sent: ' Hello World\n'
what we get: 'd\r\n Hello World\n\r\n0\r\n\r\n'
The reason is the `Transfer-Encoding: chunked` in the response headers.

link and reference

So in the first chunk, 'd' is a hexadecimal number, indicating that there are 13 characters, which are 'HelloWorld', two spaces, and a '\n'. The second chunk, the empty one, means the body part is over.

We have successfully got the response data. Next, let's parse it.

## Parse HTTP response

### parse response line and response head

As you can see, the response is such a string:
 the request line and the request header are separated by `\r\n`
 there are two `\r\n` between the last request header and the request body
`HTTP/1.1 200 OK\r\nContent-Type: text/html\r\nDate: Sun, 02 Aug 2020 17:05:31 GMT\r\nConnection: keep-alive\r\nTransfer-Encoding: chunked\r\n\r\nd\r\n Hello World\n\r\n0\r\n\r\n`

We can use a state machine to parse this string. The states can be designed according to the format of the response, like below.

link: what is a state machine?

CODE PIECE 4 add response parser

In the index.js, we add the ResponseParser class. Then, in the send method of the Request class, we pass the data to the parser.

Executing the file, we can see that the response line and response header have been parsed successfully, and the content of the response body has been printed.

![](cm01 response state machine)

Different types of response bodies require different parsing methods. Next, we only take chunked as an example.


### parse response body

In the newly added ChunkedBodyParser class, we also use a state machine to parse the response body.

The hexadecimal number and the content are separated by `\r\n`, so we design the states like this.

CODE PIECE 5 ChunkedBodyParser

Then, we use the parser in the receiveChar method of the ResponseParser class.

CODE PIECE 6

At this point, the response line, response header, and response body are all parsed, we can complete the `isFinished` and `response` methods in ResponseParser class.

CODE PIECE 7

This is the printed result.

![](cm01 response body state machine)

Let's returned HTML from the server. After saving the changes, the node monitor should automatically restart the server. Requesting again from the client-side,  the response is printed out.

CODE PIECE 8
PICTURE cm01 HTML 上+下

So far, we have successfully received the HTML from the network request.

Now, we can try to parse HTML to get a DOM tree.

# Step two: HTML parsing

PICTURE

## tokenization (lexical analysis)

A token represents the smallest meaningful unit in the compilation principle. In terms of HTML, 90% of the token that we need for daily development is only about tag start, attributes, tag end, comments, and CDATA nodes.

Take `<p class="a">text</p>` as an example, split it to  the smallest meaningful units, we can get:
 `<p`: the start of an opening tag
`class="a"` attrtubes
`>`  “ the end of an opening tag
text
</p> closing tag
Similar to the previous, we also use a state machine for parsing. Fortunately, the state machine has been designed in the HTML standard, 
what we need to do is just translate it to JavaScript. 

a link to the standard

There are more than 80 states specified, but we don't need so many states to parse our simple HTML. So in our sample code,  some simplifications are made.

## setup an empty state machine

We design the state as a function so that the state transition part is very simple.

CODE PIECE 9

Add a new file "parser.js".

CODE PIECE 10

Then, use the new parser in the "index.js".

CODE PIECE 11

Run the "index.js”,  now the state machine is running! You can see the correspondence between characters and states at each step in the printed log.(↘）

PICTURE `![](cm02 empty state machine)`

The dom is `undefined` because now this state machine is empty, it does nothing. In the next step, let's add the corresponding logic code in these states to build the DOM tree.

## Emit tokens

The tag types are nothing more than the starting-tag, the closing tag, and the self-closing tag. In the state machine, we emit the tag token when we encounter the end state of the tag.(标签结束状态)

The emit function takes the token generated from the state machine. In this step, it does nothing but prints the received token.

So the `parser.js` looks like this.

CODE PIECE 12

PICTURE `![](cm2 token 上+下)`

Now, we get these tokens. You might have noticed that the attributes are missing. For example, the token of the `img` tag with id "myid" is `{ type: 'startTag', tagName: 'img', isSelfClosing: true }`. The id is not included.

The way of adding attributes is similar to the previous step. Add the `currentAttribute` variable and complete the logic in the state machine like below.

CODE PIECE 13

Run the index again, you can find out that now the attributes come back!

PICTURE `![](cm02 token IMG-id attr)`

At this point, we have split the character stream into tokens, and then let's turn these tokens into a DOM tree.

## DOM tree construction (syntactic analysis)

In real browsers, the HTML nodes inheritance from different subclasses of Node. To simplify our code, we only divide Node into Element and Text.

For elements, we construct the DOM tree by using the stack to match the tags.

Specifically, when the emit function receives the token, it starts to build the DOM tree. When it encounters a start tag, it pushes the element into the stack, and when it encounters the matching end tag, it pops the top element of the stack. Self-closing tags do not need to be pushed into the stack.

It is worth mentioning that when a tag mismatch occurs, the real browser will do fault-tolerant processing, while we just throw an error here.

When all tokens are received, the top element (the document element) of the stack is the root node of the DOM tree.

For Text nodes, we need to merge them when they are adjacent. When it is pushed into the stack,  check whether the top node of the stack is a Text node. If so, merge the Text nodes and then add the text nodes to the DOM tree.

CODE PIECE 14

Run the index again, instead of undefined,  now we can see a DOM tree here! Under the document element is the html element, below it is the head element and the body element. 

# CSS computing

Look at the DOM tree we have here.  The structure is correct, but it is bare, without any decoration. Let's make it a beautiful Christmas tree with CSS in this step!

Note that CSS computing also occurs during the construction of the DOM tree.

We need to install the `css` library which can parse the CSS code into a CSS AST.

`link of AST and npm install CSS`

## Gather the CSS rules

First, we need to gather all CSS rules together from the CSS AST.

CODE PIECE 15

Then, execute it at the end of the style tag. (Ignore CSS in other locations)

So, in the emit method:

CODE PIECE 16

If you print the rules, you'll find that the CSS rules have been collected.

PICTURE `![](cm03 rules)`

## Match CSS rules and elements

CSS computing is to add the CSS rules to the matched DOM nodes.

When to do it? Generally speaking, we will try to ensure that all selectors can be judged when `startTag` is entered. With the later addition of advanced selectors, this rule has been loosened. Due to the limited length of the article, we only use simple selectors in our sample code. So when there is a `startTag`, it is already possible to determine which CSS rules the element matches.

In real browsers, there may be a style tag in the body that requires recalculation of CSS. We ignore this kind of situation.

call the `computeCSS` in the emit function.

CODE PIECE 17

In the `computeCSS` function, in order to match the selector with the corresponding element, we need to get the parent element sequence of the current element. Such as'div #myid'

CODE PIECE 18 one line

All parent elements of the current element are stored in the stack. Because the stack is changing, the slice method is used to save a copy of the current state.

When we try to determine whether an element matches the selector, we start with the current element, then its parent element, and step by step outward. Take "div #myid" as an example, the `div` can be any ancestor element, but `#id` must be the current element.

In addition, we need a way to calculate whether the selector matches the element. Also for simplicity, we only deal with the case of descendant selectors composed of simple selectors and spaces, like 'div #myid'.

CODE PIECE 19

From the printed result, we can see that the element and the selector have been correctly matched.

`Selector "body div #myid" has matched element {"type":"element","children":[],"attributes":[{"name":"id","value":"myid"},{"name":"isSelfClosing","value":true}],"tagName":"img"}`

After the successful match, we should add CSS rules to the corresponding elements.

This step is very simple. In the `computeCSS` function, we add a `computedStyle` property to the element and save the CSS rules into it when matching.

CODE PIECE 20

The elements already have their CSS rules, but it's not enough yet. Go check these two img elements, you'll find that they have the same `computedstyle` property, which is wrong.

`{ width: '30px', 'background-color': '#ff1111' }`

Why? The reason is the CSS specificity. The rules with higher specificity should always have higher priority. The currently wrong result is because the later low-specificity rules overwrite the previous high-specificity rules.

To calculate the CSS specificity, we need a quaternion: `[inline, id, class/attr, tagName]`（Number of each type of selectors）.

The calculation formula is: `specificity = 0 * N3️⃣ + 2 * N² + 1 * N + 1 `

N is a big number. In the old version of IE, in order to save memory, the value of N is 255, which is not large enough, resulting in a funny bug that 256 classes are equivalent to one id. Of course, any sane developer would not write 256 selectors. Nowadays, most browsers set N as 65536.

Based on this, we have implemented functions for calculating and comparing CSS priorities. Again, this is also a simplified version.

CODE PIECE 21

In the `computeCSS` function, when an element matches a selector, instead of simply apply the CSS rules, we need to compare the CSS specificity and use the rules with higher specificity.

CODE PIECE 22 `if matched`

Run it again, as it printed, the `img` element with ID is correctly displayed as #ff5000.

Now, if you print the DOM tree again, you will see a beautiful Christmas tree with all the CSS decoration.

Next, our job is to calculate the position of each element so we can get a DOM tree with position information.

# Layout

In CSS, there are three generations of layout technology: normal flow, flexbox, and grid.

We are going to implement the most popular one: flexbox.  If you are not familiar with this technique, please check this link.

## main axis and cross axis

Before we start, we must first understand the concepts of the main axis and the cross axis. By default, the main axis is horizontal (from left to right) and the cross axis is vertical (top to bottom). It can also change with our settings. For example, when the value of 'flex-direction' is 'row', the main axis is horizontal, when the value is 'column', the main axis is vertical. Using these concepts can help us reduce a lot of unnecessary `if-else` code.

PICTURE

## when to do the layout

Suppose we already have the layout function, when should we call it? To calculate the flexbox layout, we need to know all the children elements of the current element. So we should call the layout method in the 'endTag' branch.

CODE  PIECE 23 parser

## pre-processing

In the newly created `layout.js` file, let's add a `getStyle` function to do some pre-processing work like filtering out unwanted elements and type conversion(string to number).

CODE  PIECE 24

## Set default values

Set default values in the layout function.

CODE  PIECE 25

## Collect elements into flex line

Put the elements in one line if there is enough room or  `no-wrap`, otherwise, start a new line.

There might be a special case: If the parent element does not have a main axis size (like width), the parent element is stretched by the child element. This mode is called the "auto main size". In this case, the elements are gathered in one line.

CODE PIECE 26 special case

CODE PIECE 27 normal case

## calculate the main axis

Find all flex items and assign the remaining space mainspace in the main axis direction to these flex items proportionally.  Special case: if the remaining space is a negative number (like the "no-wrap" mode), set the size on the main axis of all flex items elements to 0, and compress the remaining elements proportionally

If there are no flex items, calculate the position of each element according to the value of `justify-content`.

CODE PIECE 28 normal case

If the flex-direction is "row", at this point we have the value of `width`, `left`, and `right`. The next step is to calculate the cross axis to get the value of `height`, `top`, and `bottom`.  When these are determined, the position of the element is also determined.

## calculate the cross axis

determine the height of the line based on the size of the largest element
calculate the specific position of elements according to the value of  `flex-align` and `item-align`

CODE PIECE 29

Now, we have a DOM tree with the position data!

## send an HTML with flexbox layout from the server

CODE PIECE 30

PICTURE (![]cm04 dom element position)

As you can see from the new DOM tree, the positions of the elements are already there. Let's render it in the next step!

# Render

In the field of computer graphics, the term "render" specifically refers to the process of turning a model into a bitmap. Note that some frameworks, like React, call the "process from data to HTML code" as to render, which is different.

The bitmap here is a two-dimensional table built in memory, and the color corresponding to each pixel of a picture is saved.

Here we use the images library for painting in the viewport, it supports painting background-color, border, background-image, etc.

`npm install images`

## render one flex item

In the beginning, let's try to render one flex item. 

CODE PIECE 31

CODE PIECE 32

Go check the viewport.jpg, we rendered the item out successfully!

PICTURE

## Full rendering

This step is quite easy, calling the drawing method of the child elements recursively can render the whole DOM tree. 

Text drawing is difficult, it relying on the font library to turn the font into a picture. We ignore it in our mini-browser project.

CODE PIECE 33
PICTURE

Run the index.js and check the picture again, you'll find that the elements are rendered perfectly. Let's change some flex rules to see whether the browser can handle it, like this.

PICTURE

The answer is yes!










----->








