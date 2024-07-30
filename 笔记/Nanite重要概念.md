## Compute Shader

用处是：减少draw call，减少cpu overhead

GPU driven render pipeline: 使用GPU做LOD selection，Culling

![Pasted image 20240730002101.png](/笔记/attachments/Pasted%20image%2020240730002101.png)

## 刺客信条：大革命

### Mesh Cluster Pipeline

64 triangles 做一个cluster

cluster：单独bounding，单独culling，承担每帧随机对物体部分更新

CPU： Coarse Frustum Culling(视锥体剔除，遮挡剔除)+合批
	在CPU端做剔除：牺牲CPU性能但节省上传带宽

GPU：
1. **拆分Instance**
2. **Cluster剔除**
3. **三角面片剔除**
4. **Orientation and Zero Area Culling**
5. **Depth Culling-Hi-z**
6. **Index buffer compaction**
7. **间接绘制**(DX12提供了**间接调用函数**调用Index Buffer)
8. **drawcall compaction**
![Pasted image 20240730010734.png](/笔记/attachments/Pasted%20image%2020240730010734.png)
![Pasted image 20240730010758.png](/笔记/attachments/Pasted%20image%2020240730010758.png)
![Pasted image 20240730010821.png](/笔记/attachments/Pasted%20image%2020240730010821.png)

![Pasted image 20240730011644.png](/笔记/attachments/Pasted%20image%2020240730011644.png)

### Occlusion

关键是减少overdraw：因为并行计算性能损耗小于buffer写入

通过上一帧的zbuffer预先culling(保守过滤)

对visible objects生成新的zbuffer

测试其他culled objects(可能被测试两边)

>测试次数会增多，但是减少overdraw

![Pasted image 20240730011839.png](/笔记/attachments/Pasted%20image%2020240730011839.png)


通过相机生成frustrum，长度为max z

如果投射阴影的物体被spectrum的背面遮挡，则不用在其上绘制阴影(cast a visible shadow)(举例：在街道上行走)

![Pasted image 20240730012404.png](/笔记/attachments/Pasted%20image%2020240730012404.png)

### Visibility Buffer

![Pasted image 20240730012650.png](/笔记/attachments/Pasted%20image%2020240730012650.png)

对传统G-buffer，复杂场景产生overdraw，查询阴影慢，cache miss...

生成V-Buffer
![Pasted image 20240730012934.png](/笔记/attachments/Pasted%20image%2020240730012934.png)

![Pasted image 20240730013050.png](/笔记/attachments/Pasted%20image%2020240730013050.png)
可以兼容deferred shading

缺少rasterizer：ddx ddy 2x2

![Pasted image 20240730013310.png](/笔记/attachments/Pasted%20image%2020240730013310.png)

## Nanite

#### Virtual Texture

mipmap 分为块，按view范围和LOD加载，减小物理内存中texture占用

![Pasted image 20240730013441.png](/笔记/attachments/Pasted%20image%2020240730013441.png)

### Geometry

SDF - (filterable，不易绘制)

Voxels - (可以用Clipmap octotree组织)
![Pasted image 20240730014055.png](/笔记/attachments/Pasted%20image%2020240730014055.png)



subdivision surface 曲面细分 (upsampling)

![Pasted image 20240730014128.png](/笔记/attachments/Pasted%20image%2020240730014128.png)

Point Clouds (massive overdraw，缺失details，三维重建)

![Pasted image 20240730014526.png](/笔记/attachments/Pasted%20image%2020240730014526.png)

Maps-based (normal parallax ...)

左边可以，右边困难
![Pasted image 20240730014236.png](/笔记/attachments/Pasted%20image%2020240730014236.png)

1 filterable -> downsampling

2 mipmap

3 drawable -> overdraw

### nanite

more triangles than screen pixel!

Cluster based rendering

Geometry representation

![Pasted image 20240730015200.png](/笔记/attachments/Pasted%20image%2020240730015200.png)

![Pasted image 20240728163152.png](/笔记/attachments/Pasted%20image%2020240728163152.png)


组织Cluster数据结构： 
- 根据128或64个面为一组生成Cluster，利用Mesh LOD 0的原始模型面进行构建。 
- 使用图切分（graph partition）规则处理，确保Cluster面积接近、边尽可能少。 
图切分条件： 
- 面积尽可能均匀，以在LOD切换时在屏幕上的投影尽可能均匀。 
- 边尽可能少，以确保减面质量好且在锁住边界时具有大的减面自由度。 
使用第三方库matrix，经过针对特殊边界划分需求的定制修改。 
减面过程： 
- 从LOD 0的Mesh开始，生成一组Cluster，再根据图切分条件合并成Cluster Group。 
- 在Cluster Group内部进行减面操作，通过解开边界、锁定Cluster边界并逐步减面生成新的Cluster Group。 
存储额外信息： 
- 每次减面后记录最大误差，每个Cluster存储当前级别的误差和上一级别的误差。 
- 确保所有Cluster Group的合并串按误差降序排列，保证误差依次增大。

![Pasted image 20240728163752.png](/笔记/attachments/Pasted%20image%2020240728163752.png)


LOD cluster Error 二次度量误差？

https://mstifiy.github.io/2023/08/02/%E7%BD%91%E6%A0%BC%E7%AE%80%E5%8C%96(QEM)/


关于三角面减少可以去games102查看。

