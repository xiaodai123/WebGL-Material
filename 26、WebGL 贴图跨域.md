---
created: 2023-11-24T14:39:32 (UTC +08:00)
tags: [WebGL]
source: https://juejin.cn/post/7067384672954613796
author: 李伟_Li慢慢
---

# WebGL 贴图跨域

---
源码：[github.com/buglas/webg…](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fbuglas%2Fwebgl-lesson "https://github.com/buglas/webgl-lesson")

我这里要说的跨域图像问题，并不单是webgl 问题，也是canvas 2d 里的问题。

为了更好的说清楚这个问题的来龙去脉，我这里就先从最简单的img标签说起。

### 1-img标签与跨域图像

我们用img 标签显示跨域图像的时候，我们只能将图像展示给用户，但并不能获取图像中的数据。

```
<img src="https://img.kaikeba.com/70350130700202jusm.png'"> 
```

我们不能通过img标签获取图像数据，是浏览器出于安全考虑的，因为有的图像里可能会含有验证码、签名、或者其它不可告人的东东，这种数据是不能随意被别人获取的。

这时候，我们可能会想到通过canvas 2d获取图像数据。

### 2-获取图像数据

若图像是同域的，下面的方法是没问题的。

```
const canvas=document.getElementById('canvas');
const ctx=canvas.getContext('2d');

const img=new Image();
img.src='https://img.kaikeba.com/70350130700202jusm.png';

img.onload=function(){
    const {width,height}=canvas
    ctx.drawImage(img,0,0)
    ctx.getImageData(0,0,width,height)
}
```

若图像是跨域的，那就会报错：

```
Failed to execute 'getImageData' on 'CanvasRenderingContext2D': The canvas has been tainted by cross-origin data.
```

其实不仅canvas 2d 是这样的，我们用webgl 获取跨域图像数据时，也会报错。

```
const image = new Image()
image.src = 'http://img.yxyy.name/erha.jpg'
image.onload = function () {
    gl.texImage2D(
        gl.TEXTURE_2D,
        0,
        gl.RGB,
        gl.RGB,
        gl.UNSIGNED_BYTE,
        image
    )

    gl.texParameteri(
        gl.TEXTURE_2D,
        gl.TEXTURE_MIN_FILTER,
        gl.LINEAR
    )

    const u_Sampler = gl.getUniformLocation(gl.program, 'u_Sampler')
    gl.uniform1i(u_Sampler, 0)

    render()
}
```

错误信息：

```
Failed to execute 'texImage2D' on 'WebGLRenderingContext': The image element contains cross-origin data, and may not be loaded.
```

由错误信息可知，代码跑到texImage2D()方法时卡住了，因为我们在这里面引用跨域图像了。

接下来我们可以使用CORS 跨域解决此问题。

### 3-CORS 跨域

CORS 是Cross Origin Resource Sharing的简写，译作跨域资源共享。

CORS 是一种网页向图像服务器请求使用图像许可的方式。

CORS 的实现过程如下：

1.先让服务器向其它域名放权，比如 Apache 服务器需要这样设置：

```
Header set Access-Control-Allow-Origin "*"
```

上面代码的意思就是允许其它域名的网页跨域获取自己服务端的资源。

上面的\* 是允许所有域名获取其服务端资源。

我们也可以写具体的域名，从而允许特定域名获取其服务端资源。

2.在网页里，通过特定的方法告诉服务端，我需要跨域获取你的资源。

其代码如下：

```
const image = new Image()
image.src = 'http://img.yxyy.name/erha.jpg'
image.setAttribute("crossOrigin", 'Anonymous')
```

有了上面crossOrigin 属性的设置，无论是在canvas 2d里使用此图像源，还是在webgl里使用此图像源，都不会报错。

crossOrigin 接收的值有三种：

-   undefined 是默认值，表示不需要向其它域名的服务器请求资源共享
-   anonymous 向其它域名的服务器请求资源共享，但不需要向其发送任何信息
-   use-credentials 向其它域名的服务器请求资源共享，并发送 cookies 等信息，服务器会通过这些信息决定是否授权
