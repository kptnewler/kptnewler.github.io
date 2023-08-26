---
title: Http介绍
date: 2023-08-26 11:19:37
tags: 
- http 
- 计算机网络
categories: 
- 计算机网络
---

<meta name="referrer" content="no-referrer" />

# HTTP协议介绍

HTTP协议，超文本传输协议，表示两个计算机通信点之间传输文字、图片、视频等超文本数据的约定和规范。
它不关心寻址、路由、数据完整性等传输细节，而要求这些工作都由下层的TCP协议来处理，提供可靠，字节流形式的通信。

在传输过程中图片、视频、文字等超文本数据和传输参数都会转换为字符，把这些字符连起来发送给接收方，接收方接收到这些字符根本不知道这些字符的含义是什么，也就无法解析。
就好比一个发了一大串没有联系的字母，另外一个人根本不知道这些到底什么意思。

有了HTTP协议，就好比有了一个说明书，通过key-value的形式，知道每个key及对应的value是什么意思，比如`Accept`表示的就是客户机支持的数据类型，`Date`表示请求的时间是什么，接收方通过解析知道发送方的意图，再根据HTTP协议作出响应。

HTTP协议是应用层协议。TCP/IP协议栈对应4层网络模型分别是应用层、传输层、网际层、链路层。传输资源根据HTTP协议包装好，包装好的数据依靠底下3层和对应的协议发送给接收方。

HTML和HTTP协议密切相关，但是不要把HTTP协议和HTML混为一谈。HTML是一种标记语言，使用各种标签描述文字、图片、超链接等资源，是超文本资源的载体，相当于给传输的超文本资源加上描述符。
浏览器根据HTML标签，知道资源要如何展示出来，比如`<h1></h1>`，其中的文字被标签描述为标题，浏览器给它加大加粗。所以HTML数据也是传输数据的一种。

可以参考[MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP)进行学习。

## HTTP特点

HTTP 是灵活可扩展的，可以任意添加头字段实现任意功能；
HTTP 是可靠传输协议，基于 TCP/IP 协议“尽量”保证数据的送达；
HTTP 是应用层协议，比 FTP、SSH 等更通用功能更多，能够传输任意数据；
HTTP 使用了请求 - 应答模式，客户端主动发起请求，服务器被动回复请求；
HTTP 本质上是无状态的，每个请求都是互相独立、毫无关联的，协议不要求客户端或服务器记录请求相关的信息，通过cookie技术实现有状态。
HTTP 是明文传输，数据完全肉眼可见，能够方便地研究分析，但也容易被窃听
HTTP 是不安全的，无法验证通信双方的身份，也不能判断报文是否被窜改
HTTP 有性能瓶颈，请求是顺序发送的，当队头请求被堵塞，后面排队的请求也会被堵塞，这就是队头堵塞。

## HTTP报文

HTTP报文如下图所示，由起始行、头部、空行、实体组成，实体不是必须的，可以不包含实体。

HTTP报文可以分为请求报文和响应报文两类。
请求报文是客户端向服务器发起请求的HTTP报文，包含客户端请求的信息。
响应报文是服务端给客户端的响应的HTTP报文，包含服务端响应的信息。

