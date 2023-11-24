---
created: 2023-11-24T14:27:21 (UTC +08:00)
tags: [WebGL]
source: https://juejin.cn/post/7067369603235577893
author: 李伟_Li慢慢
---

# WebGL 用js 控制顶点的颜色

---
源码：[github.com/buglas/webg…](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fbuglas%2Fwebgl-lesson "https://github.com/buglas/webgl-lesson")

首先我们要知道，限定颜色变量的限定符叫uniform。

uniform 翻译过来是一致、统一的意思。

接下来咱们说一下用js 控制顶点颜色的步骤。

![1.gif](assets/c699d1d6b79748f1a6ad988f998be105tplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

### 1-用js 控制顶点颜色的步骤

1.在片元着色器里把控制顶点颜色的变量暴露出来。

```
<script id="fragmentShader" type="x-shader/x-fragment">
    precision mediump float;
    uniform vec4 u_FragColor;
    void main() {
        gl_FragColor = u_FragColor;
    }
</script>
```

上面的uniform 就是咱们刚才说过的限定符，vec4 是4维的变量类型，u\_FragColor 就是变量名。

这里还要注意一下，第一行的precision mediump float 是对浮点数精度的定义，mediump 是中等精度的意思，这个必须要有，不然画不出东西来。

2.在js 中获取片元着色器暴露出的uniform 变量

```
const u_FragColor=gl.getUniformLocation(gl.program,'u_FragColor');
```

上面的getUniformLocation() 方法就是用于获取片元着色器暴露出的uniform 变量的，其第一个参数是程序对象，第二个参数是变量名。这里的参数结构和获取attribute 变量的getAttributeLocation() 方法是一样的。

3.修改uniform 变量

```
gl.uniform4f(u_FragColor,1.0,1.0,0.0,1.0);
```

用js 控制顶点的颜色的基本步骤就是这样，其整体思路和控制顶点点位是一样的。

下面咱们一起对比着看一下整体代码。

```
<script id="vertexShader" type="x-shader/x-vertex">
    attribute vec4 a_Position;
    attribute float a_PointSize;
    void main(){
        gl_Position = a_Position;
        gl_PointSize = a_PointSize;
    }
</script>
<script id="fragmentShader" type="x-shader/x-fragment">
    precision mediump float;
    uniform vec4 u_FragColor;
    void main() {
        gl_FragColor = u_FragColor;
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
    const a_PointSize=gl.getAttribLocation(gl.program,'a_PointSize');
    const u_FragColor=gl.getUniformLocation(gl.program,'u_FragColor');
    gl.vertexAttrib3f(a_Position,0.0,0.0,0.0);
    gl.vertexAttrib1f(a_PointSize,100.0);
    gl.uniform4f(u_FragColor,1.0,1.0,0.0,1.0);
    gl.clearColor(0.0, 0.0, 0.0, 1.0);
    gl.clear(gl.COLOR_BUFFER_BIT);
    gl.drawArrays(gl.POINTS, 0, 1);
</script>
```

知道了这个原理后，我们就可以用鼠标随机改变顶点的颜色。

```
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
    const a_PointSize=gl.getAttribLocation(gl.program,'a_PointSize');
    const u_FragColor=gl.getUniformLocation(gl.program,'u_FragColor');
    gl.clearColor(0.0, 0.0, 0.0, 1.0);
    gl.clear(gl.COLOR_BUFFER_BIT);

    const arr=[];
    canvas.addEventListener('click',function(event){
        const {clientX,clientY}=event;
        const {left,top,width,height}=canvas.getBoundingClientRect();
        const [cssX,cssY]=[
            clientX-left,
            clientY-top
        ];
        const [halfWidth,halfHeight]=[width/2,height/2];
        const [xBaseCenter,yBaseCenter]=[cssX-halfWidth,cssY-halfHeight];
        const yBaseCenterTop=-yBaseCenter;
        const [x,y]=[xBaseCenter/halfWidth,yBaseCenterTop/halfHeight];
        const color=new Float32Array([
            Math.random(),
            Math.random(),
            Math.random(),
            1.0
        ])
        arr.push({x,y,z:Math.random()*50,color});
        gl.clear(gl.COLOR_BUFFER_BIT);
        arr.forEach(({x,y,z,color})=>{
            gl.vertexAttrib2f(a_Position,x,y);
            gl.vertexAttrib1f(a_PointSize,z);
            gl.uniform4fv(u_FragColor,color);
            gl.drawArrays(gl.POINTS, 0, 1);
        })
    })
</script>
```

在上面的代码中，我们使用uniform4fv() 修改的顶点颜色，类似的代码结构咱们之前提到过，在这里再给大家详细介绍一下。

### 2-uniform4fv() 方法

我们在改变uniform 变量的时候，既可以用uniform4f() 方法一个个的写参数，也可以用uniform4fv() 方法传递类型数组。

-   uniform4f 中，4 是有4个数据，f 是float 浮点类型，在我们上面的例子里就是r、g、b、a 这四个颜色数据。
-   uniform4fv 中，4f 的意思和上面一样，v 是vector 矢量的意思，这在数学里就是向量的意思。由之前的4f 可知，这个向量由4个浮点类型的分量构成。

在上面呢的案例中，我们可以知道，在修改uniform变量的时候，这两种写法是一样的：

```
gl.uniform4f(u_FragColor,1.0,1.0,0.0,1.0);
//等同于
const color=new Float32Array([1.0,1.0,0.0,1.0]);
gl.uniform4fv(u_FragColor,color);
```

uniform4f() 和uniform4fv() 也有着自己的同族方法，其中的4 可以变成1|2|3。

uniform4fv() 方法的第二个参数必须是Float32Array 数组，不要使用普通的Array 对象。

Float32Array 是一种32 位的浮点型数组，它在浏览器中的运行效率要比普通的Array 高很多
