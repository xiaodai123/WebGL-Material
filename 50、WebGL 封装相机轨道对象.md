---
created: 2023-11-24T14:48:29 (UTC +08:00)
tags: [WebGL]
source: https://juejin.cn/post/7067401279055593509
author: 李伟_Li慢慢
---

# WebGL 封装相机轨道对象

---
源码：[github.com/buglas/webg…](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fbuglas%2Fwebgl-lesson "https://github.com/buglas/webgl-lesson")

接下来我会参考three.js 里的OrbitControls对象，封装一个轨道控制器出来。

至于轨迹球的封装，我就留给大家来练手了，可以参考着TrackballControls 来实现。

首先我们先对之前所学的正交相机轨道和透视相机轨道做一个差异的对比：

-   旋转轨道
    
    -   正交相机和透视相机的旋转轨道都是一样的，都是使用球坐标，让相机视点围绕目标点做的旋转。
-   位移轨道
    
    -   正交相机的位移轨道是鼠标从canvas画布到近裁剪面，再到世界坐标系的位移量的转换，最后这个位移量会同时作用于相机的目标点和视点。
    -   透视相机的位移轨道是鼠标从canvas画布到目标平面，再到世界坐标系的位移量的转换，最后这个位移量会同时作用于相机的目标点和视点。
-   缩放轨道
    
    -   正交相机的缩放轨道是通过对其可视区域的宽高尺寸的缩放实现的。
    -   透视相机的缩放轨道是通过相机视点在视线上的位移实现的。

整体的原理就是这样，接下来就可以封装轨道控制器了。

### 1-轨道控制器的封装

我这里没有原封不动抄袭there.js，我参考其整体原理，做了一下简化和取舍。

大家若有更好的想法，也可以自己封装一个出来，然后努力超越three.js。

整体代码如下：

