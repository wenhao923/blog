# Percentage-Closer Filtering(PCF)
接着上文，本篇文章讲**软阴影**的生成。

## BRDF的阴影
在介绍**PCF**前，我们把目光聚焦于**Path tracing**。我们知道**Path tracing**基于的**BRDF**或者说是**Rendering Equation**是无比正确的。观察下图它的阴影表现
<div align=center>
<img src='GAMES202\pathtacing.png' width = "500" height = "500"/>

此图只计算了direct light的贡献
</div>

可以观察到，基于**Rendering Equation**渲染的阴影已经是**软阴影**，它是如何得到的。我们再看一遍**Path tracing**算法：
<div align=center>
<img src='https://www.notion.so/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F79ee41ad-ea81-4428-af12-192288ea4f52%2FUntitled.png?table=block&id=16af0981-6bd1-49ea-a5c1-29e19969b85f&spaceId=96c1a001-9b6c-416f-b81b-8dfad4ed30b3&width=2000&userId=&cache=v2'/>
</div>

- Ray Casting：从相机处连接fragment，发出射线，与Scene相交，打在shading point上。就是说，我们当前在**近平面**观察到的正是shading point(遍历 $\lbrace Position(p), B(p) \wedge p\in Scene \rbrace$ )。
- 对于当前正在渲染的point，我们使用**Rendering Equation**进行计算：
  
$$L(p,\omega_r)=L_e(p,\omega_o)+\displaystyle\int_{\Omega^+}L(p,\omega_i)·f_r(p,\omega_r)·(\vec{n}·\omega_i)d\omega_i$$
>最终计算point的颜色与$L(p,\omega_r)$密切相关(使用gamma矫正把radiance成颜色)

<div align=center>
<img src='GAMES202\jietu.png'/>
</div>

这两步对我们来说非常熟悉，参见上篇文章，这里查看截图：**Path tracing**的两步正是**shadow mapping**的四步。第一步完全一样(我复制粘贴的)，而所谓的二三四步则是**使用trick来逼近真实结果**(简化渲染方程)。
<!-- 所谓“**简化**”体现在**两个**方面： -->

<!-- - 使用Phong/Blinn Phong简化渲染方程
- 从渲染方程提取出$Visility(p)$，表示shading point的可见性，这个值正是shadow mapping中判断阴影第四步的0/1，不过下面我们 -->

我们再来看shading point的最终结果呢：**Path tracing**取决于渲染方程的结果，姑且可以看作是取值范围较大的连续值；而**shadow mapping**后的取值范围为$\{ 0,1 \}$。显而易见，**非零即一**表达最终的颜色要出大问题，**我们需要尽可能放大阴影空间**($\{x,x\in [0,1]\wedge x\in R \}$)。

## 走样的原因
现在来看，导致走样有两个原因：
1. 一个是上文提出的$\{ 0,1 \}$取值范围，阴影表达不正确(**表达阴影含义的误差，没有考虑半影**)
2. 一个是上篇文章提出的与分辨率相关的采样不足导致ray casting时fragment对应的shading point不正确，**其实也是一种自遮挡**

<div align=center>
<img src='GAMES202\pixel.png'/>
</div>

## Percentage-Closer Filtering(PCF)
>改善第一个误差

为了方便，我们把做完**shadow mapping**后的取值标记为$Visility(p)$

为了改变$Visility(p)$非0即1，观察**判断阴影**算法，我们抽象出公式:
$$\chi^+(x) = \begin{cases}  
1 & x >= 0 \\
0 & x < 0
\end{cases}$$
$$Visility(x,y,z)=\chi^+\{SM(x,y)-z\},p=(x,y,z)$$
>由于$Visility(p)$不是真实的物理量，所以为了达到好的渲染效果(半影)，实现从实到无的阴影渐变。

我们使用**卷积**，让$(x,y)$周围的点对$Visility(x,y,z)$产生影响：

<div align=center>
<img src='GAMES202\juanji.png'/>
</div>

如图，使用以$(x,y)$为中心，计算九宫格(或者更多)的$\chi^+(x)$,最后求平均。公式：
$$p=(x,y,z)$$
$$N(p)=(x+bias,y+bias),bias\in [-max,max]\wedge bias,max\in N^+$$
$$Visility(p)=\sum_{q\in N(p)} weight(q)·\chi^+\{SM(x,y)-z\}\tag{1}$$

渲染阴影的效果时很好的。此方法为了**反走样抗锯齿**而提出，而对生成软阴影有着巨大的帮助。
<div align=center>
<img src='GAMES202\CAR.png'/>
</div>

## Percentage closer soft shadows(PCSS)
以上公式新引入了两个量$weight(q)$和$bias$/$max$,姑且认为$weight(q)$都是1(这是图像处理的知识)，那bias如何确定？不同的bias范围让阴影或清晰或模糊，每个shading point都是不同的。我们需要了解**阴影的生成**原理
<div align=center>
<img src='GAMES202\04.PNG'/>
</div>

从图中的相似三角形得出公式：
$$W_{penumbra}=\frac{W_{light}·SM(blocker)}{penumbra.z-SM(blocker)}$$

整理我们已知的：
- shading point的位置 $penumbra:(x,y,z)$
- shadow mapping：$SM$
- 光源大小：$W_{light}$

所以我们只需要计算$blocker$的位置(或深度)：所谓**blocker depth**,GAMES202是指**Average Blocker Depth**，我们需要计算**一定范围**的blocker的平均深度。
$$W_{penumbra}=\frac{W_{light}·ABD(penumbra)}{penumbra.z-ABD(penumbra)}\tag{2}$$

>疑惑这里为什么求blocker的平均深度，从物理的角度难以理解，GAMES202中也没有给出解释或bad case。正确计算$Visility$应当计算当前shading point可以接收到的光线量，与blocker大小无关。猜测这里使用Average Blocker Depth近似了不同shading point接收的光线量。

## 计算Average Blocker Depth
只要拥有所有点的深度即可轻松计算：
$$ABD(p)=\frac{1}{Blocker(p)}·\sum_{q\in Blocker(p)}SM(q)\tag{3}$$

因此，我们又需要确定$Blocker(p)$的范围，我们可以简单地设为3x3或5x5等，也可以更加合理地查找范围，如图：
<div align=center>
<img src='GAMES202\blocker.png'/>
</div>

$$Blocker(p)\subset Light(p)\subset SM$$
- $Blocker(p)$是我们需要采样的范围
- $Light(p)$是光源对shading point的影响范围
- $SM$是shadow map
>这里指的范围是从光源出进行**光栅化**时建立的frustum的**近平面**处，此近平面与shadow map大小相同

我们从光源对shading point的影响范围中采样到了遮挡住光源的blocker(也是全部的blocker)，这样便获得了$Blocker(p)$

## 总结
PCSS的流程可分为三步：
1. Blocker search： 公式$(3)$
2. Penumbra estimation：公式$(2)$
3. PCF：公式$(1)$

$$ABD(p)\rightarrow W_p\rightarrow bias/size\rightarrow Visilty(p)$$

## Reference
[](https://www.bilibili.com/video/BV11a4y1s7Cy)

[](https://zhuanlan.zhihu.com/p/359377010)

