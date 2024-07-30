
## Rendering Equation

![Pasted image 20240728223325.png](/笔记/attachments/Pasted%20image%2020240728223325.png)


| **求解GI(光照)** | 积分  | 较低频信号 | volume progation，cone sampling     |
| ------------ | --- | ----- | ---------------------------------- |
| **求解几何/阴影**  | 细分  | 高频信号  | culling，visibility buffer sampling |


### Monte Carlo 积分

Monte Carlo积分是使用随机采样方法来逼近复杂函数的积分。
Monte Carlo ray tracing是使用随机采样方法来模拟光线在场景中传播的渲染方法。(离线渲染)

Uniform sampling
![Pasted image 20240728224750.png](/笔记/attachments/Pasted%20image%2020240728224750.png)

Importance sampling (with PDF)
![Pasted image 20240728224646.png](/笔记/attachments/Pasted%20image%2020240728224646.png)

sampling is matter：
1. 平均
2. Cosine
3. GGX

## Reflective Shadow Map

我们尝试让ShadowMap(被主光源首次照射到的面)成为光源，考虑ShadowMap对像素的照射。

Photon Mapping 将光子注入到ShadowMap上

关于RSM的GI：求解间接光

对每个像素在ShadowMap上撒400个cone(使用泊松采样)，对于cone大小可以使用合适的mipmap来查询
![Pasted image 20240728225541.png](/笔记/attachments/Pasted%20image%2020240728225541.png)

低精度采样+插值获得低频，fail check，用正常精度计算高频
![Pasted image 20240728230808.png](/笔记/attachments/Pasted%20image%2020240728230808.png)

sub indirect light
cone tracing - mipmap query

### Light Propagation Volumes

Volumetric radiance propagation

使用3D grid，保存SH(1级2级共1+3=4个)。

SH函数：对球面函数的正交函数集，较为低频。

从6个面更新下一帧的radiance。

物理上不正确

### SparseVoxelOctreeGI

sparse稀疏的

triangle voxel

对每个三角面，保证有一个尽量小的voxel包围，使用八叉树组织所有小voxel。

使用上面的模型来统计RSM的radiance。

对于像素通过八叉树查询，使用cone查询时可以直接用八叉树上的mipmap。

现在游戏三角面数增大，不再具有实际意义了。

### VXGI

Clipmap：每个LOD都有相同个voxel elements(k x k x k)，以view point展开，对近处更细节

voxel update，类似循环队列，保留帧变化之间重复的内存块

![Pasted image 20240728233956.png](/笔记/attachments/Pasted%20image%2020240728233956.png)

对每个voxel对6个面计算opacity，使用了MSAA算法来细化opacity计算

注入RSM到voxel得到radiance

cone tracing

![Pasted image 20240728234115.png](/笔记/attachments/Pasted%20image%2020240728234115.png)

缺点：遮蔽错误，能量不守恒漏光

### SSGI

对像素使用ray marching，使用HighZBuffer，Mipmap优化marching速度，快速找到第一个碰撞位置。

对临近像素复用光线(如果没有遮挡的话)
![Pasted image 20240728234952.png](/笔记/attachments/Pasted%20image%2020240728234952.png)

优点：速度快，高频，支持动态光照。

但是只能处理屏幕内的radiance。

### DDGI (没有讲到的)

Diffuse Direct Global Illumination

![Pasted image 20240730234815.png](/笔记/attachments/Pasted%20image%2020240730234815.png)
![Pasted image 20240730234839.png](/笔记/attachments/Pasted%20image%2020240730234839.png)

## SDF 光线追踪

SDF：

Mesh SDF

Global SDF

raytracing


Cone Tracing with SDF， soft shadow
![Pasted image 20240728235956.png](/笔记/attachments/Pasted%20image%2020240728235956.png)

![Pasted image 20240728235756.png](/笔记/attachments/Pasted%20image%2020240728235756.png)

![Pasted image 20240728235827.png](/笔记/attachments/Pasted%20image%2020240728235827.png)

![Pasted image 20240728235402.png](/笔记/attachments/Pasted%20image%2020240728235402.png)

对Sparse Mesh SDF节省空间

![Pasted image 20240729000129.png](/笔记/attachments/Pasted%20image%2020240729000129.png)

![Pasted image 20240729000218.png](/笔记/attachments/Pasted%20image%2020240729000218.png)


Global SDF：根据Camera Clipmap



## lumen 光辉缓存
#### generate surface cache

mesh card 6个面的正交投影，AABB box

4096x4096 (albedo normal opacity depth emissive) cache atlas

128x128 pages

将mesh card放入pages
![Pasted image 20240729014736.png](/笔记/attachments/Pasted%20image%2020240729014736.png)
![Pasted image 20240729022111.png](/笔记/attachments/Pasted%20image%2020240729022111.png)

将direct light(多光源)+emission light cache固定到surface cache上