```
import {
    Matrix4, Vector2, Vector3,Spherical
} from 'https://unpkg.com/three/build/three.module.js';

const pi2 = Math.PI * 2
const pvMatrix=new Matrix4()

const defAttr = () => ({
    camera: null,
    dom: null,
    target: new Vector3(),
    mouseButtons:new Map([
        [0, 'rotate'],
        [2, 'pan'],
    ]),
    state: 'none',
    dragStart: new Vector2(),
    dragEnd: new Vector2(),
    panOffset: new Vector3(),
    screenSpacePanning: true,
    zoomScale: 0.95,
    spherical:new Spherical(),
    rotateDir: 'xy',
})

export default class OrbitControls{
    constructor(attr){
        Object.assign(this, defAttr(), attr)
        this.updateSpherical()
        this.update()
    }
    updateSpherical() {
        const {spherical,camera,target}=this
        spherical.setFromVector3(
            camera.position.clone().sub(target)
        )
    }
    pointerdown({ clientX, clientY,button }) {
        const {dragStart,mouseButtons}=this
        dragStart.set(clientX, clientY)
        this.state = mouseButtons.get(button)
    }
    pointermove({ clientX, clientY }) {
        const { dragStart, dragEnd, state, camera: { type } } = this
        dragEnd.set(clientX, clientY)
        switch (state) {
            case 'pan':
                this[`pan${type}`] (dragEnd.clone().sub(dragStart))
                break
            case 'rotate':
                this.rotate(dragEnd.clone().sub(dragStart))
                break
        }
        dragStart.copy(dragEnd)
    }
    pointerup() {
        this.state = 'none'
    }
    wheel({ deltaY }) {
        const { zoomScale, camera: { type } } = this
        let scale=deltaY < 0?zoomScale:1 / zoomScale
        this[`dolly${type}`] (scale)
        this.update()
    }
    dollyPerspectiveCamera(dollyScale) {
        this.spherical.radius *= dollyScale
    }
    dollyOrthographicCamera(dollyScale) {
        const {camera}=this
        camera.zoom *= dollyScale
        camera.updateProjectionMatrix()
    }
    panPerspectiveCamera({ x, y }) {
        const {
            camera: { matrix, position, fov,up },
            dom: { clientHeight },
            panOffset,screenSpacePanning,target
        } = this

        //视线长度：相机视点到目标点的距离
        const sightLen = position.clone().sub(target).length()
        //视椎体垂直夹角的一半(弧度)
        const halfFov = fov * Math.PI / 360
        //目标平面的高度
        const targetHeight = sightLen * Math.tan(halfFov) * 2
        //目标平面与画布的高度比
        const ratio = targetHeight / clientHeight
        //画布位移量转目标平面位移量
        const distanceLeft = x * ratio
        const distanceUp = y * ratio

        //相机平移方向
        //鼠标水平运动时，按照相机本地坐标的x轴平移相机
        const mx = new Vector3().setFromMatrixColumn(matrix, 0)
        //鼠标水平运动时，按照相机本地坐标的y轴，或者-z轴平移相机
        const myOrz = new Vector3()
        if (screenSpacePanning) {
            //y轴，正交相机中默认
            myOrz.setFromMatrixColumn(matrix, 1)
        } else {
            //-z轴，透视相机中默认
            myOrz.crossVectors(up, mx)
        }
        //目标平面位移量转世界坐标
        const vx = mx.clone().multiplyScalar(-distanceLeft)
        const vy = myOrz.clone().multiplyScalar(distanceUp)
        panOffset.copy(vx.add(vy))

        this.update()
    }

    panOrthographicCamera({ x, y }) {
        const {
            camera: { right, left, top, bottom, matrix, up },
            dom: { clientWidth, clientHeight },
            panOffset,screenSpacePanning
        } = this

        const cameraW = right - left
        const cameraH = top - bottom
        const ratioX = x / clientWidth
        const ratioY = y / clientHeight
        const distanceLeft = ratioX * cameraW
        const distanceUp = ratioY * cameraH
        const mx = new Vector3().setFromMatrixColumn(matrix, 0)
        const vx = mx.clone().multiplyScalar(-distanceLeft)
        const vy = new Vector3()
        if (screenSpacePanning) {
            vy.setFromMatrixColumn(matrix, 1)
        } else {
            vy.crossVectors(up, mx)
        }
        vy.multiplyScalar(distanceUp)
        panOffset.copy(vx.add(vy))
        this.update()
    }


    rotate({ x, y }) {
        const {
            dom: { clientHeight },
            spherical, rotateDir,
        } = this
        const deltaT = pi2 * x / clientHeight
        const deltaP = pi2 * y / clientHeight
        if (rotateDir.includes('x')) {
            spherical.theta -= deltaT
        }
        if (rotateDir.includes('y')) {
            const phi = spherical.phi - deltaP
            spherical.phi = Math.min(
                Math.PI * 0.99999999,
                Math.max(0.00000001, phi)
            )
        }
        this.update()
    }

    update() {
        const {camera,target,spherical,panOffset} = this
        //基于平移量平移相机
        target.add(panOffset)
        camera.position.add(panOffset)

        //基于球坐标缩放相机
        const rotateOffset = new Vector3()
        .setFromSpherical(spherical)
        camera.position.copy(
            target.clone().add(rotateOffset)
        )

        //更新投影视图矩阵
        camera.lookAt(target)
        camera.updateMatrixWorld(true)

        //重置旋转量和平移量
        spherical.setFromVector3(
            camera.position.clone().sub(target)
        )
        panOffset.set(0, 0, 0)
    }

    getPvMatrix() {
        const { camera: { projectionMatrix, matrixWorldInverse } } = this
        return pvMatrix.multiplyMatrices(
            projectionMatrix,
            matrixWorldInverse,
        )
    }

}
```

接下来咱们将其实例化一下看看。

### 2-轨道控制器的实例化

OrbitControls对象就像three.js 里的样，可以自动根据相机类型去做相应的轨道变换。

```
/* 透视相机 */
/* 
    const eye = new Vector3(0, 0.5, 1)
    const target = new Vector3(0, 0, -2.5)
    const up = new Vector3(0, 1, 0)
    const [fov, aspect, near, far] = [
      45,
      canvas.width / canvas.height,
      1,
      20
    ]
    const camera = new PerspectiveCamera(fov, aspect, near, far)
    camera.position.copy(eye) 
*/

/* 正交相机 */
const halfH = 2
const ratio = canvas.width / canvas.height
const halfW = halfH * ratio
const [left, right, top, bottom, near, far] = [
    -halfW, halfW, halfH, -halfH, 1, 8
]
const eye = new Vector3(1, 1, 2)
const target = new Vector3(0, 0, -3)
const up = new Vector3(0, 1, 0)

const camera = new OrthographicCamera(
    left, right, top, bottom, near, far
)
camera.position.copy(eye)

const pvMatrix = new Matrix4()

……

/* 实例化轨道控制器 */
const orbit = new OrbitControls({
    camera, 
    target,
    dom: canvas,
})
pvMatrix.copy(orbit.getPvMatrix())
render()

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
    pvMatrix.copy(orbit.getPvMatrix())
    render()
})
canvas.addEventListener('pointerup', event => {
    orbit.pointerup(event)
})

//滚轮事件
canvas.addEventListener('wheel', event => {
    orbit.wheel(event)
    pvMatrix.copy(orbit.getPvMatrix())
    render()
})
```
