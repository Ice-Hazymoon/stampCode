# stampCode
> 一个触摸印章技术的实现方案

## [Demo](https://browniu.github.io/stampCode/)

目前建议在Pc Chrome测试，事件监听的是`mouse`类

[![demo](./static/stampCode2.gif)](https://browniu.github.io/stampCode/) 

## 简述

### 需求分析
许多开发者大会/漫展/游戏展，都会有集印章的线下推广活动。就是在入口处给你发张DM单，上面会有一些空格位置，需要你到指定的展摊集印。到摊位前完
成一些简单任务（加个微信或关注个公众号什么的）就会有小姐姐给你拿钢印盖个章或者签个名，集满或达到一定数量就可以兑换礼品。

![展会](./static/zh.jpg)

### 传统印章方案

在我看来这种传统的印章签到方式本身存在很多问题。

* 制作成本高。首先需要印刷纸质的DM传单来接受盖印，又要给每个摊位制作专用章，还要有一个工作人员值班负责这件事（无法离开摊位）
* 效率很低，盖印要很用力的压在一个平整的桌面上，盖印盖多了要沾印泥，这一系列动作耗时耗力，在火爆的情况下，很容易大排长龙，造成用户流失
* 用户体验不佳。作为吃瓜群众的我要随身携带这张纸。还要不停掏出来折回去，保存麻烦还很容易弄丢。而且刚盖完上面没干的墨水还会弄脏我新买的衣服

![传统盖章](./static/gz.jpg)

### H5交互印章

身为一个前端工程师，我就想能否通过移动端的传感器技术去优化这个流程。

这个需求点分析透彻就是一个 **线下签到** 问题。最简单得做法就是扫码吧。工作人员扫你的身份识别二维码就搞定了，简单轻松。既可以杜绝虚假签到，
用户需要做的事也不多。但似乎少了点什么？那就是仪式感。采集是人类的原始爱好，印章方式虽然麻烦但是很有仪式感，让我们想起了石器时代的采集的乐趣。
扫码是简单，简单到没什么参与感就结束了这个流程。

![印章实体](static/touchStamp.jpg)

能不能更好玩一点？想象一下，我拿个印章在你的手机上盖了一下，就像在纸上那么做一样，H5上浮现出了一个动态的盖印，甚至跳出一个相关的游戏角色的彩
蛋，我觉得这种体验既包含收集的乐趣，又省去很多麻烦和成本，H5上也能承载更多的互动方式，让用户的参与度更高。所以这件事还是值得讨论的。

### 逻辑分析
这个H5盖印识别的技术让我想到了屏下指纹识别，但是又不需要那么高的精度。所以通过多点触控应该就能实现。

#### 模拟多点触控
H5印章实体大概就是有一些圆点凸起的方块，每个凸起的地方使用导电橡胶材料制作（类似电容笔），这样平整放置在屏幕上时就能在屏幕上形成多个触控点，模拟
手指多点触控的交互方式。

![印章实体](static/stamp.jpg)

#### 获取点位信息
通过BOM对象的`touch`事件就能够同时获取到屏幕上的多个触控点的坐标信息，拿到一个如 `[[x1,y1],[x2,y2],[x3,y3]]` 这样的二维数组，拿这个刚
采集到的数据更数据库保存的一个固定的同格式数据进行比对，得出他们的相似程度。

#### 忽略方向和尺寸
在给用户盖印的过程中（采集新的点位信息），会存在一些不确定性

* 因为需要通过手机屏幕采集触摸点，而不同型号的手机屏幕尺寸/分辨率都存在差异，同一个印章盖上去，每次拿到的坐标数都有可能不同
* 每次盖章时，章的方向和手机的方向可能会有所不同

基于以上几点，我们计算相似性的算法就要忽略点位整体的方向和绝对尺寸。也就是证相似而不是全等。

![相似多边形](static/simi3.jpg)

> 如果两个边数相同的多边形的对应角相等，对应边成比例，这两个或多个多边形叫做相似多边形

我们采集到的点位虽然不一定能够形成凸多边形，但是可以参考证明相似多边形这个思路，对比边的比例。距离可以不固定，就是各点间距离（即多边形的边）的比例是固定的。

#### 边的比例

假设印章上只有三点位，通过`touch`事件采集到的三个触摸点形成一个三角形，通过计算两点间的距离获取到三角形的三边长D1、D2、D3。

```JavaScript
// 两点间距离
const distance = (x0, y0, x1, y1) => Math.hypot(x1 - x0, y1 - y0);
```

如果完全按照相似多边形定理去证明，除了要比较各个对应边的比例，还要计算出两条边之间的夹角，这太麻烦了。能不能换一个思路？先找到最小边，然后求出其他边与最小边
的比例，升序重排后的数据不就可以过滤掉方向和尺寸的特征了。

```JavaScript
// 假设D1最小 求其余边与最小边的比例
const R1 = D2/D1
const R2 = D3/D1
const resultFromTouch = [R1,R2].sort()
```

从手机屏幕拿到一个数组 `resultFromTouch=[R1,R2]`，同样的我们可以从用来验证的印章点位信息中也获取到相同结构的数据`resultFromStamp=[Ra,Rb]`。
理想情况下，在用户手机上采集点位时都刚好拿到的是触摸点中心点像素的坐标信息，那么`resultFromTouch`与`resultFromStamp`对应位置的值应该
能全等。然而实物印章的点位并不是一像素，而是一个面，并且这个面不能过小。所以要设置一个合理的容差。

#### 余弦相似性

那现在就变成了比较两个一维数组（向量）相似性的问题。我想要的最终结果一个是`0-100`的分数，`0`表示完全不同，`100`表示完全相同。这样可以通过分
数来设置容差，超过90分就判定通过验证。

> 余弦相似度是用向量空间中两个向量夹角的余弦值作为衡量两个个体间差异的大小。余弦值越接近1，就表明夹角越接近0度，也就是两个向量越相似。

这里使用余弦相关性的计算公式拿到这个匹配分数
![余弦相似度公式](./static/yxxsd.png)

```JavaScript
async function compute(x, y) {
    x = tf.tensor1d(x);
    y = tf.tensor1d(y);
    const p1 = tf.sqrt(x.mul(x).sum());
    const p2 = tf.sqrt(y.mul(y).sum());
    let p12 = x.mul(y).sum();
    let score = p12.div(p1.mul(p2));
    score = ((await score.data())[0] - 0.9) * 10;
    return score
}
```

#### 差值相关性（自创）

除了直接使用余弦相似度，其实还可以尝试其他途径计算向量的相关性。比如我原创的一个算法 **差值相关性**。如果总分为100，这个数组的每一位对总分
的影响程度就是`总分/数组长度`，也就是`R1`与`Ra`之间的差异对总分的影响在`0-50`分之间。`abs.(R1-Ra)`与这个分数负相关。

```JavaScript
const gap = abs.(R1-Ra)
```

极限情况下，`gap=0`时，没有差异得50分,`gap=gamMax`时完全不同得0分，最后求和得出总分。

```JavaScript
const partScope = 50 * (1 - gap / gapMax);
```

那么问题来了`gapMax`即这个最大值怎么求？我不知道啊（🤦‍️）。不过据我手动观察这个值大部分情况90%情况下介于`0-10`之间，所以暂时假设`gapMax=10`。
得到的`scope`就是能用`0-100`的数值表示两个向量相似度的值。

整个思路大致如此，也许有更高明的算法去解决整个问题，或者是优化其中的一部分计算，欢迎讨论。

## 方法总结

### 两点距离
```JavaScript
const distance = (x0, y0, x1, y1) => Math.hypot(x1 - x0, y1 - y0);
```

### 指定坐标范围的点
```JavaScript
const randomPosInRange = (min, max) => Array.from({length: 2}, () => Math.floor(Math.random() * (max - min + 1)) + min);
```

### 数组中的最小值
```JavaScript
const minInArray = (array) => array.sort((a, b) => a - b)[0];
```

### 四舍五入指定位
```JavaScript
const round = (n, decimals = 0) => Number(`${Math.round(`${n}e${decimals}`)}e-${decimals}`);
```

### 向量余弦相似
```JavaScript
async function compute(x, y) {
    x = tf.tensor1d(x);
    y = tf.tensor1d(y);
    const p1 = tf.sqrt(x.mul(x).sum());
    const p2 = tf.sqrt(y.mul(y).sum());
    let p12 = x.mul(y).sum();
    let score = p12.div(p1.mul(p2));
    score = ((await score.data())[0] - 0.9) * 10;
    return score
}
```

### 差值相关
```JavaScript
const approximateMatchArray2 = (arr1, arr2) => {
    arr1 = arr1.sort();
    arr2 = arr2.sort();
    let scope = eval(arr1.map((item, i) => {
        const reduce = Math.abs(arr1[i] - arr2[i]);
        return (100 / arr1.length) * (1 - reduce / 10);
    }).join('+'));
    scope = scope * (scope / (scope < 90 ? 125 : 105));
    return scope <= 100 ? scope : 0
};
```

## 运行
```bash
cd src && serve
```