实际上是在做RSM到mesh card的注入，photon to surface cache

多光源解决方法：使用tile(8x8)方法，先确定光会影响的tile，对于每个tile取影响最大的8个光，遍历tile，通过一次SDFraytracing来判断shadows，可以被1bit mask去掉。
![Pasted image 20240729023455.png](/笔记/attachments/Pasted%20image%2020240729023455.png)
![Pasted image 20240729021414.png](/笔记/attachments/Pasted%20image%2020240729021414.png)


#### Voxel Clipmap for Radiance cache

原因：Global SDF can't sample surface cache. SDF hits position and normal

Clipmap0 = 50m^3 ， Clipmap2 Clipmap3调低更新频率

#### 计算Voxel lighting(Mesh cache -> Voxel)

对Voxel做filtering，先分割Voxel为 4x4x4 Tile，生成光线从6面射入。
使用Mesh SDF(因为比较近)求交得到VisBuffer(nanite中的vbuffer)获得distance，object id信息，于是可以采样final mesh card得到这一帧的voxel lighting。

![Pasted image 20240729024330.png](/笔记/attachments/Pasted%20image%2020240729024330.png)
![Pasted image 20240729020409.png](/笔记/attachments/Pasted%20image%2020240729020409.png)


![Pasted image 20240729015706.png](/笔记/attachments/Pasted%20image%2020240729015706.png)

这一帧的voxel lighting会生成indirect lighting，作用下一帧的surface cache。

#### 生成Indirect Lighting(Voxel -> Mesh cache)

在Texture上使用probe，通过raytracing(Jitter)获取voxel信息，probe里面保存SH，对于mesh cache的像素会从四个probe的SH插值得到间接光

![Pasted image 20240729231924.png](/笔记/attachments/Pasted%20image%2020240729231924.png)

![Pasted image 20240729232135.png](/笔记/attachments/Pasted%20image%2020240729232135.png)

$\text{Final Lighting = (Direct Lighting + Indirect Lighting) * DiffuseLambert(albedo) + Emissive}$

课程中没有讲到：
	Terrain mesh怎么用cache表达
	体积云体积雾怎么计算

每帧更新策略
![Pasted image 20240729233251.png](/笔记/attachments/Pasted%20image%2020240729233251.png)
## lumen 探针

光辉缓存和Voxel，维护了全局的光照缓存，现在我们通过Probe来从缓存中获得光照信息，展示到相机中。

### Screen Space Probe

* 采样：Octahedron mapping球面采样 (Radiance Probe)

![Pasted image 20240729233432.png](/笔记/attachments/Pasted%20image%2020240729233432.png)


* 分布：屏幕上16x16，对区域内深度分布，如果差距过大会fail到8x8，4x4
	另外，可以通过packing，对自适应的probe合理存储

![Pasted image 20240729234222.png](/笔记/attachments/Pasted%20image%2020240729234222.png)
![Pasted image 20240729234329.png](/笔记/attachments/Pasted%20image%2020240729234329.png)

* 插值：像素上，通过法线得到平面位置，和平面距离一起插值计算

![Pasted image 20240729234113.png](/笔记/attachments/Pasted%20image%2020240729234113.png)


raytracing(Jitter)

![Pasted image 20240729234645.png](/笔记/attachments/Pasted%20image%2020240729234645.png)


1. 通过光源位置 => 对上一帧的probe的更亮的方向jitter，得到PDF
2. 在32x32邻域pixel中采样64个，将它们积分到一个SH中，得到PDF

对PDF统一排序，对不足64rays，依次super sampling
![Pasted image 20240729235126.png](/笔记/attachments/Pasted%20image%2020240729235126.png)

Denoise：filtering neighbours
1. 光线夹角小于10度 (同方向)
2. 光线hit距离差距要小 (无遮挡)

### World Space Radiance Cache

补充远处光采样

* 采样： 32x32(Radiance Probe)
* 分布：Clipmap0： 50m x 50m x 50m，48x48x48
* 剔除：从对角线长度之外发射光线

连接到Screen Space Probe光线
![Pasted image 20240730000952.png](/笔记/attachments/Pasted%20image%2020240730000952.png)

每个Screen Space Probe会mark周围对角线上的World Probe，表示需要更新

最终：表面法线normal texture + Screen Space Probe转SH => 屏幕光照
![Pasted image 20240730001306.png](/笔记/attachments/Pasted%20image%2020240730001306.png)


## Lumen

![Pasted image 20240730001420.png](/笔记/attachments/Pasted%20image%2020240730001420.png)

![Pasted image 20240730001500.png](/笔记/attachments/Pasted%20image%2020240730001500.png)

1/2 ray per pixel = 3.74ms
1/8 ray per pixel = 2.15ms

![Pasted image 20240730001632.png](/笔记/attachments/Pasted%20image%2020240730001632.png)
