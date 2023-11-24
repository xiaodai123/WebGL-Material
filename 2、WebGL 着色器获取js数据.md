---
created: 2023-11-24T14:25:21 (UTC +08:00)
tags: [WebGL]
source: https://juejin.cn/post/7067368813838204959
author: 李伟_Li慢慢
---

# WebGL 着色器获取js数据

---
源码：[github.com/buglas/webg…](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fbuglas%2Fwebgl-lesson "https://github.com/buglas/webgl-lesson")

### 1-attribute 变量的概念。

回顾一下我们上一篇中点的定位：

```
gl_Position = vec4(0,0,0,1);
```

这是一种将数据写死了的硬编码，缺乏可扩展性。

我们要让这个点位可以动态改变，那就得把它变成attribute变量。

attribute 变量是只有顶点着色器才能使用它的。

js 可以通过attribute 变量向顶点着色器传递与顶点相关的数据。

### 2-js向attribute 变量传参的步骤

1.  在顶点着色器中声明attribute 变量。

```
<script id="vertexShader" type="x-shader/x-vertex">
    attribute vec4 a_Position;
    void main(){
        gl_Position = a_Position;
        gl_PointSize = 50.0;
    }
</script>
```

2.  在js中获取attribute 变量

```
const a_Position=gl.getAttribLocation(gl.program,'a_Position');
```

3.  修改attribute 变量

```
gl.vertexAttrib3f(a_Position,0.0,0.5,0.0);
```

整体代码

```
<canvas id="canvas"></canvas>
<script id="vertexShader" type="x-shader/x-vertex">
    attribute vec4 a_Position;
    void main(){
        gl_Position = a_Position;
        gl_PointSize = 50.0;
    }
</script>
<script id="fragmentShader" type="x-shader/x-fragment">
    void main() {
        gl_FragColor = vec4(1.0, 1.0, 0.0, 1.0);
    }
</script>
<script type="module">
    import {initShaders} from '../jsm/Utils.js';

    const canvas = document.getElementById('canvas');
    canvas.width=window.innerWidth;
    canvas.height=window.innerHeight;
    const gl = canvas.getContext('webgl');
    const vsSource = document.getElementById('vertexShader').innerText;
    const fsSource = document.getElementById('fragmentShader').innerText;
    initShaders(gl, vsSource, fsSource);
    const a_Position=gl.getAttribLocation(gl.program,'a_Position');
    gl.vertexAttrib3f(a_Position,0.0,0.0,0.0);
    gl.clearColor(0.0, 0.0, 0.0, 1.0);
    gl.clear(gl.COLOR_BUFFER_BIT);
    gl.drawArrays(gl.POINTS, 0, 1);
</script>
```

接下来详细解释一下。

### 3-js向attribute 变量传参的原理

#### 3-1-着色器中的attribute 变量

```
attribute vec4 a_Position;
void main(){
    gl_Position = a_Position;
    gl_PointSize = 50.0;
}
```

-   attribute 是存储限定符，是专门用于向外部导出与点位相关的对象的，这类似于es6模板语法中export 。
-   vec4 是变量类型，vec4是4维矢量对象。
-   a\_Position 是变量名，之后在js中会根据这个变量名导入变量。这个变量名是一个指针，指向实际数据的存储位置。也是说，我们如果在着色器外部改变了a\_Position所指向的实际数据，那么在着色器中a\_Position 所对应的数据也会修改。

接下来，咱们说一下在js 里如何获取attribute 变量。

#### 3-2-在js中获取attribute 变量

我们在js 里不能直接写a\_Position 来获取着色器中的变量。

因为着色器和js 是两个不同的语种，着色器无法通过window.a\_Position 原理向全局暴露变量。

那我们要在js 里获取着色器暴露的变量，就需要找人来翻译，这个人就是程序对象。

```
const a_Position=gl.getAttribLocation(gl.program,'a_Position');
```

-   gl 是webgl 的上下文对象。
    
-   gl.getAttribLocation() 是获取着色器中attribute 变量的方法。
    
-   getAttribLocation() 方法的参数中：
    
    -   gl.program 是初始化着色器时，在上下文对象上挂载的程序对象。
    -   'a\_Position' 是着色器暴露出的变量名。

这个过程翻译过来就是：gl 上下文对象对program 程序对象说，你去顶点着色器里找一个名叫'a\_Position' 的attribute变量。

现在a\_Position变量有了，接下来就可以对它赋值了。

#### 3-3-在js中修改attribute 变量

attribute 变量即使在js中获取了，他也是一个只会说GLSL ES语言的人，他不认识js 语言，所以我们不能用js 的语法来修改attribute 变量的值：

```
a_Position.a=1.0
```

我们得用特定的方法改变a\_Position的值：

```
gl.vertexAttrib3f(a_Position,0.0,0.5,0.0);
```

-   gl.vertexAttrib3f() 是改变变量值的方法。
    
-   gl.vertexAttrib3f() 方法的参数中：
    
    -   a\_Position 就是咱们之前获取的着色器变量。
    -   后面的3个参数是顶点的x、y、z位置

a\_Position被修改后，我们就可以使用上下文对象绘制最新的点位了。

```
gl.clearColor(0.0, 0.0, 0.0, 1.0);
gl.clear(gl.COLOR_BUFFER_BIT);
gl.drawArrays(gl.POINTS, 0, 1);
```

### 4-扩展

#### 4-1-vertexAttrib3f()的同族函数

gl.vertexAttrib3f(location,v0,v1,v2) 方法是一系列修改着色器中的attribute 变量的方法之一，它还有许多同族方法，如：

```
gl.vertexAttrib1f(location,v0) 
gl.vertexAttrib2f(location,v0,v1)
gl.vertexAttrib3f(location,v0,v1,v2)
gl.vertexAttrib4f(location,v0,v1,v2,v3)
```

它们都可以改变attribute 变量的前n 个值。

比如 vertexAttrib1f() 方法自定一个矢量对象的v0值，v1、v2 则默认为0.0，v3默认为1.0，其数值类型为float 浮点型。

#### 4-2-webgl 函数的命名规律

GLSL ES里函数的命名结构是：<基础函数名><参数个数><参数类型>

以vertexAttrib3f(location,v0,v1,v2,v3) 为例：

-   vertexAttrib：基础函数名
-   3：参数个数，这里的参数个数是要传给变量的参数个数，而不是当前函数的参数个数
-   f：参数类型，f 代表float 浮点类型，除此之外还有i 代表整型，v代表数字……