![Pasted image 20240730015556.png](/笔记/attachments/Pasted%20image%2020240730015556.png)
error和normal有关，和view距离有关

得到一个DAG

![Pasted image 20240730015258.png](/笔记/attachments/Pasted%20image%2020240730015258.png)

cull条件
![Pasted image 20240730015416.png](/笔记/attachments/Pasted%20image%2020240730015416.png)

DAG向上传递时取在cluster算法中更大的Error，保证parentError是唯一确定的。因此不会出现渲染两次的情况。

将DAG拍平为数组遍历。利于并行


用BVH加速，BVH节点记录子树整体max error，对threshold可以范围筛选

![Pasted image 20240730015515.png](/笔记/attachments/Pasted%20image%2020240730015515.png)

四叉树，保持树形态平衡

![Pasted image 20240730015637.png](/笔记/attachments/Pasted%20image%2020240730015637.png)

当父亲节点失败后立即递归子节点

使用multi producer multi consumer queue控制并发，一种无锁并发模型

## Rasterization

#### 硬件Rasterization

1. 填充2x2像素来计算ddx ddy(用于Mipmap)
2. tile traversal扫描，用来加速三角形在屏幕的绘制


![Pasted image 20240730195329.png](/笔记/attachments/Pasted%20image%2020240730195329.png)
![Pasted image 20240730195526.png](/笔记/attachments/Pasted%20image%2020240730195526.png)


这在现代游戏对小三角形是非常浪费的(2x2绘制，扫描)

#### Software Rasterization

接管小三角形的Rasterization，3x faster

* 对于小于1像素的三角形，直接绘制单像素并通过三个顶点信息计算ddx ddy
* 对于三个边边长都小于18的三角形，找出box(应该小于18x18)，扫描遍历
* 否则使用硬件Rasterization

![Pasted image 20240730200523.png](/笔记/attachments/Pasted%20image%2020240730200523.png)
![Pasted image 20240730200538.png](/笔记/attachments/Pasted%20image%2020240730200538.png)

nanite怎么写入buffer？

使用64bit atomic操作(需要显卡扩展支持)，并采用depth减少overdraw

得到visibility buffer

![Pasted image 20240730200855.png](/笔记/attachments/Pasted%20image%2020240730200855.png)

实际场景使用频率图：

![Pasted image 20240730204706.png](/笔记/attachments/Pasted%20image%2020240730204706.png)


对tiny instance生成imposter来表示远处小物体

![Pasted image 20240730204754.png](/笔记/attachments/Pasted%20image%2020240730204754.png)


![Pasted image 20240730204913.png](/笔记/attachments/Pasted%20image%2020240730204913.png)


## Deferred Material

### Material Sort with Tile-Based Rendering

#### Material Tile Remap Table

pixels -> tiles -> tile groups

建一张表存储tile groups和materials的关系，用threads计算这张表

![Pasted image 20240730210215.png](/笔记/attachments/Pasted%20image%2020240730210215.png)


![Pasted image 20240730211118.png](/笔记/attachments/Pasted%20image%2020240730211118.png)

当depth不相等时cull
![Pasted image 20240730211434.png](/笔记/attachments/Pasted%20image%2020240730211434.png)

目的是，利用现代图形API的indirect draw减少drawcall

上面是UE5.0的nanite方法，UE5.1使用了Programmable Raster

在GDC2024有最新的Material Pipeline方法 - UE5.4

https://www.unrealengine.com/en-US/blog/take-a-deep-dive-into-nanite-gpu-driven-materials

(PPT太长了)
## Shadow

CSM:多层CSM导致内存消耗大
#### Sample Distribution Shadow Maps

![Pasted image 20240730212421.png](/笔记/attachments/Pasted%20image%2020240730212421.png)
![Pasted image 20240730212439.png](/笔记/attachments/Pasted%20image%2020240730212439.png)

### Virtual Shadow Map

1 shadow map texel = n pixel ?

按照在屏幕的大小，分为shadow map page，重要的是可以Cache它们。(因为大部分光是不动的)

当相机移动时，可以类似clipmap的cache方法来做部分更新

![Pasted image 20240730212524.png](/笔记/attachments/Pasted%20image%2020240730212524.png)


对光源使用Shadow Map，点光源使用6个

![Pasted image 20240730212627.png](/笔记/attachments/Pasted%20image%2020240730212627.png)

对三种光源的shadow划分

![Pasted image 20240730230533.png](/笔记/attachments/Pasted%20image%2020240730230533.png)


### Shadow Page Cache

只缓存屏幕可见的像素

Index表示
![Pasted image 20240730231031.png](/笔记/attachments/Pasted%20image%2020240730231031.png)

更新方式：
1. 相机移动，更新clipmap边界的page
2. 光源移动，更新所有page
3. Geometry that casts shadows moving
4. material that modify mesh positions



## Streaming

#### 对cluster LOD使用Pages缓存

Root Page(64k)：永久保存在GPU中

Streaming Page(128k):在CPU使用LRU决定生命周期

![Pasted image 20240730233303.png](/笔记/attachments/Pasted%20image%2020240730233303.png)

#### Memory Represetation

尽量用定点数保存顶点位置，可以避免浮点误差。编码和解码，并节约存储空间。

#### Disk Representation

![Pasted image 20240730234150.png](/笔记/attachments/Pasted%20image%2020240730234150.png)

