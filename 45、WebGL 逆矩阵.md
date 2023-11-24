---
created: 2023-11-24T14:46:57 (UTC +08:00)
tags: [WebGL]
source: https://juejin.cn/post/7067394244540891173
author: 李伟_Li慢慢
---

# WebGL 逆矩阵

---
源码：[github.com/buglas/webg…](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fbuglas%2Fwebgl-lesson "https://github.com/buglas/webgl-lesson")

我之前在说matrixWorldInverse 的时候说过，它是matrixWorld 的逆矩阵。

逆矩阵在图形项目的应用很广，所以咱们接下来就系统说一下逆矩阵的概念。

#### 1.逆矩阵的概念

逆矩阵就好比咱们学习除法的时候，一个实数的倒数。

如：

2的倒数是1/2。

那么，矩阵m的倒数就是1/m。

只不过，1/m不叫做矩阵m的倒数，而是叫做矩阵m的逆矩阵。

由上，我们可以推导出的一些特性。

已知：

-   矩阵m
-   矩阵n

可得：

1.矩阵与其逆矩阵的相乘结果为单位矩阵

因为：

2\*1/2=1

所以：

m\*1/m=单位矩阵

2.矩阵m除以矩阵n就等于矩阵m乘以矩阵n的逆矩阵

因为：

3/2=3\*1/2

所以：

m/n=m\*1/n

#### 2.矩阵转逆矩阵

对于矩阵转逆矩阵的方法，我不说复杂了，就举几个简单例子给大家理解其原理。

1.  位移矩阵的逆矩阵是取位移因子的相反数

```
const m=new Matrix4()
m.elements=[
  1,0,0,0,
  0,1,0,0,
  0,0,1,0,
  4,5,6,1,
]
console.log(m.invert().elements);
//打印结果
[
  1,0,0,0,
  0,1,0,0,
  0,0,1,0,
  -4,-5,-6,1,
]
```

2.  缩放矩阵的逆矩阵是取缩放因子的倒数

```
{
  const m=new Matrix4()
  m.elements=[
    2,0,0,0,
    0,4,0,0,
    0,0,8,0,
    0,0,0,1,
  ]
  console.log(m.invert().elements);
}
//打印结果
[
  0.5, 0, 0, 0, 
  0, 0.25, 0, 0,
  0, 0, 0.125, 
  0, 0, 0, 0, 1
]
```

3.旋转矩阵的逆矩阵是基于旋转弧度反向旋转

```
{
  const ang=30*Math.PI/180
  const c=Math.cos(ang)
  const s=Math.sin(ang)
  const m=new Matrix4()
  m.elements=[
    c,s,0,0,
    -s,c,0,0,
    0,0,1,0,
    0,0,0,1,
  ]
  console.log(m.invert().elements);
}
//打印结果
[
  0.866, -0.45, 0, 0, 
  0.45, 0.866, 0, 0, 
  0, 0, 1, 0, 
  0, 0, 0, 1
]
```

关于即旋转又缩放还位移的复合矩阵，也是按照类似的原理转逆矩阵的，只不过过程要更复杂一些。

复合矩阵转逆矩阵的方法我就先不说了，等走完整个课程我再给你大家详解。

若有同学对其感兴趣，可以先自己看一下three.js的Matrix4对象的invert() 方法。

