---
title: URI、URL、URN 介绍
date: 2023-08-27 15:39:54
categories:
- 计算机网络
---

<meta name="referrer" content="no-referrer" />

# URI、URL、URN 介绍

URI：统一资源标识符
URL：统一资源定位符
URN：统一资源名

三者之间关系如下所示：

URI包含了URL和URN，因为两者都可以作为资源标识符。
URI 由 3 部分组成，以下面 URI 为例
`https://feishu.cn/drive/home/`
- 协议名: 访问该资源应当使用的协议，使用 https。
- 主机名: 互联网上主机的标记，可以是域名或 IP 地址，使用 feishu.cn。
- 路径: 资源在主机上的位置，使用“/”分隔多级目录,使用 /drive/home。

URL类似于地址，用于定位资源，如`https://leetcode-cn.com`，我们可以通过访问URL找到资源，就比如通过北京市朝阳区东新小区2幢1单元2室家庭地址，定位到某人的具体位置。

URN类似于身份证号码，通过特定命名空间的名字标识资源，如`bitpoetry.io/posts/hello.html`。

URL一定是URI，但是URI不一定是URL。无论URN还是URL都是URI其中的一类。

开发过程中，所以不用分的太细，一律当成 URI 统一资源标识符看即可。

![URI、URL和URN之间的关系](https://upload-images.jianshu.io/upload_images/4538003-60d69ee010eb7e84.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## DNS 和 URI 之间的关系 
DNS 负责将 URI 中的域名部分解析成 IP 地址，定位到服务器，再根据 URI 中的路径名找到服务器中的资源。
举个例子，你要买农夫山泉矿泉水，DNS 根据超市名，帮助你找到超市的位置，根据 URI 知道农夫山泉在超市的摆放位置，然后找到它。