下面依次介绍HTTP报文的组成部分。
![HTTP报文组成](https://cdn.jsdelivr.net/gh/kptnewler/blog-image/blogs/picture/20230826165411.png)
### 起始行

起始行可以根据HTTP请求报文和响应报文分成，请求行和响应行。

### 请求行

请求行有**请求方法**、**请求URL**、**HTTP版本号**组成，并用空格隔开。比如`GET /user/login HTTP/1.1`。

![请求行](https://upload-images.jianshu.io/upload_images/4538003-dc408b54368b8a39.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

请求方法：表示操作服务端超文本资源的方式。比如请求修改资源，获取资源，上传资源等等。操作方式由对应的请求方法表示，这里简单介绍一下。最常用的是`GET`和`POST`请求方法。

1. GET：请求获取资源，即希望服务器返回指定的资源内容。
2. POST：提交资源，即向服务器新增超文本资源。
3. PUT：修改资源，即向服务器修改已有的资源内容。
4. DELETE：删除资源，即从服务器删除某个资源。
5. HEAD：获取资源元信息，即从服务器返回资源的信息，不需要内容。
6. OPTIONS：列出可对资源实行的方法
7. CONNECT：建立特殊的连接隧道
8. TRACE：追踪请求 - 响应的传输路径。

URL：表示请求服务端超文本资源的地址，用于定位服务端超文本资源。URL可以只是路径部分，因为协议名和主机名已经分别出现在了请求行的版本号和请求头部字段的`Host`字段里，没有必要再重复。当然，在请求行里使用完整的 URI 也是可以的。

HTTP版本号：目前包含HTTP/0.9、HTTP/1.0、HTTP/1.1、HTTP/2，这几个版本。

### 响应行

响应行由HTTP版本号、响应状态码、状态描述组成，并用空格隔开。比如`HTTP/1.1 200 ok`。
![响应行](https://upload-images.jianshu.io/upload_images/4538003-94191192caa447a7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

HTTP版本号：和请求行中的一样。

响应状态码：表示服务器处理请求的结果，客户端可以根据状态码进行下一步处理工作或用于调试。目前主要有41个状态码，也可以自行扩展。状态码主要分为以下几类，没类都有对应的请求码。

1. 1xx：表示提示信息，是协议处理的中间状态，实际能够用到的时候很少，比如101-Switching 协议切换，比如WebSocket，100-continue客户端通过POST发送大数据。
2. 2XX: 表示服务端接收并成功处理了请求，204-No Content，和200意思一样，但没有响应body，206-Partial Content-用于分块和断点续传，服务正确处理结果，但是响应body中数据不完全。
3. 3xx：表示重定向，服务端资源变动，让客户端使用返回的新url请求资源，301-永久重定向，302-临时重定向，主要在搜索引擎上不同，304 - 资源未修改，用于缓存控制。
4. 4xx：表示客户端请求出错。400-请求报文有错误，401-未授权需要登录身份验证，403禁止访问，404资源不存在。
5. 5xx：表示服务端处理请求报错，500-服务器内部出错，501-客户端请求功能不支持，502-服务器作为网关或代理返回，访问出错，503-服务器很忙无法响应服务。

状态描述：表示状态码的描述信息。

### 头部介绍

头部分为请求头和响应头，使用key-value的形式。
除了可以使用http协议提供了标准的头部字段或自定义头部字段作为key值。
HTTP标准头部字段主要分为：

- 通用字段：在请求头和响应头里都可以出现。
- 请求字段：仅能出现在请求头里，进一步说明请求信息或者额外的附加条件。
- 响应字段：仅能出现在响应头里，补充说明响应报文的信息；
- 实体字段：它实际上属于通用字段，但专门描述 body 的额外信息。

下面看看头部字段的具体应用场景，按照功能能主要划分为**描述信息、实体数据传输、文件传输、连接管理、重定向、Cookie管理、HTTP代理、缓存**，下面根据应用场景一一介绍HTTP的头部字段。

#### 描述说明

描述请求或响应的信息说明，比如时间、主机名等等。
比如`host:` 如果服务器一个IP地址下有多个虚拟主机，就可以通过`host`来找到接收请求的主机。

#### 实体数据传输

客户端请求服务端，要告知服务端，客户端支持处理的资源信息，比如类型，语言，字符集。请求时通过请求头部字段添加支持处理的资源描述信息。

服务端响应客户端，也要告知客户端，响应的资源信息，客户端根据资源信息，对资源进行处理，通过实体头部字段添加响应的资源描述信息。

如果客户端使用POST请求，则添加资源，客户端会携带资源发送给服务端，客户端也要通过实体头部字段添加请求的资源描述信息。

如果没有资源实体描述信息，客户端和服务端每次都要检查资源的内容来获取资源信息，如类型、长度等，非常低效和不准确。

##### Content-Length

`Content-Length`: 表示客户端提交资源实体数据的长度或服务器响应返回实体数据的长度，属于实体首部。

因为Body部分，没有准确的结束符，比如`\n`，所以需要根据长度来提取出实体数据，如果客户端给`Content-Length`和实体数据长度不匹配，可能会造成服务端接收的数据被截断。

##### MIME介绍

MIME，全名为多用途互联网邮件扩展，最早应用于电子邮件传输。
HTTP协议传输和邮件传输类似，可以传输多种格式的数据，比如图片、文件、音乐、视频等。
MIME类型的格式是`<MIME_type>/<MIME_subtype>`,如果`text/html`、`image/png`，前面是主类别，后面是主类别下的子类别。
如果`type`为通配符`*`,表示MIME的所有主类别，`subtype`为通配符`*`表示主类别下的所有子类别，比如所有图片类型`image/*`，所有MIME类型`*/*`。

常用的MIME有以下几种，所有的MIME类型可以查询[所有MIME类型](https://www.iana.org/assignments/media-types/media-types.xhtml)

- `text`：表示文本格式类型，子类别如超文本`text/html`、css格式`text/css`、纯文本格式`text/plain`等。

- `image`：表示图片格式类型，子类别如png格式`image/png`、gif格式`image/gif`、jpg格式`image/jpeg`等。

- `audio`：表示音频格式类型，子类别如mp3格式`audio/mpeg`、wav格式`audio/wav`等。

- `video`：表示视频格式类型，子类别如mp4格式`video/mp4`等。

- `application`：表示数据格式不固定，可能是文本也可能是二进制，必须由上层应用程序来解释。比如json格式`application/json`、pdf格式`application/pdf`、js格式`application/javaScript`。如果无法判断具体格式类型，则使用`application/octet-stream`，可以用在所有格式类型的资源上，服务端返回此种类型格式，浏览器会把当做文件下载。

- `multipart`：表示数据包含多种格式类型，如用于表单数据传输`multipart/form-data`。

**客户端和服务端根据头部字段判断传输实体的类型，做相应的处理。**

##### Accept
`Accept`表示客户端支持的资源MIME类型，属于请求字段，只能用于HTTP请求头。

`Accept`中的值为MIME类型和权重，两者用`;`隔开，格式为`<MIME_type>/<MIME_subtype>;q=value`。
`Accept`的值可以为多种MIME类型，每个值用`,`隔开，表示客户端支持多种资源MIME类型。
权重表示客户端最希望服务器返回的资源类型。`q`的最高权重为1,最小权重为0.01。如果没有`q`，则默认为最高权重

如下`Accept`头部信息表示：
客户端最希望返回的是html类型的资源，权重为1。
其次为xml类型的资源，权重为0.9。
最后是其他任意MIME类型的资源，权重为0.8。

```http
Accept: text/html,application/xml;q=0.9,*/*;q=0.8
```

#### Content-Type
`Content-Type` 表示客户端上传或服务端返回的资源MIME类型，对方接收到数据后根据`Content-Type` 判断出具体数据类型，方便处理。
比如客户端上传json字符串，如果不注明`application/json`，服务端就不知道数据类型，也不会调用JSON转实体函数。

`Content-Type`的值只能用一种MIME类型表示，如果实体数据包含多种类型数据则用MIME类型中`multipart`表示。
`Content-Type`中MIME类型后面还会有实体数据的字符集，因为没有单独的头部字段表示实体数据的字符集。
MIME类型和字符集用`;`隔开。

表示资源实体数据为html，字符集为UTF-8。
```http
Content-Type: text/html; charset=UTF-8
```
`Content-Type` 常用用法如下：
  - text/html: 在服务器响应头中，表示返回的是HTML文本。
  - application/json，在客户端请求头中，表示上传的是json数据。在服务器响应头中，表示返回的是json数据。
  - image/jpeg等图片、视频格式在服务端响应头中用于返回的多媒体文件类型。
  - application/octet-stream ，在客户端请求头，可以用于单文件上传。
  - application/x-www-form-unlencoded，客户端表单上传，只能提交文本类型，表单内的数据转换为键值对，&分隔。
  - multipart/form-data：客户端表单上传，包括文本和文件类型，`boundary`表示分隔符可自定义，每个键值对或文件通过分隔符断开，另外每部分都会用`Content-Type`表示当前数据的类型。
```http
POST /upload.do HTTP/1.1
User-Agent: SOHUWapRebot
Accept-Language: zh-cn,zh;q=0.5
Accept-Charset: GBK,utf-8;q=0.7,*;q=0.7
Connection: keep-alive
Content-Length: 60408
Content-Type:multipart/form-data; boundary=ZnGpDtePMx0KrHh_G0X99Yef9r8JZsRJSXC
Host: www.sohu.com

--ZnGpDtePMx0KrHh_G0X99Yef9r8JZsRJSXC

// 文本
Content-Disposition: form-data;name="desc"
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
[......][......][......][......]...........................

--ZnGpDtePMx0KrHh_G0X99Yef9r8JZsRJSXC

// 图片
Content-Disposition: form-data;name="pic"; filename="photo.jpg"
Content-Type: application/octet-stream
Content-Transfer-Encoding: binary
[图片二进制数据]

--ZnGpDtePMx0KrHh_G0X99Yef9r8JZsRJSXC--
```
##### Content-Disposition

我们可以看到上面出现了 `Content-Disposition`。

`Content-Disposition`:表示服务端返回给客户端的资源表现形式。

** 如果应用在单MIME类型实体资源上，属于响应首部 **，有两种类型，浏览器根据类型不同处理方式也不同。

- `inline`: 服务端告知客户端将返回的资源直接显示出来，类似邮件发送将文件内容直接显示在正文中。

- `attachment`: 服务端告知客户端，返回的资源作为附件。客户端如果为浏览器，则会直接弹出下载对话框。类似邮件发送中将文件作为邮件的附件。

`filename`表示返回资源的文件名。

如下报文所示，服务端返回的`Content-Disposition`类型为`inline`，浏览器会直接显示文件中的内容。

![inline报文](https://upload-images.jianshu.io/upload_images/4538003-f78427a4f0d67575.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![inline浏览器显示](https://upload-images.jianshu.io/upload_images/4538003-65eb3628f6d9dbfc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如下报文所示，服务端返回的`Content-Disposition`类型为`attachment`，浏览器会弹出下载框，文件名`filename`中的值。

![attachment报文](https://upload-images.jianshu.io/upload_images/4538003-b5ac299e0a063a8f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![attachment浏览器展示](https://upload-images.jianshu.io/upload_images/4538003-e9a8ee889e960211.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**如果应用在多MIME类型实体资源上，作为通用首部。** 

客户端使用表单提交数据，表单中提交内容包含文字、文件多种类型。

`Content-Disposition`紧跟在分隔符`--boundary`的后面，格式为`form-data; name="filedname"; filename="filename"`，如果表单值不为文件可以不加`filename`字段。
如果需要说明内容类型再添加`Content-Type`，最后为表单值或者文件内容。

如下报文所示，客户端提交表单内容，通过分隔符分隔，每个部分都包含`Content-Disposition`请求头部字段。

![表单提交报文](https://upload-images.jianshu.io/upload_images/4538003-b47fc95d88d0e423.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##### Accpet-Encoding和Content-Encoding

`Accept-Encoding`：表示客户端支持的压缩格式，属于请求首部。
`Content-Encoding`：表示客户端提交资源实体数据的压缩格式或服务端响应返回资源实体数据的压缩格式，属于实体首部。

** Encoding的目的负责将传输的数据进行压缩，对文本压缩率较高，图片、视频等多媒体文件压缩率失效。**

HTTP协议中常用的压缩格式有以下几种，可以使用通配符`*`，表示所有的压缩格式。

- gzip：由GNU zip生成，使用Lempel-Ziv coding (LZ77)压缩算法，也是互联网上最流行的压缩格式。
- compress：由UNIX文件压缩程序compress生成，使用Lempel-Ziv-Welch (LZW) 压缩算法。
- deflate：使用zlib格式和deflate压缩算法生成的压缩格式，流行程度仅次于 gzip。
- br：使用专门为HTTP优化的新压缩算法Brotli。
- identity：用于标识，不执行压缩和修改的默认压缩格式。  

`Accpet-Encoding`中的值为压缩格式和权重，两者用`;`隔开，格式为`<Encoding-Type>;q=value`。
`Accpet-Encoding`的值可以为多种压缩格式，表示客户端支持多种压缩格式。权重的含义同上。

客户端最希望服务端返回的压缩格式为`gzip`，其次为`deflate`，权重为0.9。

```http
Accept-Encoding: gzip, deflate;q=0.9。
```

`Content-Encoding`的值只能为压缩格式中的一种，实体数据只会用一种压缩格式压缩。

##### Accept-Language和Content-Language

`Accept-Language`:表示客户端支持的自然语言类型，属于请求首部。
`Content-Language`：表示客户端提交资源实体数据的自然语言类型或服务端响应返回实体数据的自然语言类型，属于实体首部。

自然语言类型一般为中文、英文等，如中文`zh-CN`、英文`en-US`。

`Accept-Language`的值为语言类型和权重，两者用`;`隔开，格式为`<Language-Type>;q=value`。
`Accept-Language`可以有多个值，表示客户端可以支持的自然语言。权重的含义同上。

`Content-Language`的值只能有一个，实体数据只会统一用一种自然语言。

##### Accept-Charset

`Accept-Charset`：客户端支持的字符集。
资源实体数据的字符集在`Content-Type`中。

字符集有`utf-8`、`iso-8859-1`等。

`Accept-Charset`的值为字符集和权重，两者用`;`隔开，格式为`<Charset>;q=value`
`Accept-Charset`可以有多个值，表示客户端可以支持的字符集。权重含义同上。

#### 文件传输

HTTP除了传输资源实体数据，还要传输文件，而且有可能是大文件。
服务器返回给客户端的文件比较大或者是由PHP、Java等动态生成的资源实体数据，服务端无法预估生成的大小。

下面从客户端和服务端两个方面分别说明设置哪些头部字段处理。

##### Transfer-Encoding

`Transfer-Encoding`：表示报文的编码格式，值可以为`compress`、`deflate`、`gzip`、`identity`、`chunked`。

使用方式主要为`Transfer-Encoding: chunked`，用于分块传输。

`Transfer-Encoding`有些资料显示属于通用头部，但是查询[MDN Transfer-Encoding](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Transfer-Encoding)，它属于响应首部，
并且请求测试，即使手动将其添加到请求头部，请求时请求头中该字段也不会出现。

**分块传输主要应用于服务端响应返回数据给客户端。**分块传输会将数据进行分块，逐个返回给客户端。
1. **服务端传输大文件**，可以一边压缩数据，一边返回数据，客户端逐个接收，而不用一次性压缩所有的数据返回给客户端，客户端也不用使用很大的byte数组来接收大文件。
2. **服务端是动态生成的资源实体数据**，一开始不知道数据的大小，可以逐块传输给客户端。
服务端不用生成完毕之后，再返回资源，会导致客户端等待时间过久。客户端逐块接收，逐块加载。

使用分块传输的编码规则如下：

1. 每个分块包含两个部分，长度头和数据块。

2. 长度头为16进制，且以`CRLF`(即为/r/n)结尾。

3. 数据块按照长度头切分，跟在数据头后面，也以`CRLF`(即为/r/n)结尾。

4. 最后一行把长度头0作为结束符，即为`0\r\n\r\n`;

![分块编码格式](https://upload-images.jianshu.io/upload_images/4538003-4db36913b8ac13af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

具体格式如下图所示，红色矩形数字即为长度头，2000为16进制，十进制为8192byte。紧跟的就是实体数据，由于太长做了截取。最后的0就是结束符。
![image.png](https://upload-images.jianshu.io/upload_images/4538003-92a27b00daca233b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

分块传输我们常见就是下载文件，浏览无法知道文件的大小，服务端分包逐个传输，浏览器逐个接收，去掉分块编码，重新组装成完整文件。
![下载文件](https://upload-images.jianshu.io/upload_images/4538003-b2eae84905ef4b2d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过Wireshark抓包，大文件被分包之后，会通过HTTP协议，将数据返回给客户端。
![HTTP分块传输](https://upload-images.jianshu.io/upload_images/4538003-5d3c7ba457f2d08d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果小文件或者已经知道资源实体数据传输大小则不需要使用分块传输，因为会导致多一次TCP传输，降低数据传输效率。

##### Range、Accept-Ranges和Content-Range

`Accept-Ranges`: 表示服务端是否支持范围请求，它属于响应首部，千万不要和上面的带`Accept`相关的字段混淆，以为它属于请求首部。有两个值：

- `none`：表示不支持范围请求，等同于服务端不返回该字段。
  
  ```http
  Accept-Ranges: none
  ```

- `bytes`：表示支持范围请求，且单位为bytes。
  
  ```http
  Accept-Ranges: bytes
  ```

`Range`: 客户端请求服务端获取资源实体数据的大小范围，属于请求首部。
值的格式为`bytes=x-y`或`bytes=x-y, x-y, x-y`。`x-y`表示起点和终点，可以是1个范围，表示单范围请求。也可以是多个范围，表示多范围请求，多个范围用`,`隔开。
使用方法如下所示，前者表示单范围请求，后者表示多范围请求。

```http
Range: bytes = 0-200
Range: bytes=-100, 200-656, 700-
```

在上面的例子中，`x-y`中起点的`x`或者终点的`y`可以省略。

- `-100`,开头可以省略`x`，表示客户端请求资源最后100字节，相当于`990-999`。

- `0-656`,表示请求的资源大小范围从文档开头到第656字节。

- `700-`尾部可以省略`y`，表示请求资源大小范围从第700字节一直到字段的最后一个字节，相当于`700-999`。

`Content-Range`: 表示服务端返回给客户端的数据在服务端资源的资源属于哪个范围，属于响应首部。值的格式为`bytes x-y/size`。`x-y`表示起点和终点，`size`表示整个资源的大小。

如果下面这个例子，表示在服务端67559大小的资源中，返回的第200字节到第1000字节范围的文件。

```http
Content-Range: bytes 200-1000/67589
```

**范围请求是客户端向服务器请求指定范围的部分资源，相当于客户端主动要求对服务端的资源进行分块。**

比如我们平时在视频网站上看视频，并不是等视频全部下载完成，才可以观看，而是边下载变播放。客户端根据进度条，请求服务端指定范围的视频文件，服务端根据客户端的请求，返回指定范围的视频文件数据范围。

另外还能用于，多段下载，比如百度云对每个账号限速100kb，我们可以开10个账号，每个账号同时现在文件的其中一部分，最后合并，那么速度就相当于 100kb * 10了。

下面分别来介绍单范围请求和多范围请求。

###### 单范围请求

客户端向服务端范围请求，如果是单个范围请求，客户端通过`Range`字段只请求其中一个范围，比如从第100字节到第200字节，步骤如下所示。

1. 客户端先向服务端`HEAD`请求，判断服务端是否支持`HEAD`请求。
   如果响应头部字段包含`Accept-Ranges: bytes`，表示服务端支持范围请求，并且通过`Content-Length`返回资源的大小值。
   如果响应头部字段不包含`Accept-Ranges`首部字段或者为`Accept-Ranges: none`，表示服务端不支持范围请求。

2. 如果服务端支持范围请求，客户端发送请求，请求头部中包含`Range`头部字段，说明资源范围。
> 请求头Range是 HTTP 范围请求的专用字段，格式是“bytes=x-y”，其中的 x 和 y 是以字节为单位的数据范围。
> 要注意 x、y 表示的是“偏移量”，范围必须从 0 计数，例如前 10 个字节表示为“0-9”，第二个 10 字节表示为“10-19”，而“0-10”实际上是前 11 个字节。
> Range 的格式也很灵活，起点 x 和终点 y 可以省略，能够很方便地表示正数或者倒数的范围。假设文件是 100 个字节，那么：
> “0-”表示从文档起点到文档终点，相当于“0-99”，即整个文件；
“10-”是从第 10 个字节开始到文档末尾，相当于“10-99”；
“-1”是文档的最后一个字节，相当于“99-99”；
“-10”是从文档末尾倒数 10 个字节，相当于“90-99”。

3. 服务端接收到之后先判断请求范围是否合法，如果超出文件范围，比如文件只有100个字节，但请求「200-300」，返回响应码`416 Range Not Satisfiable`。

4. 如果合法，返回响应码`206 Partial Content`，表示返回部分资源，响应头部包含`Content-Range`字段说明返回资源的范围。
> 告诉片段的实际偏移量和资源的总大小，格式是“bytes x-y/length”，与 Range 头区别在没有“=”，范围后多了总长度。例如，对于“0-10”的范围请求，值就是“bytes 0-10/100”。

模拟如下，先使用`HEAD`请求判断，如下图所示：

![head请求报文](https://upload-images.jianshu.io/upload_images/4538003-75304bfd36f64db1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

再使用`GET`请求单个范围的资源数据。

![单范围请求报文](https://upload-images.jianshu.io/upload_images/4538003-db283c489ffee6bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 多范围请求

多范围请求一般用的不多，但是还是讲解一下。客户端可以通过`Range`字段请求多个资源范围，比如从第100字节到第200字节，第400字节到第600字节等。步骤如下所示

1. 和单范围请求一样，客户端发送`HEAD`请求，这里不再赘述。

2. 客户端请求服务端，通过`Range`字段发送多个请求范围。

3. 服务端收到之后判断所有请求范围是否合法，不合法返回响应码`416`。
    如果合法，响应码`206`。返回所有范围的资源数据，响应头部字段的`Content-type`值为`multipart/byteranges; boundary=00000000001`，`boundary`表示数据分隔符。
    `Content-Length`值为返回数据的大小。

4. 返回数据使用分隔符`--boundary`隔开，之后要用`Content-Type`和`Content-Range`标记这段数据的类型和所在范围，然后就像普通的响应头一样以回车换行结束，再加上分段数据，最后用一个“- -boundary- -”（前后各有两个“-”）表示所有的分段结束。

5. 客户端收到之后去除分隔符，提取指定范围的资源。

![返回数据格式](https://upload-images.jianshu.io/upload_images/4538003-8be7f32fe0852656.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

具体请求报文如下所示

![多范围请求报文](https://upload-images.jianshu.io/upload_images/4538003-42c0f4f1deafc65d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 连接管理

由于HTTP开始设计时，使用短连接。客户端请求服务端，客户端和服务端通过3次握手建立TCP连接，客户端发送请求，服务端返回响应后，两者断开连接。
下次客户端再向服务端发送请求时，再重新建立连接。如果短时间内同一个客户端和服务端进行请求应答，会重复建立建立连接，传输效率低下。

为了改变这种情况，HTTP/1.1启用了长连接，长连接可以复用同一个TCP链路，，短时间同一个客户端向服务端进行请求应答，不用重复建立连接和断开连接，提高了传输效率。

![长连接和短连接](https://upload-images.jianshu.io/upload_images/4538003-9dd6f9bbeb3c1a15.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

为了解决队头堵塞的问题，相同域名下使用**并发连接**，对同个域名发起多个长连接，如果队头被堵塞，其他请求可以走另外的长连接，一般浏览器默认设置相同域名下，连接数为6-8个。

如果还不够用，就使用**域名分片**技术，将多个域名同时指向一台服务器，这样每个域名下都有长连接数名额。

##### Connection

`Connection`：表示客户端和服务端的连接管理，属于通用首部，一般有两个值。

- `keep-alive`: 请求头使用表示客户端支持长连接，响应头使用表示服务端支持长连接。使用方式`Connection: keep-alive`。

- `close`：请求头使用表示客户端这次请求之后主动要求断开TCP连接。响应头使用表示服务端这次请求之后主动要求断开TCP连接。使用方式`Connection: close`。

下面使用WireShark抓包，如果客户端和服务端请求头都没有`Connection: keep-alive`，则会被当成短连接，每次GET请求都要重新建立连接和断开连接。
当我们编写服务端代码时，不用手动添加，客户端浏览器请求头一般会带上该头部字段。

![短连接](https://upload-images.jianshu.io/upload_images/4538003-478544ec1bf84817.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果使用长连接，第一次请求建立TCP连接之后，后面的HTTP请求都复用TCP链路，直到断开TCP连接。

![长连接](https://upload-images.jianshu.io/upload_images/4538003-a22cc17445299193.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

一般情况下服务端不会主动使用`Connection: close`主动断开连接。如果客户端也不主动断开连接，会导致大量的空闲连接占用服务端的资源。
这种情况下如Tomcat、Nginx等服务器会设置最大保持连接时间、最大请求数，当超过时，服务端会主动断开连接。

每个服务器都有长连接保活措施，比如发送心跳包等。如果检测到客户端无连接，服务端会主动断开连接。
![心跳包](https://upload-images.jianshu.io/upload_images/4538003-075f23e17cd17ad7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

HTTP连接还可以通过头部字段`Keep-Alive`管理。不过这个字段的服务端和客户端的约束力并不强，很多都不会遵守，一般使用率比较低。下面简单介绍一下。

##### Keep-Alive

`Keep-Alive`: 表示客户端和服务端当前连接约定的超时时间和最大请求数。使用格式如下`Keep-Alive: parameters`，有两个值可以设置：

- `timeout`: 表示保持连接的最长时间，超过最长时间服务端主动关闭连接。

- `max_requests`: 表示连接最大发送请求的数量，如果超过请求数量，服务端主动关闭连接。

如`Keep-Alive: timeout=5, max=1000`，最长时间为5秒，最多发送的请求为2个。

#### 重定向

重定向是指当客户端请求服务端时，服务端由于当前URL下的资源失效，服务端返回给客户端一个新的URL，客户端再使用新的URL请求服务端。
重定向要适度使用，因为客户端会请求两次服务端。

##### Location

`Location`:表示服务端返回给客户端的重定向URL，属于响应首部，配合服务端返回的响应码3xx一起使用。

`Location`中的URL可以使用绝对地址或相对地址。

- 绝对地址即为完整的URL，包括包括 scheme、host:port、path 等，如`https://news.sina.com.cn/s/2019-11-29/doc-iihnzhfz2569609.shtml?cre=tianyi&mod=pchp&loc=2&r=0&rfunc=21&tj=none&tr=12`。

- 相对地址即只包括path和query部分，是不完整的。如`s/2019-11-29/doc-iihnzhfz2569609.shtml?cre=tianyi&mod=pchp&loc=2&r=0&rfunc=21&tj=none&tr=12`。

如果在站内跳转可以使用相对地址，浏览器可以从请求中的host中获取到服务端地址，拼接得到完整URL。
如果跳转到站外，必须使用绝对地址，如果使用相对地址，浏览器会拼接成站内URL，导致404，资源不存在。

`Location`还需要配合3xx响应码共同使用，才能完成重定向。

##### 响应码

重定向主要分为以下2种，服务端会返回对应的响应码，告知客户端。

- 永久重定向：响应码为**301 Permanent Redirect**，表示原来url地址下的资源永远失效了，比如更新了域名、服务器变更等，客户端以后都要请求`Location`中的url获取资源，客户端会进行相应的更新。
  比如浏览器根据响应码301会更新和失效URL相关的历史记录、书签等，防止二次跳转。

- 临时重定向：响应码为**302 Found**，表示原来URL下的资源暂时失效，系统处于临时维护的状态。客户端只是暂时使用`Location`中的URL获取资源。
  客户端会认为原来的URL仍然有效，但暂时不可用，所以只会执行简单的跳转页面，不记录新的 URI，也不会有其他的多余动作，下次访问还是用原URL。

还有以下3种响应码，对上面两种响应码的拓展，不过这三个状态码的接受程度较低，有的浏览器和服务器可能不支持，开发时应当慎重，测试确认浏览器的实际效果后才能使用。

- 303 See Also：属于临时重定向，但是使用新的URL访问服务端只能用GET请求，不能使用POST、PUT等其他请求方式。

- 307 Temporary Redirect：属于临时重定向，但是使用新的URL访问服务端的请求方式，必须和之前使用旧的URL访问一样。

- 308 Permanent Redirect：属于永久重定向，但是使用的新的URL访问服务端的请求方式必须，必须和之前使用旧的URL访问一样。这点和307一致。

有很多网站都会使用重定向技术，下面是访问京东的一个网址，返回了响应码302，响应头包含`Location`提供了新的URL。

![重定向报文](https://upload-images.jianshu.io/upload_images/4538003-0812a56f1c512a20.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

浏览器使用`Location`中新的URL再次请求服务端。
![请求新的URL](https://upload-images.jianshu.io/upload_images/4538003-03ffb2887722c459.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### Cookie

#### 代理服务

#### HTTP缓存

虽然目前传输速度和可靠性基本可以得到保证，但是还是会遇到各种传输问题，传输时延不确定，以及需要的传输成本。
因此要使用缓存将获取的数据重复利用，将数据保存在本地磁盘中，减少请求-响应次数，增加响应速度。

HTTP是客户端请求，服务端响应的模式。按照缓存端点，分为**客户端缓存**和**服务端缓存**。

**客户端缓存将资源保存在本地磁盘中。
服务端缓存主要通过代理服务器实现(缓存代理)，当然服务器内部也有如 Memcache、Redis、Varnish 等缓存工具，HTTP无法涉及到，比如服务端不会根据http请求头部字段去刷新redis数据。**

客户端和服务端就相当于茅台零售商和茅台总公司的关系，缓存代理服务器相当于每个地区的代理商。
零售商会根据客户的需求多进一批货，客户下次再要就不用再跑去贵州了。零售商仓库的茅台酒就是客户端缓存。零售商有库存就不需要去贵州进货。
零售商没库存了，就需要找茅台总公司下订单，去贵州进货，茅台根据订单配货给零售商。订单就相当于HTTP请求报文。
茅台总公司仓库的酒，是生产的所有茅台，并没有根据零售商的订单进行分类备货，相当于内部缓存。
全国一共有几万个茅台零售商，茅台总公司无法保存所有订单，所以每次都需要根据订单重新配货，效率大大降低。
零售商每次都要去贵州进货，运输成本也很高。
为了解决问题，茅台总公司就在每个地区设置一个代理商，代表茅台总公司处理每个地区零售商的订单。
零售商向代理商下过一次订单之后，代理商就知道了零售商需要的茅台酒品类之后，下次先备齐零售商的货，这就是服务器缓存。
以后零售商没库存就找代理商下单，代理商有库存就发货给零售商，没库存可再把订单发给茅台总公司要货。
零售商也不用每次跑到贵州进货了，直接找当地代理商就好了，大大提高了效率。

![缓存流程](https://upload-images.jianshu.io/upload_images/4538003-c43fd9381a3c67f2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 缓存控制

仅仅有缓存还不够，还需要通过HTTP协议控制服务端缓存和客户端缓存。
服务端和客户端均能通过头部字段控制客户端缓存资源，下面分别介绍服务端和客户端的缓存控制。

#### 客户端缓存控制

客户端在请求头中使用头部字段控制缓存的使用，比如缓存要符合指定时间才使用，使用缓存的条件，相当于零售商只卖在保质期1年内的茅台。
服务器在响应头中使用缓存字段控制缓存的存储，比如存储时间，是否缓存资源等等，相当于茅台公司控制茅台酒的保值期为3年，限制零售商、代理商是否能多进货有多余库存。
下面一一介绍缓存控制的相关头部字段。

##### Cache-Control

`Cache-Control`: 表示缓存控制的方式，实现缓存机制，属于通用首部，可以在请求头和响应头中使用。

###### 请求头中使用

请求头中的用法，表示客户端中从缓存时间和缓存模式两个方面管理缓存。
前三个表示缓存时间，后4个表示缓存模式。

```http
Cache-Control: max-age=<seconds>
Cache-Control: max-stale[=<seconds>]
Cache-Control: min-fresh=<seconds>
Cache-control: no-cache
Cache-control: no-store
Cache-control: no-transform
Cache-control: only-if-cached
```

3个缓存时间相关的值：

- `max-age`:表示缓存的过期时间，超过过期时间，资源过期，客户端会向服务器请求，可以理解成商品的保质期。
比如`max-age=20`，表示保质期是20秒，如果资源缓存超过了20秒，视为过期，重新请求新资源。

- `max-stale`: 表示最大过期时间，缓存资源如果过期，但是在该时间范围内还可以被使用。
  比如`max-stable = 5，max-age=20`,如果资源在本地保存了23秒超过20秒保质期，已经过期了。过期时间为3秒，在5秒以内，资源还可以被使用。
  就好比罐头虽然过期了3天，但是过期时间还不久，兜里没钱也能凑活吃。

- `min-fresh`: 表示最小新鲜时间，表示缓存资源几秒之后还没有过期，缓存要满足`min-fresh+资源本地保存时间<保质期`才可以被使用。比如还有`max-age=20 min-fresh=2`，资源已经在本地保存了19秒，还有1秒就过期了，但是`min-fresh`要求资源2秒之后还是新鲜的，2+19=21>20，所以客户端缓存不能使用了。

4个缓存模式相关的值：

- `no-store`: 不使用缓存，每次向服务器请求最新资源，相当于零售商不使用库存的茅台，每次都向茅台总公司进货。
- `no-cache`:并不是不使用缓存，而是使用缓存之前，先询问服务器，一般配合条件缓存头部后面介绍，好比零售商要用库存的茅台，都要打电话给茅台总公司，询问是否有最新生产的茅台。
- `no-transform`: 不得对缓存资源进行转换或转变。
- `only-if-cached`：客户端只用已缓存的资源，无论如何都不会向服务器发获取新资源。

###### 响应头中使用

如果没有缓存代理服务器，服务器在响应头使用`Cache-control`的以下值。

`must-revalidate`：一旦资源过期（比如已经超过max-age），再向服务器认证之前，缓存不能使用。
`no-store`:客户端不使用任何缓存。
`no-cache`：客户端可以使用缓存资源，但是使用缓存之前必须要向服务器验证是否过期。
`max-age`：客户端缓存资源的是时间，相当于缓存保质期。

```http
Cache-control: must-revalidate
Cache-control: no-cache
Cache-control: no-store
Cache-Control: max-age=<seconds>
```

###### Expires

`Expires`: 响应首部，表示资源过期时间，比如`Expires: Wed, 21 Oct 2019 08:00:00 GMT`，比如资源在2019年10月21号，早上8点过期。

但是如果服务器响应头中使用`Cache-control:max-age`指定了过期时间，会覆盖`Expires`中的时间。

#### 条件控制

`cache-control`可以控制是否使用本地缓存，如果缓存可以使用，一般为了保证缓存的准确性，客户端会请求服务端验证缓存是否符合条件可以被使用。
客户端请求头就需要用到HTTP协议中的条件请求字段，一共有5个，`If-Modified-Since`和`If-Unmodified-Since`、`If-Match`和`If-None-Match`、`If-Ranage`。

**最常被使用的是If-Modified-Since和If-None-Match**。

##### If-Modified-Since、If-Unmodified-Since和Last-modified

`Last-modified`:响应首部，表示服务器告诉客户端返回资源被修改的日期，如`Wed, 21 Oct 2015 07:28:00 GMT`。
`If-Modified-Since`:请求首部，用于`GET`和`HEAD`请求，向服务器验证资源在指定日期后是否改变，如果改变返回状态码200，并且带上最新资源。如果未改变返回状态码304(Not Modified)，并且不返回资源，客户端使用缓存。
`If-Unmodified-Since`：请求首部，和`If-modified-Since`意思相反，向服务器验证资源在指定日期后是否未改变，如果改变返回状态码412(Precondition Failed)错误，如果未改变返回状态码200，带上最新的资源。

客户端第一次向服务器请求获取资源不会使用条件请求，服务端返回资源，响应头会带上`Last-modified`，客户端会保存`Last-modified`的值，即修改日期。
当客户端再向服务端请求资源时，`If-Modified-Since`或`If-Unmodified-Since`会带上修改日期，让服务器进行验证。

`Last-modified`相当于零售商第一次进货，茅台总公司告诉零售商茅台的生产日期2019-12-18 15:00:00。
`If-Modified-Since`相当于零售商打电话问茅台总公司，12月18号15点之后有新茅台吗，茅台总公司如果生产新的会提供给零售商，如果没有让他用库存里面的。
`If-Unmodified-Since`相当于零售商打电话问茅台总公司，12与18号15点之后没有新茅台吗，茅台总公司如果生产新的会让零售商加价，如果没有继续供货。

##### If-Match、If-None-Match和ETag

`ETag`:响应首部，标示实体标签，是服务器告知客户端资源的唯一标识，主要是用来解决修改时间无法准确区分文件变化的问题。比如，一个文件在一秒内修改了多次，但因为修改时间是秒级，所以这一秒内的新版本无法区分。

`If-Match`:请求首部，用于`GET`和`HEAD`请求，向服务器验证本地资源实体标签和服务端的是否一致，如果一致返回状态码返回状态码200，并且带上最新资源，如果不一致返回状态码412（Precondition Failed）先决条件失败。

`If-None-Match`:请求首部，向服务器验证本地资源标签和服务端是否不一致，如果一致返回状态码304(Not Modified)，如果不一致返回状态码200并且带上服务端最新资源。

和上面一样，客户端`If-Match`或`If-None-Match`会带上服务器响应报文中的`Etag`头部字段的值向服务器验证是否一致。

`Etag`有强弱之分，强`ETag`使用强比较算法，只有在每一个字节都相同的情况下，才可以认为两个资源是相同的，如`"bfc13a64729c4290ef5b2c2730249c88ca92d82d"`。
弱`ETag`在值前有个“W/”标记，只要求资源在语义上没有变化，但内部可能会有部分发生了改变（例如 HTML 里的标签顺序调整，或者多了几个空格，如`W/"67ab43"`。

##### If-Range

`If-Range`:请求首部，服务器若指定的If-Range字段值和请求资源的ETag值一致时，则作为范围请求处理，否则返回全部资源。

##### 疑问

使用了客户端缓存还要向服务器验证，效率不是还没提高吗？
客户端向服务器发送请求，如果验证客户端缓存可用，就不用返回资源数据了，如果不可用再返回资源数据。
就好比零售商打电话给茅台公司询问现在库存的茅台符合你们要求吗，不符合我再去贵州提货，如果符合就卖库存里面的，省的我跑一趟了。

#### 服务端缓存控制

服务端缓存控制会使用缓存代理服务器。
客户端请求头中使用的缓存头部，不光控制客户端的缓存，也是要求代理服务器中的缓存资源，代理服务器要根据客户端的请求头部字段返回符合要求的资源。
源服务器不光要控制客户端的缓存，也要控制代理服务器的缓存。

代理服务器对于客户端来说是服务端，对于源服务器来说则是客户端，相当于资源的中转站，既不生产资源，也不消费资源。
就如同代理商，将茅台总公司的茅台酒转发给零售商，如果无法满足零售商的需求，就告知服茅台总公司生产对应的茅台。

缓存代理服务器可以在请求头和响应头中使用头部字段控制缓存，客户端和源服务器也要对代理服务器进行缓存控制，`Cache-Control`的语义性也会发生变化。

![代理模式](https://upload-images.jianshu.io/upload_images/4538003-852b34e48f560432.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 源服务器

源服务器中的`Cache-Control`针对代理服务器和客户端的约束要进行区分。
其中`max-age`、`no_store`、`no_cache`和`must-revalidate`不光能约束客户端，也能约束代理服务器。

但是客户端缓存和代理服务器缓存是不同的，如果源服务器需要对客户端和代理服务器的缓存控制进行区分，源服务器需要使用`Cache-control`中新的值对代理服务器进行控制。

`no-transform`：代理服务器不能对资源进行转换或转变，包括对报文的修改，比如`Content-Encoding`、`Content-Range`、`Content-Type`等HTTP头不能由代理修改。
`public`:表示资源可以在任何地方被缓存，包括客户端和代理服务器。
`private`:表示资源只能在客户端缓存，其他任何地方都不能缓存。
`proxy-revalidate`:要求代理服务器的缓存过期后必须验证，不约束客户端。
`s-maxage=<seconds>`:要求代理服务器的缓存过期时间，不约束客户端。

```http
Cache-control: no-transform
Cache-control: public
Cache-control: private
Cache-control: proxy-revalidate
Cache-control: s-maxage=<seconds>
```

源服务器使用`Cache-control`中的字段，并组合使用相关的值，对客户端和服务端的缓存进行控制。
比如`private, max-age=5`，代理服务器不能缓存资源，客户端缓存资源的过期时间为5秒。
`max-age=30, proxy-revalidate, no-transform`，客户端缓存资源的时间为30秒，代理服务器缓存过期之后必须向服务器请求新的资源，且不能改变资源。

下面流程图完整展示了源服务器对客户端和服务器缓存策略的控制。

![源服务器缓存控制](https://upload-images.jianshu.io/upload_images/4538003-2238f93a1a1f73b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 客户端

客户端使用`Cache-control`的值，不光控制客户端本地缓存的使用，也要控制代理服务器缓存的使用。
`max-age`、`max-stale`、`min-fresh`要求对代理服务器缓存时间的要求，用法上面控制客户端缓存一样。
`no-transform`：代理服务器不能改变资源
`only-if-cached`：表示只接受代理缓存的数据，不接受源服务器的响应。如果代理上没有缓存或者缓存过期，就应该给客户端返回一个504（Gateway Timeout）。
`no-store`：代理服务器不使用任何缓存，直接请求源服务器获取新资源。
`no-cache`：代理服务器使用缓存之前，先向源服务器进行验证。

```http
Cache-Control: max-age=<seconds>
Cache-Control: max-stale[=<seconds>]
Cache-Control: min-fresh=<seconds>
Cache-control: no-cache
Cache-control: no-store
Cache-control: no-transform
Cache-control: only-if-cached
```
