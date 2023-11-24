---
created: 2023-11-24T14:43:16 (UTC +08:00)
tags: [WebGL]
source: https://juejin.cn/post/7067388539314372644
author: 李伟_Li慢慢
---

# WebGL 极坐标渐变

---
源码：[github.com/buglas/webg…](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fbuglas%2Fwebgl-lesson "https://github.com/buglas/webgl-lesson")

我们将之前径向渐变的代码改改，还可以实现极坐标渐变。

![image-20210525223153491](assets/61f9fd2d3cff40f490f8de2cf0d8d9b0tplv-k3u1fbpfcp-zoom-in-crop-mark1512000.webp)

1.在矩形面中，注释终点

```
// u_End: {
//   type: 'uniform2fv',
//   value: [canvas.width, canvas.height]
// },
```

2.在片元着色器中也做相应调整。

```
//起始位
uniform vec2 u_Start;
//结束位
//uniform vec2 u_End;
//终点减起点向量
//vec2 se=u_End-u_Start;
//四阶矩阵
uniform mat4 u_ColorStops;
//长度
//float seLen=length(se);
//单位向量
//vec2 se1=normalize(se);
//一圈的弧度
float pi2=radians(360.0);
```

3.修改获取片元颜色的方法，基于极角取ratio比值。

```
//获取片元颜色
vec4 getColor(vec4 colors[8],float ratios[8]){
    //片元颜色
    vec4 color=vec4(1);
    //当前片元减起始片元的向量
    vec2 sf=vec2(gl_FragCoord)-u_Start;
    //当前片元在se上的投影长度
    //float fsLen=clamp(dot(sf,se1),0.0,seLen);
    //长度比
    //float ratio=clamp(fsLen/seLen,ratios[0],ratios[8-1]);
    //向量方向
    float dir=atan(sf.y,sf.x);
    if(dir<0.0){
        dir+=pi2;
    }
    float ratio=dir/pi2;
    ……
}
```
