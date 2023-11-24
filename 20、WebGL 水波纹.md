---
created: 2023-11-24T14:37:35 (UTC +08:00)
tags: [WebGL]
source: https://juejin.cn/post/7067378141425041415
author: 李伟_Li慢慢
---

# WebGL 水波纹

---
源码：[github.com/buglas/webg…](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fbuglas%2Fwebgl-lesson "https://github.com/buglas/webgl-lesson")

在视图矩阵中，我们可以将算法和艺术相融合，让它充满乐趣。

就像下面的顶点，就是我通过三角函数来实现的。

风乍起，吹皱一池春水。

![1](assets/b224db8e99af4e649e579f3870c96b4etplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

接下来，咱们就说一下正弦型函数 。

### 1-正弦型函数

1.正弦型函数公式

y=Asin(ωx+φ)

2.正弦型函数概念分析

![image-20201129105710682](assets/1f3b992e31e340728b486751bd072584tplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

已知：

-   圆O半径为A
-   点P1 在圆O上
-   ∠xOP1=φ
-   点P1 围绕z轴旋转t 秒后，旋转到点P2的位置
-   点P1的旋转速度为ω/秒

可得：

-   点P1旋转的量就是ω\*t
-   点P2基于x 正半轴的弧度就是∠xOP2=ω\*t+φ
-   点P1 的转动周期T=周长/速度=2π/ω
-   点P1转动的频率f=1/T=ω/2π

3.正弦型函数的图像性质 y=Asin(ωx+φ)

-   A 影响的是正弦曲线的波动幅度

![image-20201129112010521](assets/e060755041ed46b6a89666be2321ad56tplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

-   φ 影响的是正弦曲线的平移

![image-20201130101026144](assets/e3dd084af5cb4612ab2fdf3ffa65c346tplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

-   ω 影响的是正弦曲线的周期，ω 越大，周期越小

![image-20201129114839843](assets/522c708478c74a36872f145e04d46338tplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

通过A、ω、φ我们可以实现正弦曲线的波浪衰减。

![image-20201129120652448](assets/99f25687c40f4472bfbb0f29608bf087tplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

### 2-代码实现

#### 2-1-布点

1.准备好顶点着色器

```
<script id="vertexShader" type="x-shader/x-vertex">
    attribute vec4 a_Position;
    uniform mat4 u_ViewMatrix;
    void main(){
      gl_Position = u_ViewMatrix*a_Position;
      gl_PointSize=3.0;
    }
</script>
```

2.着色器初始化，定义清理画布的底色。

```
const canvas = document.getElementById('canvas');
canvas.width = window.innerWidth;
canvas.height = window.innerHeight;
const gl = canvas.getContext('webgl');

const vsSource = document.getElementById('vertexShader').innerText;
const fsSource = document.getElementById('fragmentShader').innerText;
initShaders(gl, vsSource, fsSource);
gl.clearColor(0.0, 0.0, 0.0, 1.0);
```

3.建立视图矩阵

```
/* 视图矩阵 */
const viewMatrix = new Matrix4().lookAt(
    new Vector3(0.2, 0.3, 1),
    new Vector3(),
    new Vector3(0, 1, 0)
)
```

4.建立波浪对象

```
const wave = new Poly({
  gl,
  vertices: crtVertices(),
  uniforms: {
    u_ViewMatrix: {
      type: 'uniformMatrix4fv',
      value: viewMatrix.elements
    },
  }
})
```

Poly 对象，我在之前"绘制三角形" 里说过，我这里又对其做了下调整，为其添加了uniforms 属性，用于获取和修改uniform 变量。

```
updateUniform() {
    const {gl,uniforms}=this
    for (let [key, val] of Object.entries(uniforms)) {
        const { type, value } = val
        const u = gl.getUniformLocation(gl.program, key)
        if (type.includes('Matrix')) {
            gl[type](u,false,value)
        } else {
            gl[type](u,value)
        }
    }
}
```

crtVertices() 建立顶点集合

```
/* 建立顶点集合 */
function crtVertices(offset = 0) {
    const vertices = []
    for (let z = minPosZ; z < maxPosZ; z += 0.04) {
        for (let x = minPosX; x < maxPosX; x += 0.03) {
            vertices.push(x, 0, z)
        }
    }
    return vertices
}
```

5.渲染

```
gl.clear(gl.COLOR_BUFFER_BIT)
wave.draw()
```

效果如下：

![image-20210331164019359](assets/8b2b3512d7c641e99233226ddc02f740tplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

#### 2-2-塑形

1.建立比例尺，将空间坐标和弧度相映射

```
/* x,z 方向的空间坐标极值 */
const [minPosX, maxPosX, minPosZ, maxPosZ] = [    -0.7, 0.8, -1, 1]
/* x,z 方向的弧度极值 */
const [minAngX, maxAngX, minAngZ, maxAngZ] = [    0, Math.PI * 4, 0, Math.PI * 2]

/* 比例尺：将空间坐标和弧度相映射 */
const scalerX = ScaleLinear(
    minPosX, minAngX, 
    maxPosX, maxAngX
)
const scalerZ = ScaleLinear(minPosZ, minAngZ, maxPosZ, maxAngZ)
```

2.建立正弦型函数

```
function SinFn(a, Omega, phi) {
    return function (x) {
        return a * Math.sin(Omega * x + phi);
    }
}
```

SinFn中的参数与正弦型函数公式相对应：

y=Asin(ωx+φ)

-   y-顶点高度y
-   A-a
-   ω-Omega
-   φ-phi

3.更新顶点高度

基于顶点的x,z 获取两个方向的弧度

将z方向的弧度作为正弦函数的参数

a, Omega, phi 先写死

```
function updateVertices(offset = 0) {
  const { vertices } = wave
  for (let i = 0; i < vertices.length; i += 3) {
    const [posX, posZ] = [vertices[i], vertices[i + 2]]
    const angZ = scalerZ(posZ)
    const Omega = 2
    const a = 0.05
    const phi = 0
    vertices[i + 1] = SinFn(a, Omega, phi)(angZ)
  }
}
```

效果如下：

![image-20210331171833848](assets/7d30bfc989f846818edb46fcec216a7ctplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

4.修改updateVertices()方法中的 phi 值，使其随顶点的x位置变化

```
const phi = scalerX(posX)
```

-   scalerX() 通过x轴上的空间坐标取弧度

效果如下：

![image-20210331172418286](assets/bea310cf18a94efe80a5a622fdafa034tplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

5.修改修改updateVertices()方法中的 a值，使其随顶点的z向弧度变化

```
const a = Math.sin(angZ) * 0.1 + 0.03
```

效果如下：

![image-20210331172842080](assets/aa4068050cd54720af1e4e839b06d3fatplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

#### 2-3-风起

接下来我们可以基于正弦型函数的φ值，来一场动画，让它吹皱一池春水。

1.给updateVertices() 方法一个偏移值，让phi加上此值

```
function updateVertices(offset = 0) {
    const { vertices } = wave
    for (let i = 0; i < vertices.length; i += 3) {
        const [posX, posZ] = [vertices[i], vertices[i + 2]]
        const angZ = scalerZ(posZ)
        const Omega = 2
        const a = Math.sin(angZ) * 0.1 + 0.03
        const phi = scalerX(posX) + offset
        vertices[i + 1] = SinFn(a, Omega, phi)(angZ)
    }
}
```

2.动画

```
let offset = 0
!(function ani() {
    offset += 0.08
    updateVertices(offset)
    wave.updateBuffer()
    gl.clear(gl.COLOR_BUFFER_BIT)
    wave.draw()
    requestAnimationFrame(ani)
})()
```

效果如下：

![1](assets/22a2c489efd6415cb3bd042370ba140ctplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)
