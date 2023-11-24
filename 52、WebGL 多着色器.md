---
created: 2023-11-24T14:50:51 (UTC +08:00)
tags: [WebGL]
source: https://juejin.cn/post/7067401878312583182
author: 李伟_Li慢慢
---

# WebGL 多着色器

---
源码：[github.com/buglas/webg…](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fbuglas%2Fwebgl-lesson "https://github.com/buglas/webgl-lesson")

到目前为止，我们都是用了一套着色器做的例子，这一套着色器包含了顶点着色器和片元着色器。

在实际开发中，我们是不可能只用一套着色器做项目的，就比如场景里有两个三角形，一个三角形需要着纯色，一个三角形需要着纹理。

![image-20210916143835616](assets/42eae97fb2ef4418b6469eed5ba77ebbtplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

接下来我们就拿这两个三角形来说一下多着色器的实现方法。

### 1-多着色器绘图

#### 1-1-准备两套着色器

两套着色器，一套着纯色，一套着纹理。

```
<!-- 着纯色 -->
<script id="solidVertexShader" type="x-shader/x-vertex">
    attribute vec4 a_Position;
    void main(){
      gl_Position = a_Position;
    }
</script>
<script id="solidFragmentShader" type="x-shader/x-fragment">
    precision mediump float;
    void main(){
      gl_FragColor=vec4(0.9,0.7,0.4,1);
    }
</script>

<!-- 着纹理 -->
<script id="textureVertexShader" type="x-shader/x-vertex">
    attribute vec4 a_Position;
    attribute vec2 a_Pin;
    varying vec2 v_Pin;
    void main(){
      gl_Position = a_Position;
      v_Pin=a_Pin;
    }
</script>
<script id="textureFragmentShader" type="x-shader/x-fragment">
    precision mediump float;
    uniform sampler2D u_Sampler;
    varying vec2 v_Pin;
    void main(){
      gl_FragColor=texture2D(u_Sampler,v_Pin);
    }
</script>
```

#### 1-2-绘制纯色三角形

1.建立程序对象，并应用

```
const solidVsSource = document.getElementById('solidVertexShader').innerText
const solidFsSource = document.getElementById('solidFragmentShader').innerText
const solidProgram = createProgram(gl, solidVsSource, solidFsSource)
gl.useProgram(solidProgram)
```

上面的createProgram() 方法是基于一套着色器建立程序对象的方法：

```
function createProgram(gl, vsSource, fsSource) {
  //创建程序对象
  const program = gl.createProgram();
  //建立着色对象
  const vertexShader = loadShader(gl, gl.VERTEX_SHADER, vsSource);
  const fragmentShader = loadShader(gl, gl.FRAGMENT_SHADER, fsSource);
  //把顶点着色对象装进程序对象中
  gl.attachShader(program, vertexShader);
  //把片元着色对象装进程序对象中
  gl.attachShader(program, fragmentShader);
  //连接webgl上下文对象和程序对象
  gl.linkProgram(program);
  return program;
}
```

关于程序对象的概念咱们之前在webgl 入门里已经说过。

之前我们的初始化着色器方法initShaders() 只是比上面的方法多了一个应用程序对象的步骤：

```
gl.useProgram(program);
```

我们这里之所以把启用程序的步骤提取出来，是因为一套着色器对应着一个程序对象。

而一个webgl 上下文对象，是可以依次应用多个程序对象的。

通过不同程序对象，可以绘制不同材质的图形。

2.用当前的程序对象绘制图形

```
const solidVertices = new Float32Array([
    -0.5, 0.5,
    -0.5, -0.5,
    0.5, -0.5,
])
const solidVertexBuffer = gl.createBuffer()
gl.bindBuffer(gl.ARRAY_BUFFER, solidVertexBuffer)
gl.bufferData(gl.ARRAY_BUFFER, solidVertices, gl.STATIC_DRAW)
const solidPosition = gl.getAttribLocation(solidProgram, 'a_Position')
gl.vertexAttribPointer(solidPosition, 2, gl.FLOAT, false, 0, 0)
gl.enableVertexAttribArray(solidPosition)

gl.clear(gl.COLOR_BUFFER_BIT)
gl.drawArrays(gl.TRIANGLES, 0, 3)
```

因为我们后面还需要绘制一个纹理图形，纹理图形是需要等纹理加载成功才能绘制的。

这会造成两个三角形的异步绘制，这不是我想要的，因为一异步，就会把之前三角形的缓冲数据给清理掉，这个原理，我们之前说过。

因此，我需要同步绘制两个三角形。

我先把上面的绘图方法封装到一个函数里，等纹理三角形的图片加载成功了，再执行。

```
function drawSolid() {
    /* 1.建立程序对象，并应用 */
    const solidVsSource = document.getElementById('solidVertexShader').innerText
    const solidFsSource = document.getElementById('solidFragmentShader').innerText
    const solidProgram = createProgram(gl, solidVsSource, solidFsSource)
    gl.useProgram(solidProgram)
    /* 2.用当前的程序对象绘制图形 */
    const solidVertices = new Float32Array([
        -0.5, 0.5,
        -0.5, -0.5,
        0.5, -0.5,
    ])
    const solidVertexBuffer = gl.createBuffer()
    gl.bindBuffer(gl.ARRAY_BUFFER, solidVertexBuffer)
    gl.bufferData(gl.ARRAY_BUFFER, solidVertices, gl.STATIC_DRAW)
    const solidPosition = gl.getAttribLocation(solidProgram, 'a_Position')
    gl.vertexAttribPointer(solidPosition, 2, gl.FLOAT, false, 0, 0)
    gl.enableVertexAttribArray(solidPosition)
    gl.clear(gl.COLOR_BUFFER_BIT)
    gl.drawArrays(gl.TRIANGLES, 0, 3)
}
```

#### 1-3-绘制纹理三角形

纹理三角形的绘制原理和纯色三角形一样，要用一套新的着色器建立一个新的程序对象，然后用这个新的程序对象进行绘图。

```
function drawTexture(image) {
    /* 1.建立程序对象，并应用 */
    const textureVsSource = document.getElementById('textureVertexShader').innerText
    const textureFsSource = document.getElementById('textureFragmentShader').innerText
    const textureProgram = createProgram(gl, textureVsSource, textureFsSource)
    gl.useProgram(textureProgram)
    
    /* 2.用当前的程序对象绘制图形 */
    // 顶点
    const textureVertices = new Float32Array([
        0.5, 0.5,
        -0.5, 0.5,
        0.5, -0.5,
    ])
    const textureVertexBuffer = gl.createBuffer()
    gl.bindBuffer(gl.ARRAY_BUFFER, textureVertexBuffer)
    gl.bufferData(gl.ARRAY_BUFFER, textureVertices, gl.STATIC_DRAW)
    const texturePosition = gl.getAttribLocation(textureProgram, 'a_Position')
    gl.vertexAttribPointer(texturePosition, 2, gl.FLOAT, false, 0, 0)
    gl.enableVertexAttribArray(texturePosition)
    // 图钉
    const pins = new Float32Array([
        1, 1,
        0, 1,
        1, 0,
    ])
    const pinBuffer = gl.createBuffer()
    gl.bindBuffer(gl.ARRAY_BUFFER, pinBuffer)
    gl.bufferData(gl.ARRAY_BUFFER, pins, gl.STATIC_DRAW)
    const pin = gl.getAttribLocation(textureProgram, 'a_Pin')
    gl.vertexAttribPointer(pin, 2, gl.FLOAT, false, 0, 0)
    gl.enableVertexAttribArray(pin)
    // 纹理
    gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, 1)
    gl.activeTexture(gl.TEXTURE0)
    const texture = gl.createTexture()
    gl.bindTexture(gl.TEXTURE_2D, texture)
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
    const u_Sampler = gl.getUniformLocation(textureProgram, 'u_Sampler')
    gl.uniform1i(u_Sampler, 0)
    
    gl.drawArrays(gl.TRIANGLES, 0, 3)
}
```

#### 1-4-同步绘图

当纹理图形加载成功后，进行同步绘图。

```
const image = new Image()
image.src = './images/erha.jpg'
image.onload = function () {
    drawSolid()
    drawTexture(image)
}
```

接下来我们考虑一个问题。

我想给场景中的图形添加一个动画，比如我想让纯色三角形不断的变换颜色。

那我首先要给纯色三角形的片元着色器开一个uniform变量。

然后接下来是不是就要不断修改这个uniform变量，然后执行上面的drawSolid()和drawTexture(image) 方法呢？

### 2-多着色器动画

在上面的绘图方法中，很多操作都是不需要重复执行的，比如程序对象在建立之后，就不需要再建立了，后面在绘图的时候再接着用。

#### 2-1-给纯色三角形添加一个代表时间的uniform变量

```
<script id="solidFragmentShader" type="x-shader/x-fragment">
    precision mediump float;
    uniform float u_Time;
    void main(){
      float r=(sin(u_Time/200.0)+1.0)/2.0;
      gl_FragColor=vec4(r,0.7,0.4,1);
    }
</script>    
```

#### 2-2-声明不需要重复获取变量

```
let solidProgram, solidVertexBuffer, solidPosition, solidTime = null
let textureProgram, textureVertexBuffer, texturePosition = null
let pinBuffer, pin = null
```

程序对象、缓冲对象、从着色器中获取的attribute变量和uniform变量都是不需要重复获取的。

#### 2-3-初始化方法

-   把绘制纯色三角形的方法变成初始化方法

```
function initSolid() {
    /* 1.建立程序对象*/
    const solidVsSource = document.getElementById('solidVertexShader').innerText
    const solidFsSource = document.getElementById('solidFragmentShader').innerText
    solidProgram = createProgram(gl, solidVsSource, solidFsSource)
    gl.useProgram(solidProgram)

    /* 2.用当前的程序对象绘制图形 */
    const solidVertices = new Float32Array([
        -0.5, 0.5,
        -0.5, -0.5,
        0.5, -0.5,
    ])
    solidVertexBuffer = gl.createBuffer()
    gl.bindBuffer(gl.ARRAY_BUFFER, solidVertexBuffer)
    gl.bufferData(gl.ARRAY_BUFFER, solidVertices, gl.STATIC_DRAW)
    solidPosition = gl.getAttribLocation(solidProgram, 'a_Position')
    gl.vertexAttribPointer(solidPosition, 2, gl.FLOAT, false, 0, 0)
    gl.enableVertexAttribArray(solidPosition)

    solidTime = gl.getUniformLocation(solidProgram, 'u_Time')

    /* gl.clear(gl.COLOR_BUFFER_BIT)
       gl.drawArrays(gl.TRIANGLES, 0, 3) */
}
```

-   把绘制纹理三角形的方法变成初始化方法

```
function initTexture(image) {
    /* 1.建立程序对象，并应用 */
    const textureVsSource = document.getElementById('textureVertexShader').innerText
    const textureFsSource = document.getElementById('textureFragmentShader').innerText
    textureProgram = createProgram(gl, textureVsSource, textureFsSource)
    gl.useProgram(textureProgram)

    /* 2.用当前的程序对象绘制图形 */
    // 顶点
    const textureVertices = new Float32Array([
        0.5, 0.5,
        -0.5, 0.5,
        0.5, -0.5,
    ])
    textureVertexBuffer = gl.createBuffer()
    gl.bindBuffer(gl.ARRAY_BUFFER, textureVertexBuffer)
    gl.bufferData(gl.ARRAY_BUFFER, textureVertices, gl.STATIC_DRAW)
    texturePosition = gl.getAttribLocation(textureProgram, 'a_Position')
    gl.vertexAttribPointer(texturePosition, 2, gl.FLOAT, false, 0, 0)
    gl.enableVertexAttribArray(texturePosition)
    // 图钉
    const pins = new Float32Array([
        1, 1,
        0, 1,
        1, 0,
    ])
    pinBuffer = gl.createBuffer()
    gl.bindBuffer(gl.ARRAY_BUFFER, pinBuffer)
    gl.bufferData(gl.ARRAY_BUFFER, pins, gl.STATIC_DRAW)
    pin = gl.getAttribLocation(textureProgram, 'a_Pin')
    gl.vertexAttribPointer(pin, 2, gl.FLOAT, false, 0, 0)
    gl.enableVertexAttribArray(pin)
    // 纹理
    gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, 1)
    gl.activeTexture(gl.TEXTURE0)
    const texture = gl.createTexture()
    gl.bindTexture(gl.TEXTURE_2D, texture)
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
    const u_Sampler = gl.getUniformLocation(textureProgram, 'u_Sampler')
    gl.uniform1i(u_Sampler, 0)

    // gl.drawArrays(gl.TRIANGLES, 0, 3)
}
```

上面已经注释掉的是需要在绘图时重复执行的。

#### 2-4-把需要重复执行的方法收集一下，用于连续渲染

```
function render(time = 0) {
    gl.clear(gl.COLOR_BUFFER_BIT)
    
    gl.useProgram(solidProgram)
    gl.bindBuffer(gl.ARRAY_BUFFER, solidVertexBuffer)
    gl.vertexAttribPointer(solidPosition, 2, gl.FLOAT, false, 0, 0)
    gl.uniform1f(solidTime, time)
    gl.drawArrays(gl.TRIANGLES, 0, 3)

    gl.useProgram(textureProgram)
    gl.bindBuffer(gl.ARRAY_BUFFER, textureVertexBuffer)
    gl.vertexAttribPointer(texturePosition, 2, gl.FLOAT, false, 0, 0)
    gl.bindBuffer(gl.ARRAY_BUFFER, pinBuffer)
    gl.vertexAttribPointer(pin, 2, gl.FLOAT, false, 0, 0)
    gl.drawArrays(gl.TRIANGLES, 0, 3)

    requestAnimationFrame(render)
}
```

由上可知，在连续渲染时，必须要有的操作：

-   clear() 清理画布
-   useProgram() 应用程序对象
-   bindBuffer() 绑定缓冲区对象
-   vertexAttribPointer() 告诉显卡从当前绑定的缓冲区（bindBuffer()指定的缓冲区）中读取顶点数据
-   drawArrays() 绘图方法

若attribute变量、uniform变量或者图片发生了变化，那就需要对其进行更新。

#### 2-4-当纹理图像加载成功后，初始化绘图方法，连续绘图

```
const image = new Image()
image.src = './images/erha.jpg'
image.onload = function () {
    initSolid()
    initTexture(image)
    render()
}
```

关于进入三维世界的基础知识，我们就先说到这。

接下来，我们需要做几个案例巩固一下之前所学的知识。

做案例之前，我需要先对webgl 做一个简单的架构，将咱们之前所说的顶点索引和多着色器绘图功能都融进去，以便于快速开发。
