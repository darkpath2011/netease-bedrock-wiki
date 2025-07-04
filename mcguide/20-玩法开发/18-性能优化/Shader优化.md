---
front: https://nie.res.netease.com/r/pic/20210728/5a263f49-e1f3-4a9a-96b3-307b11848590.png
hard: 进阶
time: 30分钟
---

# Shader优化

## 前言

为了做出更加炫酷的效果，我们往往会自定义材质，然后定义自己的Shader，或者直接重写替换原版Shader，特别是目前的很多光影MOD，基本都需要对Shader进行修改。Shader与GPU性能密切相关，Shader写得好能让配置不是很高的玩家也能流畅体验炫酷的效果。写得不好，则可能导致高端机也很卡。

## 尽量少用 if else 条件语句

GPU是并行处理逻辑的，采用SIMD（单指令多数据）结构, 同一段代码，同一时刻会被多个GPU的处理单元同时处理，这段代码的执行耗时取决于会被执行到的时间最长的代码。为了充分发挥GPU的并行性，我们尽量少用条件分支逻辑，让所有GPU处理单元都执行相同的代码。

if else的写法如果底下逻辑不多，大部分可以合并，则常常可以改用step()函数进行优化

step(a,b)的功能为：
```glsl
若b >= a则返回1，否则返回0
```

所以如果有这么一个写法：
```glsl
if(r >= 0.5)
{
    r = 0.6;
}else{
    r = 0.4;
}
```

则应该写成：
```glsl
r = 0.4 + step(0.5, r) * (0.6 - 0.4);
```

逻辑上相当于若r >= 0.5 ，则:
```glsl
r = 0.4 + 1 * 0.2 = 0.6;
```
否则r < 0.5，则:
```glsl
r = 0.4 + 0 * 0.2 = 0.4;
```
由此即可消去if else语句。


- 错误写法:
(if else大量使用)
```python
    // 简单的卡通着色例子，把连续的颜色值映射到几个特殊的离散的值上面
    //根据传入的颜色值取得一个新的颜色值, 这里为了展示简单，我们用仅用一个通道进行举例
    void main()
    {
        color.r = getNewRedColor(color.r);
        ...(省略无关代码)
    }
    float getNewRedColor(float r)
    {
        float newR;
        if(r >= 0.6)
        {
            newR = 0.8;
        }else if(r >= 0.3)
        {
            newR = 0.5;
        }else{
            newR = 0.1;
        }
        return newR;
    }
```

- 正确写法:
(分开每帧只处理5个)
```python
    // 简单的卡通着色例子，把连续的颜色值映射到几个特殊的离散的值上面
    //根据传入的颜色值取得一个新的颜色值, 这里为了展示简单，我们用仅用一个通道进行举例
    void main()
    {
        color.r = getNewRedColor(color.r);
        ...(省略无关代码)
    }
    float getNewRedColor(float r)
    {
        float newR = 0.0;
        newR = newR + step(0.6, r) * 0.8;
        newR = newR + step(0.3, r) * step(r, 0.6) * 0.5;
        newR = newR + step(r, 0.3) * 0.1;
        return newR;
    }
```

## 循环语句

for, while这类的循环语句内部实现其实也会有条件判断if else语句，并行性比较低，所以如果可以不用尽量不用，但这并不是让大家去复制粘贴多少次代码，这没有意义，则是尽量从逻辑上避免循环逻辑的出现，如果实在需要使用，则建议循环体内不要做太多耗性能的操作。
除此之外，循环变量一定要记得初始化！变量的初始值在不同设备上有时候是不一样的, 比如int的初始值并不一定在所有设备上都是0。

- 错误写法:
(循环变量i没有初始化)
```python
    for(int i; i < 5; i ++)
    {
        func();
    }
```
在一些设备上i的值会被初始化为0，循环5次没有问题。但在一些设备上，i的默认值可能是一个没有意义的数，甚至是负数！例如是−2147483648，上面循环则会循环超级多次，玩家会直接卡到动不了。

- 正确写法:
(分开每帧只处理5个)
```python
    //这里i一定要给一个默认值
    for(int i = 0; i < 5; i ++)
    {
        func();
    }
```

## 精美贴图开关

开关在游戏中的位置：设置->视频->精美贴图
开发者可根据玩家是否开启精美贴图执行不一样的shader逻辑。这里需要声明两个文件，materials/sad.json 和 materials/fancy.json。我们先看下原版中两个文件的内容：
```python
    sad.json:
    [
        {"path":"materials/sad.material"},
        {"path":"materials/entity.material"},
        {"path":"materials/terrain.material"},
        {"path":"materials/portal.material"},
        {"path":"materials/barrier.material"},
        {"path":"materials/wireframe.material"}
    ]

    fancy.json:
    [
        {"path":"materials/fancy.material", "+defines":["FANCY"]},
        {"path":"materials/entity.material", "+defines":["FANCY"]},
        {"path":"materials/terrain.material", "+defines":["FANCY"]},
        {"path":"materials/hologram.material"},
        {"path":"materials/portal.material", "+defines":["FANCY"]},
        {"path":"materials/barrier.material"},
        {"path":"materials/wireframe.material"}
    ]
```

开启精美贴图的时候会加载下面的材质，不开启的话加载上面的材质。我们用一个材质进行举例，比如sad和fancy中都有的terrain.material材质，fancy中不同在于额外定义了FANCY字段，则在Shader中可以这样写：
```python
    void main()
    {
#ifdef FANCY
    //这里可以做更多的逻辑，渲染更好的效果
    renderBeautiful();
#else
    // 这里是关闭了精美贴图，这里不应该执行过多逻辑，只需要提供简单的显示效果就可以了
    renderSimple();
#endif
    }
```

## 降低精度

通常来说，我们写的shader不需要过于关注精度，因为大部分情况下性能瓶颈不在这里，但一些过于复杂，大部分玩家都说卡的MOD，建议可以考虑在精度方面做一些优化。

shader中变量精度越低，GPU运算越快，精度分为3档，关键字分别为：
```glsl
低：lowp
中：mediump
高：highp
```

int（整数）：建议256内的整数使用lowp, 1024内的整数使用mediump，其它情况则使用highp
float(浮点)：建议256内的浮点数使用lowp, 16384内的浮点数使用mediump，其它情况则使用highp

默认精度:
```glsl
顶点着色器中float, int均为highp
像素着色器中int为mediump，float根据设备不同无默认精度
```
声明方法例子(直接把精度关键字加在变量类型前面)：
```glsl
lowp float color
```

## 移除无用变量与逻辑

部分开发者编写Shader的时候可能会偷下懒，从自己以前写的代码里面复制粘贴过来，但这部分代码可能有很多逻辑或者变量都没有用上，导致大量的运算是无意义浪费性能的，需要去掉。

- 错误写法:
(包含大量无用逻辑)
```python
    void main()
    {
        //这里声明多个变量，又或许是从其它地方复制过来，具体值先省略
        A = ...;
        B = ...;
        C = ...;

        //这里只用了变量A；B和C都没有用到
        DoSomeThingWithA(A);
        return;
    }
```

- 正确写法:
(删除无用逻辑)
```python
    void main()
    {
        //只留下A，把其它没用上的都删除掉
        A = ...;        

        //这里只用了变量A；B和C都没有用到
        DoSomeThingWithA(A);
        return;
    }
```

## 分级做多个MOD版本

开发者可根据Shader复杂度上架不同版本，举个例子，例如可以有“网易光影低配版”， “网易光影高配版”，玩家看名字就大概知道对性能有不同的要求了，让玩家下载时自行选择。