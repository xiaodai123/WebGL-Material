---
created: 2023-11-24T14:55:01 (UTC +08:00)
tags: [WebGL]
source: https://juejin.cn/post/7067426138334691358
author: 李伟_Li慢慢
---

# WebGL 制作VR

---
源码：[github.com/buglas/webg…](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fbuglas%2Fwebgl-lesson "https://github.com/buglas/webgl-lesson")

VR(Virtual Reality) 的意思就是虚拟现实，可以通过VR 眼镜给人环境沉浸感。

VR 的制作需要考虑两点：

-   搭建场景，当前比较常见的搭建场景的方法就是将全景图贴到立方体，或者球体上。
-   场景变换，一般会把透视相机塞进立方体，或者球体里，然后变换场景。

接下来咱们 具体说一下其实现步骤。

### 1-搭建场景

1.用720°全景相机拍摄一张室内全景图。

![room](assets/415f8a6fc1af427eb1e4edb723477fcbtplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

2.在之前地球文件的基础上做修改，把地图替换成上面的室内全景图。

```
const image = new Image()
image.src = './https://blog-st.oss-cn-beijing.aliyuncs.com/16406884365522771800455335651.jpg'
```

3.把相机打入球体之中

```
// 目标点
const target = new Vector3()
//视点
const eye = new Vector3(0.15, 0, 0)
const [fov, aspect, near, far] = [
  60, canvas.width / canvas.height,
  0.1, 1
]
```

效果如下：

![image-20220107131904449](assets/0e002e311abf4ccb98e48a70c0ad996dtplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

现在VR的效果就已经有了，接下来我们还需要考虑VR 场景的变换。

### 2-VR 场景的变换

VR 场景的变换通过相机轨道控制器便可以实现。

当前相机轨道控制器已经具备了旋转、缩放和平移功能。

只不过，针对VR 还得对相机轨道控制器做一下微调。

1.取消相机的平移，以避免相机跑到球体之外。

为相机轨道控制器 OrbitControls 添加一个是否启用平移的功能。

```
const defAttr = () => ({
  ……
  enablePan: true,
})
```

在平移方法中，做个是否平移的判断：

```
pointermove({ clientX, clientY }) {
  const { dragStart, dragEnd, state,enablePan, camera: { type } } = this
  dragEnd.set(clientX, clientY)
  switch (state) {
    case 'pan':
      enablePan&&this[`pan${type}`] (dragEnd.clone().sub(dragStart))
      break
    ……
  }
  dragStart.copy(dragEnd)
}
```

这样就可以在实例化OrbitControls 对象的时候，将enablePan 设置为false，从而禁止相机平移。

```
const orbit = new OrbitControls({
  camera,
  target,
  dom: canvas,
  enablePan: false
})
```

2.使用透视相机缩放VR 场景时，不再使用视点到目标的距离来缩放场景，因为这样的放大效果不太明显。所以，可以直接像正交相机那样，缩放裁剪面。

为OrbitControls 对象的wheel 方法添加一个控制缩放方式的参数。

```
wheel({ deltaY },type=this.camera.type) {
  const { zoomScale} = this
  let scale=deltaY < 0?zoomScale:1 / zoomScale
  this[`dolly${type}`] (scale)
  this.updateSph()
}
```

这样就可以像缩放正交相机那样缩放透视相机。

```
canvas.addEventListener('wheel', event => {
  orbit.wheel(event, 'OrthographicCamera')
})
```

3.在缩放的时候，需要限制一下缩放范围，免得缩放得太大，或者缩小得超出了球体之外。

为OrbitControls 添加两个缩放极值：

-   minZoom 缩放的最小值
-   maxZoom 缩放的最大值

```
const defAttr = () => ({
  ……
  minZoom:0,
  maxZoom: Infinity,
})
```

在相应的缩放方法中，对缩放量做限制：

```
dollyOrthographicCamera(dollyScale) {
  const {camera,maxZoom,minZoom}=this
  const zoom=camera.zoom*dollyScale
  camera.zoom = Math.max(
    Math.min(maxZoom, zoom),
    minZoom
  )
  camera.updateProjectionMatrix()
}
```

在实例化OrbitControls 对象时，设置缩放范围：

```
const orbit = new OrbitControls({
  ……
  maxZoom: 15,
  minZoom: 0.4
})
```

### 3-陀螺仪

VR 的真正魅力在于，你可以带上VR 眼镜，体会身临其境的感觉。

VR 眼镜之所以能给你身临其境的感觉，是因为它内部有一个陀螺仪，可以监听设备的转动，从而带动VR 场景的变换。

目前市场上常见的VR 眼镜有两种：需要插入手机的VR眼镜和一体机。

一般手机里都是有陀螺仪的，因此我们可以用手机来体验VR。

接下来，咱们可以先整个小例子练练手。

我要画个立方体，然后用陀螺仪旋转它。

为了更好的理解陀螺仪。我们把之前的球体变成立方体，在其上面贴上画有东、西、南、北和上、下的贴图。然后在其中打入相机，用陀螺仪变换相机视点，如下图：

![image-20220221161544961](assets/903a07ad1bf24e1996786a0da61d589etplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

1.建立立方体对象Box

```
/*
属性：
  w：宽
  h：高
  d：深
  vertices：顶点集合
  normals：法线集合
  indexes：顶点索引集合
  uv：uv坐标集合
  count：顶点数量
*/
export default class Box{
  constructor(w=1,h=1,d=1){
    this.w=w
    this.h=h
    this.d=d
    this.vertices=null
    this.normals=null
    this.indexes = null
    this.uv = null
    this.count = 36
    this.init()
  }
  init() {
    const [x, y, z] = [this.w / 2, this.h / 2, this.d / 2]
    this.vertices = new Float32Array([
      // 前 0 1 2 3
      -x, y, z, -x, -y, z, x, y, z, x, -y, z,
      // 右 4 5 6 7
      x, y, z, x, -y, z, x, y, -z, x, -y, -z,
      // 后 8 9 10 11
      x, y, -z, x, -y, -z, -x, y, -z, -x, -y, -z,
      // 左 12 13 14 15 
      -x, y, -z, -x, -y, -z, -x, y, z, -x, -y, z,
      // 上 16 17 18 19
      -x, y, -z, -x, y, z, x, y, -z, x, y, z,
      // 下 20 21 22 23 
      -x,-y,z,-x,-y,-z,x,-y,z,x,-y,-z,
    ])
    this.normals = new Float32Array([
      0, 0, 1, 0, 0, 1, 0, 0, 1, 0, 0, 1,
      1, 0, 0, 1, 0, 0, 1, 0, 0, 1, 0, 0,
      0, 0, -1, 0, 0, -1, 0, 0, -1, 0, 0, -1, 
      -1, 0, 0, -1, 0, 0, -1, 0, 0, -1, 0, 0,
      0, 1, 0, 0, 1, 0, 0, 1, 0, 0, 1, 0,
      0,-1,0,0,-1,0,0,-1,0,0,-1,0,
    ])
    /* this.uv = new Float32Array([
      0,1,0,0,1,1,1,0,
      0,1,0,0,1,1,1,0,
      0,1,0,0,1,1,1,0,
      0,1,0,0,1,1,1,0,
      0,1,0,0,1,1,1,0,
      0,1,0,0,1,1,1,0,
    ]) */
    this.uv = new Float32Array([
      0,1, 0,0.5, 0.25,1, 0.25,0.5,
      0.25,1, 0.25,0.5, 0.5,1, 0.5,0.5,
      0.5,1, 0.5,0.5, 0.75,1, 0.75,0.5,
      0,0.5,0,0,0.25,0.5,0.25,0,
      0.25,0.5,0.25,0,0.5,0.5,0.5,0,
      0.5,0.5,0.5,0,0.75,0.5,0.75,0,
    ])
    this.indexes = new Uint16Array([
      0, 1, 2, 2, 1, 3,
      4, 5, 6, 6, 5, 7,
      8, 9, 10, 10, 9, 11,
      12, 13, 14, 14, 13, 15,
      16, 17, 18, 18, 17, 19, 
      20,21,22,22,21,23
    ])
  }
}
```

2.在Google浏览器中打开传感器，模拟陀螺仪的旋转。

一我们在电脑里做测试的时候，需要用浏览器里的开发者工具

![传感器](assets/6fb71d2962494f43894b322138b8dde1tplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

之后我们可以在js中通过deviceorientation 事件监听陀螺仪的变化。

从deviceorientation 事件的回调参数event里，可以解构出alpha, beta, gamma 三个参数。

```
window.addEventListener('deviceorientation', (event) => {
  const { alpha, beta, gamma }=event
})
```

alpha, beta, gamma对应了陀螺仪欧拉旋转的三个参数。

在右手坐标系中，其概念如下：

-   alpha：绕世界坐标系的y轴逆时针旋转的角度，旋转范围是\[-180°,180°)
-   beta：绕本地坐标系的x轴逆时针旋转的角度，旋转范围是\[-180°,180°)
-   gamma ：绕本地坐标系的z轴顺时针旋转的角度，旋转范围是\[-90°,90°)

注：alpha, beta, gamma具体是绕哪个轴旋转，跟我们当前程序所使用的坐标系有关。所以大家之后若是看到有些教程在说陀螺仪时，跟我说的不一样，也不要惊奇，只要在实践中没有问题就可以。

陀螺仪欧拉旋转的顺序是'YXZ'，而不是欧拉对象默认的'XYZ'。

欧拉旋转顺序是很重要的，如果乱了，就无法让VR旋转与陀螺仪相匹配了。

接下来，基于之前VR.html 文件做下调整。

3.建立立方体对象

```
import Box from './lv/Box.js'
const box = new Box(1, 1, 1)
```

4.调整相机数据

```
// 目标点
const target = new Vector3()
//视点
const eye = new Vector3(0, 0.45, 0.0001)
const [fov, aspect, near, far] = [
  120, canvas.width / canvas.height,
  0.01, 2
]
// 透视相机
const camera = new PerspectiveCamera(fov, aspect, near, far)
camera.position.copy(eye)
```

相机的视线是根据陀螺仪的初始状态设置的。

在陀螺仪的alpha, beta, gamma皆为0的情况下，手机成俯视状态。

![image-20220114152104090](assets/971c945cde284e3e86e1ef6431a96e28tplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

所以，相机也要成俯视状态，因此视点的y值设置为0.45。

然而，视线也不能完全垂直，因为这样视点绕y轴旋转就会失效。

所以，视点的z值给了一个较小的数字0.0001。

5.场景的渲染和之前是一样。

```
// 轨道控制器
const orbit = new OrbitControls({
  camera,
  target,
  dom: canvas,
  enablePan: false,
  maxZoom: 15,
  minZoom: 0.4
})

// 场景
const scene = new Scene({ gl })
//注册程序对象
scene.registerProgram(
  'map',
  {
    program: createProgram(
      gl,
      document.getElementById('vs').innerText,
      document.getElementById('fs').innerText,
    ),
    attributeNames: ['a_Position', 'a_Pin'],
    uniformNames: ['u_PvMatrix', 'u_ModelMatrix','u_Sampler']
  }
)

//立方体
const matBox = new Mat({
  program: 'map',
  data: {
    u_PvMatrix: {
      value: orbit.getPvMatrix().elements,
      type: 'uniformMatrix4fv',
    },
    u_ModelMatrix: {
      value: new Matrix4().elements,
      type: 'uniformMatrix4fv',
    },
  },
})
const geoBox = new Geo({
  data: {
    a_Position: {
      array: box.vertices,
      size: 3
    },
    a_Pin: {
      array: box.uv,
      size: 2
    }
  },
  index: {
    array: box.indexes
  }
})

//加载图片
const image = new Image()
image.src = './images/magic.jpg'
image.onload = function () {
  matBox.maps.u_Sampler = {
    image,
    magFilte: gl.LINEAR,
    minFilter: gl.LINEAR,
  }
  scene.add(new Obj3D({
    geo: geoBox,
    mat: matBox
  }))
  render()
}

function render() {
  orbit.getPvMatrix()
  scene.draw()
  requestAnimationFrame(render)
}
```

效果如下：

![image-20220114152104090](assets/2b7a8ac55c744702a6f209842a8ee809tplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

7.监听陀螺仪事件时，需要考虑三件事：

-   判断当前设备里是否有陀螺仪。
-   让用户触发浏览器对陀螺仪事件的监听，可通过click 之类的事件触发。
-   若系统是ios，需要请求用户许可。

css 样式：

```
html {height: 100%;}
body {
  margin: 0;
  height: 100%;
  overflow: hidden
}
.wrapper {
  display: flex;
  position: absolute;
  justify-content: center;
  align-items: center;
  width: 100%;
  height: 100%;
  top: 0;
  background-color: rgba(0, 0, 0, 0.4);
  z-index: 10;
}
#playBtn {
  padding: 24px 24px;
  border-radius: 24px;
  background-color: #00acec;
  text-align: center;
  color: #fff;
  cursor: pointer;
  font-size: 24px;
  font-weight: bold;
  border: 6px solid rgba(255, 255, 255, 0.7);
  box-shadow: 0 9px 9px rgba(0, 0, 0, 0.7);
}
```

html 标签：

```
<canvas id="canvas"></canvas>
<div class="wrapper">
  <div id="playBtn">开启VR之旅</div>
</div>
```

js 代码：

```
// 遮罩
const wrapper = document.querySelector('.wrapper')
// 按钮
const btn = document.querySelector('#playBtn')
// 判断设备中是否有陀螺仪
if (window.DeviceMotionEvent) {
  // 让用户触发陀螺仪的监听事件
  btn.addEventListener('click', () => {
    //若是ios系统，需要请求用户许可
    if (DeviceMotionEvent.requestPermission) {
      requestPermission()
    } else {
      rotate()
    }
  })
} else {
  btn.innerHTML = '您的设备里没有陀螺仪！'
}

//请求用户许可
function requestPermission() {
  DeviceMotionEvent.requestPermission()
    .then(function (permissionState) {
    // granted:用户允许浏览器监听陀螺仪事件
    if (permissionState === 'granted') {
      rotate()
    } else {
      btn.innerHTML = '请允许使用陀螺仪🌹'
    }
  }).catch(function (err) {
    btn.innerHTML = '请求失败！'
  });
}

//监听陀螺仪
function rotate() {
  wrapper.style.display = 'none'
  window.addEventListener('deviceorientation', ({ alpha, beta, gamma }) => {
    const rad = Math.PI / 180
    const euler = new Euler(
      beta * rad,
      alpha * rad,
      -gamma * rad,
      'YXZ'
    )
    camera.position.copy(
      eye.clone().applyEuler(euler)
    )
    orbit.updateCamera()
    orbit.resetSpherical()
  })
}
```

关于陀螺仪的基本用法我们就说到这。

我之前在网上看了一些陀螺仪相关的教程，很多都没说到点上，因为若是不知道欧拉旋转的概念，就说不明白陀螺仪。

所以，我们一定不要舍不得花时间学习图形学，图形学关系着我们自身发展的潜力。

接下来我们可以在VR.html 文件里，以同样的原理把陀螺仪写进去，并结合项目的实际需求做一下优化。

### 4-VR+陀螺仪

我们可以先把陀螺仪封装一下，以后用起来方便。

1.封装一个陀螺仪对象Gyro.js

```
import {Euler} from 'https://unpkg.com/three/build/three.module.js';

const rad = Math.PI / 180

const defAttr = () => ({
  //用于触发事件的按钮
  btn: null,
  //没有陀螺仪
  noDevice: () => { },
  //当点击按钮时
  onClick: () => { },
  //可以使用陀螺仪时触发一次
  init: () => { },
  //用户拒绝开启陀螺仪
  reject: () => { },
  //请求失败
  error: () => { },
  //陀螺仪变换
  change: () => { },
})

export default class Gyro {
  constructor(attr) {
    Object.assign(this, defAttr(), attr)
  }
  start() {
    const { btn } = this
    if (window.DeviceMotionEvent) {
      // 让用户触发陀螺仪的监听事件
      btn.addEventListener('click', () => {
        this.onClick()
        //若系统是ios，需要请求用户许可
        if (DeviceMotionEvent.requestPermission) {
          this.requestPermission()
        } else {
          this.translate()
        }
      })
    } else {
      this.noDevice()
    }
  }
  //请求用户许可
  requestPermission() {
    DeviceMotionEvent.requestPermission()
      .then((permissionState) => {
      // granted:用户允许浏览器监听陀螺仪事件
      if (permissionState === 'granted') {
        this.translate()
      } else {
        this.reject()
      }
    }).catch((err) => {
      this.error(err)
    });
  }
  // 监听陀螺仪
  translate() {
    this.init()
    window.addEventListener('deviceorientation', ({ beta, alpha, gamma }) => {
      this.change(new Euler(
        beta * rad,
        alpha * rad,
        -gamma * rad,
        'YXZ'
      ))
    })
  }
}
```

2.把陀螺仪对象引入VR文件

```
import Gyro from './lv/Gyro.js'

// 遮罩
const wrapper = document.querySelector('.wrapper')
// 按钮
const btn = document.querySelector('#playBtn')
// 陀螺仪
const gyro = new Gyro({
  btn,
  noDevice: () => {
    btn.innerHTML = '您的设备里没有陀螺仪！'
  },
  reject: () => {
    btn.innerHTML = '请允许使用陀螺仪🌹'
  },
  error: () => {
    btn.innerHTML = '请求失败！'
  },
  init: () => {
    wrapper.style.display = 'none'
  },
  change: (euler) => {
    camera.position.copy(
      eye.clone().applyEuler(euler)
    )
    orbit.updateCamera()
    orbit.resetSpherical()
  }
})
gyro.start()
```

3.优化图像的加载。

在实际开发中，为了让用户看到比较清晰的效果，往往需要使用比较大的全景图，比如4096\*2048 的尺寸。

大尺寸的图片加载起来会很慢，为了减少用户的等待，我们可以先加载一个较小的图片，然后慢慢的过度到大图。

比如，我们可以从小到大准备4张不同尺寸的全景图：

-   512\*256
-   1024\*512
-   2048\*1024
-   4096\*2048

接下来先加载第一张小图，将其显示出来后，再依次加载后面的大图。

```
//图片序号
let level = 0
//加载图片
loadImg()
function loadImg() {
  const image = new Image()
  image.src = `./images/room${level}.jpg`
  image.onload = function () {
    if (level === 0) {
      firstRender(image)
    } else {
      //更新贴图
      matEarth.setMap('u_Sampler', { image })
    }
    if (level < 3) {
      level++
      loadImg()
    }
  }
}
// 第一次渲染
function firstRender(image) {
  btn.innerHTML = '开启VR之旅'
  matEarth.maps.u_Sampler = {
    image,
    magFilte: gl.LINEAR,
    minFilter: gl.LINEAR,
  }
  scene.add(new Obj3D({
    geo: geoEarth,
    mat: matEarth
  }))
  render()
}
```

与此同时，我们还得微调一下Mat.js里的更新贴图方法：

```
updateMap(gl,map,ind) {
  ……
  //gl.bindTexture(gl.TEXTURE_2D, null)
}
```

我们需要取消对纹理缓冲区的清理。

以前我们要清理纹理缓冲区，是因为我们不需要对纹理缓冲区里的纹理对象进行更新，将其清理掉还可以节约内存。

而现在我们需要对纹理缓冲区里的纹理对象进行更新，那就不能清理掉了。

### 5-开场动画

开场动画的作用，就是给用户一个吸引眼球的效果，提高项目的趣味性。

开场动画的开场方式有很多，咱们这里就说一个比较常见的：从上帝视角到普通视角的过度。

上帝视角就是一个俯视的视角，视野一定要广，如下图：

![image-20220107220745375](assets/2e092dec6e2e480c9d21ee4aa6a4236ctplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

之后，我会用补间动画，将其过度到普通视角，如下图：

![image-20220107131904449](assets/c3aad4150b8d4a9ca59066795042aaactplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

从上帝视角到普通视角的变换涉及以下属性：

-   相机视点的位置
-   相机视椎体的垂直视角

接下来我们便可以基于上面的属性做缓动动画。

1.把当前的相机视角调为上帝视角。

```
// 目标点
const target = new Vector3()
//视点-根据陀螺仪做欧拉旋转
const eye = new Vector3( 0.15,0, 0.0001)
// 透视相机
const [fov, aspect, near, far] = [
  130, canvas.width / canvas.height,
  0.01, 2
]
const camera = new PerspectiveCamera(fov, aspect, near, far)
// 上帝视角
camera.position.set(0, 0.42, 0)
```

2.基于相机的视点和视椎体的垂直夹角建立两个目标变量

```
const endPos = camera.position.clone()
let endFov = fov
```

上面的两个目标变量默认是和当前相机一致的，之后陀螺仪发生变化时会对其做修改。

3.在陀螺仪发生变化时，设置目标变量

```
// 陀螺仪
const gyro = new Gyro({
  ……
  change: (euler) => {
    endFov = 60
    endPos.copy(
      eye.clone().applyEuler(euler)
    )
  }
})
```

当前的开场动画是针对有陀螺仪的手机而言的，接下来再做对PC端的开场动画。

4.当鼠标点击“开启VR之旅” 的时候，若浏览器在PC端，将视角调为普通视角。

```
const pc = isPC()
const gyro = new Gyro({
     ……
  onClick: () => {
    if (pc) {
      endPos.set(0.15, 0, 0.0001)
      endFov = 60
    }
  }
})
```

isPC() 是判断浏览器是否在PC端的方法。

```
const isPC=()=>!navigator.userAgent.match(/(phone|pad|pod|iPhone|iPod|ios|iPad|Android|Mobile|BlackBerry|IEMobile|MQQBrowser|JUC|Fennec|wOSBrowser|BrowserNG|WebOS|Symbian|Windows Phone)/i)
```

5.建立缓动动画方法

```
function tween(ratio = 0.05) {
  //若当前设备为PC,缓动结束后就不再缓动，之后的变换交给轨道控制器
  if (pc && camera.fov < endFov + 1) { return }

  camera.position.lerp(endPos, ratio)
  camera.fov += (endFov - camera.fov) * ratio
  camera.updateProjectionMatrix()
  orbit.updateCamera()
  orbit.resetSpherical()
}
```

-   camera.updateProjectionMatrix() 更新投影矩阵。
    
    因为上面更新了视椎体的垂直夹角，所以相机的投影矩阵也要做同步的更新，具体原理参见之前的透视投影矩阵。
    
-   orbit.updateCamera() 更新相机，将相机的视点位置和视线写入相机的本地矩阵里。
    
-   orbit.resetSpherical() 重置球坐标。用鼠标旋转相机的时候会将旋转信息写入球坐标。
    

6.在连续渲染时执行缓动动画

```
function render() {
  tween()
  orbit.getPvMatrix()
  scene.draw()
  requestAnimationFrame(render)
}
```

### 6-添加标记点

标记点可以告诉用户不同区域的名称，或者为用户指引方向。

在添加标记点的时候，我们要考虑两点：

-   如何添加标记点
-   如何制作标记点

接下来我会在VR 场景中添加HTML类型的标记点，这种方法是比较常见的。

在VR中添加HTML类型标记点的核心问题是：如何让HTML 标记点随VR 场景同步变换。

解决这个问题要按以下几步走：

![image-20220129161308872](assets/e51e82ea772445208b9841925de5ecbftplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

1.鼠标点击canvas 画布时，将此点的canvas坐标转世界坐标A。

 注：canvas坐标转世界坐标的原理在“进入三维世界”的选择立方体里说过。

2.以视点为起点，A点位方向做射线EA。

3.求射线EA与球体的交点P，此点便是标记点在世界坐标系内的世界坐标。

4.在变换VR场景时，将标记点的世界坐标转canvas坐标，然后用此坐标更新标记点的位置。

在上面的步骤中，第3步是关键，我们详细讲解以下。

已知：

-   射线 EA
-   球体球心为O，半径为 r

求：射线 EA与球体的交点P

解：

先判断射线的基线与球体的关系。

设：EA的单位向量为v

用EO叉乘EA的单位向量，求得球心O 到直线的距离|OB|

```
|OB|=|EO^v|
```

基于|OB|和半径r，可以知道基线与球体的关系：

![image-20220129172220487](assets/b337afb56dde4c4a8ecadb3a52e32cc4tplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

-   |OB|>r，直线与球体相离，没有交点
-   |OB|=r，直线与球体相切，1个交点 ，交点为B点

```
B=v*(EO·v)
```

-   |OB|<r，直线与球体相交，2个交点，其算法如下：

![image-20220129161308872](assets/32b4915a544f4a56bf664014255c25fbtplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

在直线EA上的点可以写做：

```
E+λv, λ∈R
```

直线和球体相交的点满足：

```
(E+λv-O)²=r²
```

(λv+OE)²可理解为向量OP的长度的平方。

E-O可写做OE：

```
(λv+OE)²=r²
```

展开上式：

```
λ²v²+2λv·OE+OE²=r²
```

因为：单位向量与其自身的点积等于1

所以：

```
λ²+2λv·OE+OE²=r²
λ²+2λv·OE+OE²-r²=0
```

解一元二次方程式，求λ：

为了方便书写，设：

```
b=2v·OE
c=OE²-r²
```

则：

```
λ²+λb+c=0
λ²+bλ+(b/2)²=-c+(b/2)²
(λ+b/2)²=(b²-4c)/4
λ+b/2=±sqrt(b²-4c)/2
λ=(-b±sqrt(b²-4c))/2
```

知道了λ ，也就可以直线与球体的交点。

```
λv+OE
```

注：当λ小于0时，交点在射线EA的反方向，应该舍弃。

关于射线与球体的交点，咱们就说到这。

接下来咱们基于之前的VR.html 代码，在VR球体上打一个标记点。

1.建立一个标记点。当前先不考虑标记点的文字内容的编辑，只关注标记点的位置。

```
<style>
  #mark {
    position: absolute;
    top: 0;
    left: 0;
    color: #fff;
    background-color: rgba(0, 0, 0, 0.6);
    padding: 6px 12px;
    border-radius: 3px;
    user-select: none;
  }
</style>

<div id="mark">标记点</div>
```

2.获取标记点，建立markWp变量，用于记录标记点的世界坐标。

```
// 标记
const mark = document.querySelector('#mark')
// 标记点的世界位
let markWp = null
```

3.当鼠标双击canvas 画布的时候，添加标记点。

```
canvas.addEventListener('dblclick', event => {
  addMark(event)
})
```

addMark() 方法做了3件事情：

-   worldPos() 把鼠标点击在canvas画布上的canvas坐标转换为世界坐标.
    
    此方法咱们之前写过，从Utils.js 中引入即可。
    
-   getMarkWp() 根据鼠标点的世界坐标设置标记点的世界坐标。
    
    这便是参照之前射线和球体交点的数学公式来写的。
    
    注：鼠标点的世界坐标并不是标记点的世界坐标。
    
-   setMarkCp() 设置标记点的canvas坐标位。
    

```
function addMark(event) {
  //鼠标点的世界坐标
  const A = worldPos(event, canvas, pvMatrix)
  //获取标记点的世界坐标
  markWp =getMarkWp(camera.position, A, target, earth.r)
  //设置标记点的canvas坐标位
  setMarkCp(event.clientX, event.clientY)
}

/* 获取射线和球体的交点
  E 射线起点-视点
  A 射线目标点
  O 球心
  r 半径
*/
function getMarkWp(E, A, O, r) {
  const v = A.clone().sub(E).normalize()
  const OE = E.clone().sub(O)
  //b=2v·OE
  const b = v.clone().multiplyScalar(2).dot(OE)
  //c=OE²-r²
  const c = OE.clone().dot(OE) - r * r
  //λ=(-b±sqrt(b²-4c))/2
  const lambda = (-b + Math.sqrt(b * b - 4 * c)) / 2
  //λv+OE
  return v.clone().multiplyScalar(lambda).add(OE)
}

//设置标记点的canvas坐标位
function setMarkCp(x, y) {
  mark.style.left = `${x}px`
  mark.style.top = `${y}px`
}
```

4.当旋转和缩放相机的时候，对标记点进行同步变换。

```
canvas.addEventListener('pointermove', event => {
  orbit.pointermove(event)
  updateMarkCp()
})
canvas.addEventListener('wheel', event => {
  orbit.wheel(event, 'OrthographicCamera')
  updateMarkCp()
})

//更新标记点的位置
function updateMarkCp() {
  if (!markWp) { return }

  //判断标记点在相机的正面还是背面
  const {position}=camera
  const dot = markWp.clone().sub(position).dot(
    target.clone().sub(position)
  )
  if (dot > 0) {
    mark.style.display = 'block'
  } else {
    mark.style.display = 'none'
  }

  // 将标记点的世界坐标转裁剪坐标
  const { x, y } = markWp.clone().applyMatrix4(pvMatrix)
  // 将标记点的裁剪坐标转canvas坐标
  setMarkCp(
    (x + 1) * canvas.width / 2,
    (-y + 1) * canvas.height / 2
  )
}
```

之后围绕标记点还可以再进一步优化：

-   使标记点的文字内容可编辑
-   优化标记点样式
-   使标记点可拖拽
-   添加多个标记点
-   ……

这些都是正常的前端业务逻辑，我这里就只重点说图形学相关的知识了。

之后大家可以参考一个叫“[720云](https://link.juejin.cn/?target=https%3A%2F%2F720yun.com%2F "https://720yun.com/")”的网站，它就是专业做VR的。

![image-20220201151724511](assets/227adf2ae8b140b3918dbf4de5fa80bftplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

### 7-VR 场景的切换

在实际开发中我们通常会遇到这样的需求：

在客厅的VR场景中有一扇进入卧室的门，在卧室门上有一个写着“卧室”的标记点。

当我们点击“卧室”标记点时，就进入卧室的VR 中。

这个需求便涉及了客厅和卧室两个VR场景的切换。

两个VR场景的切换最简单的实现方法就是直接换贴图了，这个方法快速、简单、直接，所以咱们先用代码写一下这个功能。

1.准备一份VR数据，我把它放进了vr.json 文件里，这就相当于后端数据库里的数据了。

```
[  {    "id": 1,    "imgSrc": "./https://blog-st.oss-cn-beijing.aliyuncs.com/16406884365522771800455335651.jpg",    "eye": [-0.14966274559865525, -0.009630159419482085, 0.002884893313037499],
    "marks": [
      {
        "name": "次卧",
        "pos": [-0.45099085840209097, 0.0889607157340315, 0.19670596506927274],
        "link": 2
      },
      {
        "name": "主卧",
        "pos": [-0.34961792927865026, 0.30943492493218633, -0.17893387258739163],
        "link": 3
      }
    ]
  },
  {
    "id": 2,
    "imgSrc": "./images/secBed.jpg",
    "eye": [-0.14966274559865525, -0.009630159419482085, 0.002884893313037499],
    "marks": [
      {
        "name": "客厅",
        "pos": [-0.34819482247111166, 0.29666506812630905, -0.20186679508508473],
        "link": 1
      }
    ]
  },
  {
    "id": 3,
    "imgSrc": "./images/mainBed.jpg",
    "eye": [-0.14966274559865525, -0.009630159419482085, 0.002884893313037499],
    "marks": [
      {
        "name": "客厅",
        "pos": [-0.07077938553590507, 0.14593627464082626, -0.47296181910077806],
        "link": 1
      }
    ]
  }
]
```

当前这个json 文件里有3个VR 场景的数据，分别是客厅、主卧、次卧。

-   imgSrc VR图片
    
-   eye 相机视点
    
-   marks 标记点
    
    -   name 标记点名称
    -   pos 标记点世界位，可在上一节添加标记点的时候，将其存储到后端
    -   link 当前标记点链接的VR 的id

2.基于之前添加标记点文件，做下调整，建立一个标记点容器marks，之后会往marks里放html类型的标记点。

```
<style>
  .mark {
    position: absolute;
    transform: translate(-50%, -50%);
    top: 0;
    left: 0;
    color: #fff;
    background-color: rgba(0, 0, 0, 0.6);
    padding: 6px 12px;
    border-radius: 3px;
    user-select: none;
    cursor: pointer;
  }
</style>

<body>
  <canvas id="canvas"></canvas>
  <div id="marks"></div>
  ……
</body>
```

3.简化出一个VR场景。

```
import { createProgram, worldPos } from "/jsm/Utils.js";
import {
  Matrix4, PerspectiveCamera, Vector3
} from 'https://unpkg.com/three/build/three.module.js';
import OrbitControls from './lv/OrbitControls.js'
import Mat from './lv/Mat.js'
import Geo from './lv/Geo.js'
import Obj3D from './lv/Obj3D.js'
import Scene from './lv/Scene.js'
import Earth from './lv/Earth.js'

const canvas = document.getElementById('canvas');
canvas.width = window.innerWidth;
canvas.height = window.innerHeight;
let gl = canvas.getContext('webgl');

// 球体
const earth = new Earth(0.5, 64, 32)

// 目标点
const target = new Vector3()
const [fov, aspect, near, far] = [
  60, canvas.width / canvas.height,
  0.1, 5
]
// 透视相机
const camera = new PerspectiveCamera(fov, aspect, near, far)
// 轨道控制器
const orbit = new OrbitControls({
  camera,
  target,
  dom: canvas,
  enablePan: false,
  maxZoom: 15,
  minZoom: 0.4
})

//投影视图矩阵
const pvMatrix = orbit.getPvMatrix()

//标记
const marks = document.querySelector('#marks')

// 场景
const scene = new Scene({ gl })
//注册程序对象
scene.registerProgram(
  'map',
  {
    program: createProgram(
      gl,
      document.getElementById('vs').innerText,
      document.getElementById('fs').innerText,
    ),
    attributeNames: ['a_Position', 'a_Pin'],
    uniformNames: ['u_PvMatrix', 'u_ModelMatrix', 'u_Sampler']
  }
)

//球体
const mat = new Mat({
  program: 'map',
  data: {
    u_PvMatrix: {
      value: orbit.getPvMatrix().elements,
      type: 'uniformMatrix4fv',
    },
    u_ModelMatrix: {
      value: new Matrix4().elements,
      type: 'uniformMatrix4fv',
    },
  },
  maps: {
    u_Sampler: {
      magFilter: gl.LINEAR,
      minFilter: gl.LINEAR,
    }
  }
})
const geo = new Geo({
  data: {
    a_Position: {
      array: earth.vertices,
      size: 3
    },
    a_Pin: {
      array: earth.uv,
      size: 2
    }
  },
  index: {
    array: earth.indexes
  }
})
scene.add(new Obj3D({ geo, mat }))

// 渲染
render()

function render() {
  orbit.getPvMatrix()
  scene.draw()
  requestAnimationFrame(render)
}

/* 取消右击菜单的显示 */
canvas.addEventListener('contextmenu', event => {
  event.preventDefault()
})
/* 指针按下时，设置拖拽起始位，获取轨道控制器状态。 */
canvas.addEventListener('pointerdown', event => {
  orbit.pointerdown(event)
})
/* 指针移动时，若控制器处于平移状态，平移相机；若控制器处于旋转状态，旋转相机。 */
canvas.addEventListener('pointermove', event => {
  orbit.pointermove(event)
  updateMarkCp()
})
/* 指针抬起 */
canvas.addEventListener('pointerup', event => {
  orbit.pointerup(event)
})
/* 滚轮事件 */
canvas.addEventListener('wheel', event => {
  orbit.wheel(event, 'OrthographicCamera')
  updateMarkCp()
})
```

当前是渲染不出东西来的，因为我还没有给球体指定贴图。

3.请求VR 数据，更新VR。

```
let data;
let curVr;
fetch('./data/vr.json')
  .then((res) => res.json())
  .then(dt => {
    data = dt
    curVr = getVrById(1)
    //更新VR
    updateVr()
    // 渲染
    render()
  });

//根据id获取VR数据
function getVrById(id) {
  for (let i = 0; i < data.length; i++) {
    if (id === data[i].id) {
      return data[i]
    }
  }
}

//根据数据更新VR
function updateVr() {
  const image = new Image()
  image.src = curVr.imgSrc
  image.onload = function () {
    //更新图片
    mat.setMap('u_Sampler', { image })
    //更新相机视点
    camera.position.set(...curVr.eye)
    orbit.updateCamera()
    orbit.resetSpherical()
    //显示标记点
    showMark()
  }
}

//显示标记点
function showMark() {
  curVr.marks.forEach(ele => {
    const div = document.createElement('div')
    div.className = 'mark'
    div.innerText = ele.name
    div.setAttribute('data-link', ele.link)
    marks.append(div)
  })
}

//更新标记点的canvas坐标位
function updateMarkCp() {
  if (!marks.children.length) { return }
  const { position } = camera
  const EO = target.clone().sub(position)
  curVr.marks.forEach((ele, ind) => {
    const markWp = new Vector3(...ele.pos)
    const mark = marks.children[ind]
    const dot = markWp.clone().sub(position).dot(EO)
    mark.style.display = dot > 0 ? 'block' : 'none'
    const { x, y } = markWp.clone().applyMatrix4(pvMatrix)
    mark.style.left = `${(x + 1) * canvas.width / 2}px`
    mark.style.top = `${(-y + 1) * canvas.height / 2}px`
  })
}
```

5.点击标记点时，根据标记点的data-link 更新VR

```
marks.addEventListener('click', ({ target }) => {
  if (target.className !== 'mark') { return }
  marks.innerHTML = ''
  curVr = getVrById(parseInt(target.getAttribute('data-link')))
  updateVr()
})
```

6.连续渲染的时候，更新标记点的canvas坐标位。

```
function render() {
  orbit.getPvMatrix()
  scene.draw()
  updateMarkCp()
  requestAnimationFrame(render)
}
```

7.把鼠标的位移事件绑定到window上。

当鼠标移动到标记点上时，会被标记点卡住，无法移动，这是因为标记点挡住了canvas，所以不能再把鼠标事件绑定到canvas上了。

```
window.addEventListener('pointermove', event => {
  orbit.pointermove(event)
})
```

到目前为止，VR 场景切换的基本功能已经搞定了。

然而，老板可能还会让我们给VR一个过度动画，因为别人家的VR 也会有这样的效果。

### 8-VR场景的过度动画

当前显示的VR就叫它旧VR，接下来要显示的VR就叫它新VR。

这两个VR可以想象成两张图片，旧VR在新VR上面，旧VR遮挡了新VR。

在切换VR的时候，我们可以先实现这样一个过渡效果：让旧VR渐隐，从而露出下面的新VR。

帧缓冲区便可以视之为存储在内存里的图片。

我们将两个VR场景分别渲染到两个帧缓冲区里后，便可以基于透明度融合一下，然后贴到一个充满窗口的平面上，从而实现过度效果。

1.封装个场景对象出来，这个场景里只有一个充满窗口的平面，之后会把帧缓冲区贴上去。

-   VRPlane.js

```
import { createProgram } from "./Utils.js";
import Mat from './Mat.js'
import Geo from './Geo.js'
import Obj3D from './Obj3D.js'
import Scene from './Scene.js'
import Rect from './Rect.js'

const vs = `
  attribute vec4 a_Position;
  attribute vec2 a_Pin;
  varying vec2 v_Pin;
  void main(){
    gl_Position=a_Position;
    v_Pin=a_Pin;
  }
`

const fs = `
  precision mediump float;
  uniform float u_Ratio;
  uniform sampler2D u_SampNew;
  uniform sampler2D u_SampOld;
  varying vec2 v_Pin;
  void main(){
    vec4 t1 = texture2D( u_SampNew, v_Pin );
    vec4 t2 = texture2D( u_SampOld, v_Pin );
    gl_FragColor = mix(t2,t1, u_Ratio);
  }
`

export default class VRPlane extends Scene{
  constructor(attr){
    super(attr)
    this.createModel()
  }
  createModel() {
    const { gl } = this
    this.registerProgram('map', {
      program: createProgram(gl,vs,fs),
      attributeNames: ['a_Position', 'a_Pin'],
      uniformNames: ['u_SampNew', 'u_SampOld', 'u_Ratio']
    })
    const mat = new Mat({
      program: 'map',
      data: {
        u_Ratio: {
          value: 0,
          type: 'uniform1f',
        },
      }
    })
    const rect = new Rect(2, 2, 0.5, 0.5)
    const geo = new Geo({
      data: {
        a_Position: {
          array: rect.vertices,
          size: 3
        },
        a_Pin: {
          array: rect.uv,
          size: 2
        }
      },
      index: {
        array: rect.indexes
      }
    })
    this.add(new Obj3D({ geo, mat }))
    this.mat=mat
  }
  
}
```

2.封装一个包含VR场景的帧缓冲区对象。

-   VRFrame.js

```
import { createProgram } from "./Utils.js";
import {Matrix4} from 'https://unpkg.com/three/build/three.module.js';
import Mat from './Mat.js'
import Geo from './Geo.js'
import Obj3D from './Obj3D.js'
import Earth from './Earth.js'
import Frame from './Frame.js'

const vs = `
  attribute vec4 a_Position;
  attribute vec2 a_Pin;
  uniform mat4 u_PvMatrix;
  uniform mat4 u_ModelMatrix;
  varying vec2 v_Pin;
  void main(){
    gl_Position=u_PvMatrix*u_ModelMatrix*a_Position;
    v_Pin=a_Pin;
  }
`
const fs = `
  precision mediump float;
  uniform sampler2D u_Sampler;
  varying vec2 v_Pin;
  void main(){
    gl_FragColor=texture2D(u_Sampler,v_Pin);
  }
`

/* 参数
  gl,
  orbit,
*/
export default class VRFrame extends Frame{
  constructor(attr){
    super(attr)
    this.createModel()
  }
  createModel() {
    const { orbit, gl } = this

    this.registerProgram('map', {
      program: createProgram(gl,vs,fs),
      attributeNames: ['a_Position', 'a_Pin'],
      uniformNames: ['u_PvMatrix', 'u_ModelMatrix', 'u_Sampler']
    })
    const mat = new Mat({
      program: 'map',
      data: {
        u_PvMatrix: {
          value: orbit.getPvMatrix().elements,
          type: 'uniformMatrix4fv',
        },
        u_ModelMatrix: {
          value: new Matrix4().elements,
          type: 'uniformMatrix4fv',
        },
      },
      maps: {
        u_Sampler: {
          magFilter: gl.LINEAR,
          minFilter: gl.LINEAR,
        }
      }
    })
    const earth = new Earth(0.5, 64, 32)
    const geo = new Geo({
      data: {
        a_Position: {
          array: earth.vertices,
          size: 3
        },
        a_Pin: {
          array: earth.uv,
          size: 2
        }
      },
      index: {
        array: earth.indexes
      }
    })
    this.add(new Obj3D({ geo, mat }))
    this.draw()
    this.mat=mat
  }

}
```

3.基于之前的场景切换文件修改代码，引入组件。

```
import {
  Matrix4, PerspectiveCamera, Vector3
} from 'https://unpkg.com/three/build/three.module.js';
import OrbitControls from './lv/OrbitControls.js'
import VRFrame from './lv/VRFrame.js';
import VRPlane from './lv/VRPlane.js';
import Track from '/jsm/Track.js';
```

4.开启透明度。

```
let gl = canvas.getContext('webgl');
gl.enable(gl.BLEND);
gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA);
```

5.实例化一个场景对象，两个帧缓冲区。

```
const scene = new VRPlane({ gl })
const vrNew = new VRFrame({ gl, orbit })
const vrOld = new VRFrame({ gl, orbit })
scene.mat.setMap('u_SampOld', {
  texture: vrOld.texture
})
```

初始状态，旧VR 是没有图片的，默认画出来就是一张黑图，将其传给场景对象的u\_SampOld后，便可以实现在黑暗中渐现新VR的效果。

新VR 需要在加载到图片后，画出VR，再传给场景对象的u\_SampNew。

6.实例化时间轨对象。

```
// 是否制作补间动画
let tweenable = false
// 补间数据
let aniDt = { ratio: 0 }
// 时间轨
let track = new Track(aniDt)
track.timeLen = 1000
track.keyMap = new Map([
  ['ratio', [[0, 0], [1000, 1]]]
])
track.onEnd = () => {
  tweenable = false
}
```

7.在切换图片时：

-   开启补间动画
-   设置新、旧VR的图片
-   把旧VR的纹理对象传入u\_SampOld
-   根据VR 数据更新视点位置
-   根据VR 数据显示标记点

```
//暂存当前的VR图像
let tempImg = null
// 更新VR图像
function updateVr() {
  const image = new Image()
  image.src = curVr.imgSrc
  image.onload = function () {
    //开启补间动画
    tweenable = true
    //时间轨起始时间
    track.start = new Date()
    
    //若tempImg 不为null
    if (tempImg) {
      // 设置旧VR的图片
      vrOld.mat.setMap('u_Sampler', { image: tempImg })
      vrOld.draw()
      // 把旧VR的纹理对象传入u_SampOld
      scene.mat.setMap('u_SampOld', {
        texture: vrOld.texture
      })
    }
    //暂存当前图片
    tempImg = image
    //设置新VR的图片
    vrNew.mat.setMap('u_Sampler', { image })
    //设置相机视点
    camera.position.set(...curVr.eye)
    orbit.updateCamera()
    orbit.resetSpherical()
    //显示当前VR的标记点
    showMark()
  }
}
```

将之前 Mat.js 对象的setMap 方法做下调整。

```
setMap(key,val) {
  const obj = this.maps[key]
  val.needUpdate = true
  if (obj) {
    Object.assign(obj,val)
  } else {
    this.maps[key]=val
  }
}
```

这样，若key在maps中存在，就合并val；若不存在，就写入val。

8.连续渲染。

```
function render() {
  if (tweenable) {
    // 更新时间轨的时间
    track.update(new Date())
    // 更新场景对象的插值数据
    scene.mat.setData('u_Ratio', {
      value: aniDt.ratio
    })
  }
  // 更新投影视图矩阵
  orbit.getPvMatrix()
  // 新VR绘图
  vrNew.draw()
  // 更新场景对象的u_SampNew
  scene.mat.setMap('u_SampNew', {
    texture: vrNew.texture
  })
  //场景对象绘图
  scene.draw()
  // 更新标记点的canvas坐标
  updateMarkCp()

  requestAnimationFrame(render)
}
```

到目前为止，我们便可以实现VR场景的透明度补间动画。

接下来，我们还可以再丰富一下补间动画效果。

9.在透明度动画的基础上，再让旧VR做一个放大效果，给用户营造一个拉进视口的效果，使项目看起来更加走心。

```
precision mediump float;
uniform float u_Ratio;
uniform sampler2D u_SampNew;
uniform sampler2D u_SampOld;
varying vec2 v_Pin;
void main(){
  vec2 halfuv=vec2(0.5,0.5);
  float scale=1.0-u_Ratio*0.1;
  vec2 pin=(v_Pin-halfuv)*scale+halfuv;
  vec4 t1 = texture2D( u_SampNew, v_Pin );
  vec4 t2 = texture2D( u_SampOld, pin );
  gl_FragColor = mix(t2,t1, u_Ratio);
}
```

放大旧VR的方法有很多，可以从模型、相机、片元着色器等方面来实现。

我这里就找了个比较简单的方法，在片元着色器里放大旧VR。

在片元着色器里放大旧VR 的方法，至少有两种：

-   基于片元放大旧VR
-   基于UV放大旧VR

基于片元放大旧VR，需要在片元着色器里知道canvas画布在gl\_FragCoord 坐标系里的中心点，有点麻烦。

基于UV放大旧VR，直接基于uv 坐标系的中心点(0.5,0.5) 缩小uv坐标即可，比较简单，所以我就使用这个方法放大旧VR了。

到目前为止，VR相关的核心原理算是告一段落了。

用过原生webgl 做项目，大家可以发现一个好处，只要底子扎实，就不会畏惧各种新需求、新功能，因为你会拥有更多、更自由、更灵活的选择，从而找到问题最优的解。