```
invert() {
  // based on http://www.euclideanspace.com/maths/algebra/matrix/functions/inverse/fourD/index.htm
  const te = this.elements,

        n11 = te[ 0 ], n21 = te[ 1 ], n31 = te[ 2 ], n41 = te[ 3 ],
        n12 = te[ 4 ], n22 = te[ 5 ], n32 = te[ 6 ], n42 = te[ 7 ],
        n13 = te[ 8 ], n23 = te[ 9 ], n33 = te[ 10 ], n43 = te[ 11 ],
        n14 = te[ 12 ], n24 = te[ 13 ], n34 = te[ 14 ], n44 = te[ 15 ],

        t11 = n23 * n34 * n42 - n24 * n33 * n42 + n24 * n32 * n43 - n22 * n34 * n43 - n23 * n32 * n44 + n22 * n33 * n44,
        t12 = n14 * n33 * n42 - n13 * n34 * n42 - n14 * n32 * n43 + n12 * n34 * n43 + n13 * n32 * n44 - n12 * n33 * n44,
        t13 = n13 * n24 * n42 - n14 * n23 * n42 + n14 * n22 * n43 - n12 * n24 * n43 - n13 * n22 * n44 + n12 * n23 * n44,
        t14 = n14 * n23 * n32 - n13 * n24 * n32 - n14 * n22 * n33 + n12 * n24 * n33 + n13 * n22 * n34 - n12 * n23 * n34;

  const det = n11 * t11 + n21 * t12 + n31 * t13 + n41 * t14;

  if ( det === 0 ) return this.set( 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0 );

  const detInv = 1 / det;

  te[ 0 ] = t11 * detInv;
  te[ 1 ] = ( n24 * n33 * n41 - n23 * n34 * n41 - n24 * n31 * n43 + n21 * n34 * n43 + n23 * n31 * n44 - n21 * n33 * n44 ) * detInv;
  te[ 2 ] = ( n22 * n34 * n41 - n24 * n32 * n41 + n24 * n31 * n42 - n21 * n34 * n42 - n22 * n31 * n44 + n21 * n32 * n44 ) * detInv;
  te[ 3 ] = ( n23 * n32 * n41 - n22 * n33 * n41 - n23 * n31 * n42 + n21 * n33 * n42 + n22 * n31 * n43 - n21 * n32 * n43 ) * detInv;

  te[ 4 ] = t12 * detInv;
  te[ 5 ] = ( n13 * n34 * n41 - n14 * n33 * n41 + n14 * n31 * n43 - n11 * n34 * n43 - n13 * n31 * n44 + n11 * n33 * n44 ) * detInv;
  te[ 6 ] = ( n14 * n32 * n41 - n12 * n34 * n41 - n14 * n31 * n42 + n11 * n34 * n42 + n12 * n31 * n44 - n11 * n32 * n44 ) * detInv;
  te[ 7 ] = ( n12 * n33 * n41 - n13 * n32 * n41 + n13 * n31 * n42 - n11 * n33 * n42 - n12 * n31 * n43 + n11 * n32 * n43 ) * detInv;

  te[ 8 ] = t13 * detInv;
  te[ 9 ] = ( n14 * n23 * n41 - n13 * n24 * n41 - n14 * n21 * n43 + n11 * n24 * n43 + n13 * n21 * n44 - n11 * n23 * n44 ) * detInv;
  te[ 10 ] = ( n12 * n24 * n41 - n14 * n22 * n41 + n14 * n21 * n42 - n11 * n24 * n42 - n12 * n21 * n44 + n11 * n22 * n44 ) * detInv;
  te[ 11 ] = ( n13 * n22 * n41 - n12 * n23 * n41 - n13 * n21 * n42 + n11 * n23 * n42 + n12 * n21 * n43 - n11 * n22 * n43 ) * detInv;

  te[ 12 ] = t14 * detInv;
  te[ 13 ] = ( n13 * n24 * n31 - n14 * n23 * n31 + n14 * n21 * n33 - n11 * n24 * n33 - n13 * n21 * n34 + n11 * n23 * n34 ) * detInv;
  te[ 14 ] = ( n14 * n22 * n31 - n12 * n24 * n31 - n14 * n21 * n32 + n11 * n24 * n32 + n12 * n21 * n34 - n11 * n22 * n34 ) * detInv;
  te[ 15 ] = ( n12 * n23 * n31 - n13 * n22 * n31 + n13 * n21 * n32 - n11 * n23 * n32 - n12 * n21 * n33 + n11 * n22 * n33 ) * detInv;

  return this;

}
```
