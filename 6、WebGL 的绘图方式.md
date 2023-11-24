---
created: 2023-11-24T14:29:32 (UTC +08:00)
tags: [WebGL]
source: https://juejin.cn/post/7067372047654977572
author: 李伟_Li慢慢
---

# WebGL 的绘图方式


源码：[github.com/buglas/webg…](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fbuglas%2Fwebgl-lesson "https://github.com/buglas/webgl-lesson")

我们先看一下webgl是怎么画图的。

1.绘制多点

![image-20200922151301533](assets/bf82119bc49c4dc689a8d79b2a2e5075tplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

2.如果是线，就连点成线

![image-20200922153449307](assets/e4b7313a44e34ec88b6b37188b27e76ftplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

3.如果是面，那就在图形内部，逐片元填色

![image-20200922153643189](assets/2398da79cb644eeb93039338db796696tplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

webgl 的绘图方式就这么简单，接下咱们就说一下这个绘图方式在程序中是如何实现的。

