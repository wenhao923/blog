# Shadow Mapping

在GAMES101的实验七:Path Tracing中，我们已经实现了全局的效果。但是在**光栅化**中，阴影的实现是需要依靠一些奇技淫巧。例如基础且重要的"**Shadow Mapping**"。


**原理**：有两个事件 $A,B$ 被定义如下

- $A(p)$：$p$ 可以被**光源**照到
- $\bar{A}(p)$：$p$ **不**可以被**光源**照到
- $B(p)$：$p$ 可以被**相机**照到
- $\bar{B}(p)$：$p$ **不**可以被**相机**照到

那么我们可以观察到的阴影可以这样表示: $\bar{A}(p) \cap B(p),p \in Scene$

>此外，仍有 $A(p) \cap B(p)$ 表示我们可以观察到的非阴影；至于 $\bar{B}(p)$ 则显得无关紧要，因为我们看不见，所以大概率在裁剪阶段就被**剔除**了

**算法**：经过上述事件转换后，我们只需要考虑当前渲染的点是否被光源照到。由于**光源位置**一般是**固定**的，所以我们先将光源能够照到的场景保存下(即**shadow map**)，然后对观察到的所有点判断是否被照到(用一个loop)。
> $\lbrace Position(p), B(p) \wedge p\in Scene \rbrace$在裁剪阶段后已经获取到，不再赘述

1. **获取 $\lbrace Depth(p), A(p) \wedge p\in Scene \rbrace$ ：保存场景中所有能够被照的点的深度(制作shadow map)**
2. **遍历 $\lbrace p,B(p) \wedge p\in Scene \rbrace$，判断 $p$ 是否满足 $B(p)$**

<div align=center>
<img src='https://github.com/wenhao923/blog/blob/main/pictures/GAMES202/01.PNG'/>
</div>

### 制作shadow map

保存场景中所有能够被照的点的**深度信息**(z值)。我们的方法如下:
- 在光源位置，模拟放置一个camera：对场景object进行**坐标变换**；**设置**near,far,fov的参数
- 对光源这个“相机”看到的场景**光栅化**，那么我们得到了一张光源视角的渲染图$SM$(shadowmap)，我们不需要每个像素的RGB信息，我们只需要每个像素的深度。这样就完成了算法的**第一步**。
> 现在拥有了对应像素的对应深度 $SM(x,y)$
<div align=center>
<img src='https://github.com/wenhao923/blog/blob/main/pictures/GAMES202/shadow_map.png'/>
</div>

### 判断阴影
我们回到真正的相机处，开始渲染最终的相片。
- Ray Casting：从相机处连接fragment，发出射线，与Scene相交，打在shading point上。就是说，我们当前在**近平面**观察到的正是shading point(遍历 $\lbrace Position(p), B(p) \wedge p\in Scene \rbrace$ )。
- 我们挑选了被相机照到的point，接下来我们需要计算point是否被光源照到：使用光源“相机”的 $MVP$ 矩阵作用于 $Position(p)$，得到point在光源“相机”的渲染图的坐标$(x,y)$和深度$z$。
- 计算$z - SM(x,y)$，若point的深度大于shadow map保存的深度，说明光源被**挡住**了，point不能被照到，处在阴影中！
- 如果point处在阴影，我们简单的把**判断阴影**中的第一步中的fragment颜色设为0，表示阴影。这就完成了算法的**第二步**。
> 光源被挡住和简单设置为0值得注意。在Soft shadow中会对这些地方更进一步地探索。

<div align=center>
<img src='https://github.com/wenhao923/blog/blob/main/pictures/GAMES202/scene.png'/>
</div>

## 问题
算法想要表达想法是件很困难的事，许多的语义的转换是**非等价**的。无论是有偏的还是无偏的算法，都会因为方法误差或者离散化在计算机中导致bad case。

接下来是Shadow Mapping算法的两个bad case:
- Self occlusion(自遮挡)
- Aliasing(走样)

### 走样
算法“判断阴影”是在shading point的阶段，与point着色直接相关。所以在这一步研究。
<div align=center>
<img src='https://github.com/wenhao923/blog/blob/main/pictures/GAMES202/frustum.png'/>
</div>

在做第一步Ray Casting时，我们从相机处连接fragment。两点确定一条直线，相机的三维坐标位置无疑时固定的，但fragment的三维坐标是什么呢？如图可见，其实观察平面(射线的另一个点)就是**近平面**，所有点的深度值固定为$z_{near}$,而$(x,y)$是栅格化后的fragment从**屏幕空间**坐标映射回**裁剪空间**，一个fragment只映射回了一个三维坐标点，**但整个fragment映射回去有无穷个点**。(一般用fragment中心代表整个fragment)显然这是由于计算机的**离散化**导致**采样不足**导致的问题。

<div align=center>
<img src='https://github.com/wenhao923/blog/blob/main/pictures/GAMES202/aliasing.png'/>
</div>

### 自遮挡
现在把目光移到二、三步计算$z - SM(x,y)$，这步存在问题么？$z$的获取没有问题(虽然在采样fragment坐标存在问题)；

那$SM(x,y)$呢？似乎存在不少问题：
1. $SM$：制作shadow map的对吗？
2. $(x,y)$：坐标位置正确吗？

#### $SM$

<div align=center>
<img src='https://github.com/wenhao923/blog/blob/main/pictures/GAMES202/rasterization.png'/>
</div>

shadow map是由**光栅化**得到的。观察上图，那么光栅化必然带来**走样**的问题，我们采样得来的深度代表了整个fragment的深度。那么shadow map表达的深度含义，其实是下图中的绿色方块，使用近平面上的黄点的深度代表了整个fragment的深度。

<div align=center>
<img src='https://github.com/wenhao923/blog/blob/main/pictures/GAMES202/occupied.png'/>
</div>

#### $(x,y)$
第二步中，计算shading point相对于相机的投影的位置，那么它的坐标$(x,y,z)$是连续的。紧接着，我们使用它的$(x,y)$采样shadow map，那么问题来了！**先前保存的shadow map是栅格化的！** 我们只能对$(x,y)$进行栅格化或者近似操作来得到光源可以“看见”的深度。这样又会带来误差。

于是，自遮挡导致了frack：
<div align=center>
<img src='https://github.com/wenhao923/blog/blob/main/pictures/GAMES202/selfoc.png'/>
</div>
>其实，$SM$和$(x,y)$是一体两面的问题。制作shadow map时采样不足，导致**重投影**后采不到正确的深度。

## 优化

### 自遮挡问题
为了避免frack的问题，有一个开销小且方便的trick。
出问题的地方是错误的$z < SM(x,y)$，由于$z$是完全正确的，我们只需要考虑$SM(x,y)$:

$z - (SM(x,y)+bias)$

加入了bias，让可见深度往后移动。也就是说，我们最好找到整个fragment映射到的最深的深度，作为shadow map。业界常用此方法，调节bias来达到不错的渲染效果。
>此方法是后处理阶段的trick，并没根除上文提到的问题，而且引入了新的问题。

### 走样问题
我们大可以使用一些**反走样**的方法处理走样问题，比如SSAA、TAA、DLAA...
我们也可以反思我们生成过程的问题，试图用一些trick来达到好的阴影效果。毕竟在渲染界中，看起来好的就是好的。此方法**Percentage Closer Filtering(PCF)**，也会引入**软阴影**，下篇博客再介绍。
