---
title: "3D 对象的渲染链路"
order: 3
---

# 一个 3D 对象是怎么被渲染出来的

很多人第一次写出一个 3D 物体时，会把“模型显示出来了”理解成一件很简单的事。

但运行时里，真正发生的是一条链：

1. `Mesh` 提供几何数据
2. `Renderer` 负责把这个对象送进渲染流程
3. `Material` 决定这个表面该怎么着色
4. `Texture` 给材质提供颜色、法线、粗糙度这类输入
5. 相机、灯光和环境再参与最终结果计算

如果这条链没分开看，就很容易把下面几件事混成一件事：

1. 为什么一个对象有节点但还是看不见
2. 为什么换了贴图，显示结果却不只是“换张图”
3. 为什么同一个模型换个材质，观感会差很多
4. 为什么 PBR 材质看起来会受光照和环境影响

## 先看最小主链

先看一个最短例子：

```ts
await Laya.init({ designWidth: 1334, designHeight: 750 });

const scene = new Laya.Scene3D();
Laya.stage.addChild(scene);

const camera = new Laya.Camera(0, 0.1, 100);
camera.transform.position = new Laya.Vector3(0, 1.5, 4);
camera.transform.lookAt(new Laya.Vector3(0, 0, 0), new Laya.Vector3(0, 1, 0));
scene.addChild(camera);

const light = new Laya.DirectionLight();
light.color = new Laya.Color(1, 1, 1, 1);
scene.addChild(light);

const box = scene.addChild(new Laya.MeshSprite3D(Laya.PrimitiveMesh.createBox())) as Laya.MeshSprite3D;

const material = new Laya.PBRStandardMaterial();
box.meshRenderer.material = material;
```

这段代码里，真正把物体显示出来的不是某一个单独 API，而是几层对象一起配合：

1. `box` 是场景里的 3D 节点
2. `createBox()` 生成了这个节点要画的 `Mesh`
3. `meshRenderer` 持有这个对象的渲染配置
4. `material` 决定这个表面的着色方式
5. `camera` 和 `light` 让这个结果能被看见，而且看起来像一个立体物体

所以“一个 3D 对象被渲染出来”不只是“场景里有个节点”，而是“这条渲染链被接完整了”。

## 先分清节点和可渲染对象不是一回事

在 3D 场景里，节点树先解决的是“对象在场景里的层级和变换关系”。

但是否真的能被画出来，还要看它是不是一个可渲染对象。

最常见的情况可以先记成：

1. `Sprite3D` 负责当 3D 节点
2. `Transform3D` 负责位置、旋转、缩放
3. `MeshSprite3D` 这类对象额外带有网格和渲染器，所以能参与绘制

例如：

```ts
const root = new Laya.Sprite3D();
scene.addChild(root);

const empty = new Laya.Sprite3D();
root.addChild(empty);

const cube = new Laya.MeshSprite3D(Laya.PrimitiveMesh.createBox());
root.addChild(cube);
```

这里的区别是：

1. `empty` 只有层级和变换，不会自己出现在画面里
2. `cube` 除了层级和变换，还有可用于绘制的网格与渲染器

这也是一个很常见的判断边界：

场景里“有这个对象”不等于“渲染器已经有东西可画”。

## `Mesh` 先回答“这个物体长什么样”

`Mesh` 可以先理解成几何数据集合。

它至少描述了绘制这个物体所需要的一部分基础信息，例如：

1. 顶点位置
2. 法线方向
3. UV 坐标
4. 索引顺序

换句话说，`Mesh` 先决定的是物体表面如何由三角形拼起来。

例如：

```ts
const sphereMesh = Laya.PrimitiveMesh.createSphere(0.5);
const sphere = new Laya.MeshSprite3D(sphereMesh);
scene.addChild(sphere);
```

这里真正变化的首先不是颜色，而是几何外形。

所以：

1. 换 `Mesh`，通常是在换形状
2. 换 `Material`，通常是在换表面表现
3. 两者经常一起出现，但职责不同

如果一个对象没有可用的网格数据，渲染器就没有表面可以提交给 GPU。

## `Renderer` 负责把对象接进渲染流程

对 `MeshSprite3D` 来说，最常接触的是 `meshRenderer`。

它的职责不要简单理解成“保存材质”，更准确一点是：

1. 它知道这个对象要不要参与渲染
2. 它管理这个对象的材质槽位
3. 它参与可见性和绘制提交
4. 它把网格、材质、变换等信息组合成真正可渲染的数据

最短示例：

```ts
const cube = new Laya.MeshSprite3D(Laya.PrimitiveMesh.createBox());
scene.addChild(cube);

cube.meshRenderer.sharedMaterial = new Laya.BlinnPhongMaterial();
```

这里值得先分清的是：

1. `MeshSprite3D` 是节点对象
2. `meshRenderer` 是这个节点上的渲染组件

所以很多“为什么这个物体不显示”的问题，真正该检查的不是只有节点在不在场景里，还包括：

1. 渲染器是否存在
2. 渲染器是否启用
3. 材质是否可用
4. 相机是否能看到它

## `Material` 决定“这个表面该怎么被画”

材质不是贴图本身。

更稳妥的理解是：

1. 材质定义了使用哪种着色模型
2. 材质持有一组会影响着色结果的参数
3. 贴图只是这些参数的一部分输入

例如同一个立方体，只换材质，观感就会变：

```ts
const cube = new Laya.MeshSprite3D(Laya.PrimitiveMesh.createBox());
scene.addChild(cube);

const material = new Laya.PBRStandardMaterial();
material.albedoColor = new Laya.Color(0.9, 0.7, 0.2, 1);
cube.meshRenderer.material = material;
```

这里改变的不是网格，而是表面的着色方式和参数。

因此，一个很实用的边界是：

1. `Mesh` 决定轮廓和表面结构
2. `Material` 决定光如何和这个表面发生作用

## `Texture` 不是“贴上去的一张图”这么简单

在 3D 里，纹理最容易被误解成“给模型贴一张图片”。

这只说对了一部分。

更准确地说，纹理是材质参数的数据来源之一。

常见情况包括：

1. 基础颜色纹理
2. 法线纹理
3. 金属度纹理
4. 粗糙度或光滑度纹理
5. 遮罩类纹理

最短示例：

```ts
const texture = await Laya.loader.load("resources/3d/wood/albedo.png") as Laya.Texture2D;

const material = new Laya.PBRStandardMaterial();
material.albedoTexture = texture;

box.meshRenderer.material = material;
```

这里的关键点不是“box 有了一张图”，而是“材质里的基础颜色输入来自这张纹理”。

所以同一张纹理：

1. 放到不同材质类型上，结果可能不同
2. 放到同一材质的不同槽位上，结果也完全不同

例如把一张法线纹理当颜色纹理使用，结果通常就不会对。

## 把 `Mesh`、`Renderer`、`Material`、`Texture` 串起来看

如果只记一条最重要的关系，可以先记这条：

1. `Mesh` 提供要画的表面
2. `Renderer` 负责把这个表面送进渲染流程
3. `Material` 规定这个表面如何着色
4. `Texture` 给材质参数提供数据输入

可以把它想成一句话：

渲染器拿着“这个物体的形状”，再按“这个表面该怎么画”的规则，把它交给相机视角下的渲染流程。

## 为什么同一个模型换材质后会像换了一个物体

原因就在于，3D 观感很大一部分来自表面如何响应光照，而不是只来自几何轮廓。

例如一个球体：

1. 用纯色、无明显高光的材质，看起来像粉笔或塑料
2. 用更强镜面、较平滑的材质，看起来可能更像抛光金属
3. 再换上法线和粗糙度纹理，表面细节会继续变化

所以项目里经常会出现这种现象：

1. 模型没换
2. 网格没换
3. 只是材质和纹理换了
4. 视觉上却像换了一个资产

这不是错觉，而是材质链本来就在决定最终观感。

## PBR 应该先怎么理解

PBR 可以先不要理解成“一组复杂参数名”。

先抓住最实用的核心：

PBR 是一种更强调按真实光照规律去描述表面的材质方式。

对使用层来说，最值得先记住的是：

1. 它不是单纯给物体上色
2. 它很依赖灯光和环境信息
3. 它通常会同时关心基础颜色、金属感、粗糙程度这类表面属性

也就是说，在 PBR 材质里，一个物体看起来像木头、塑料还是金属，不只是颜色不同，而是表面属性不同。

## 用最小方式理解基础 PBR 参数

如果只看最基础、最常用的几个概念，可以先这样理解：

### `albedo`

表示基础颜色。

它回答的是“这个表面本来的颜色大致是什么”。

### `metallic`

表示金属感强不强。

它影响的是这个表面更像非金属，还是更像金属材质。

### `smoothness`

可以先近似理解成表面有多光滑。

它越高，反射通常越集中；越低，表面通常越显得粗糙。

最短示例：

```ts
const material = new Laya.PBRStandardMaterial();
material.albedoColor = new Laya.Color(0.8, 0.8, 0.8, 1);
material.metallic = 1.0;
material.smoothness = 0.9;

box.meshRenderer.material = material;
```

这段代码的重点不是数值本身，而是：

1. 基础颜色只是一部分
2. 金属度和光滑度会直接影响高光和反射观感
3. 这类材质效果要结合灯光和环境一起看才有意义

## 为什么 PBR 材质离不开灯光和环境

如果前一篇已经分清了相机、灯光和可见性，这里就更容易理解：

1. 相机决定你从哪里看
2. 灯光决定表面怎么被照亮
3. 环境决定很多反射和整体氛围来源

这也是为什么：

1. 一个 PBR 材质在不同场景里观感会变
2. 只改环境光或方向光，物体就可能像换了质感
3. 没有合适光照时，PBR 材质往往看不出优势

所以不要把 PBR 理解成“换一个更高级的材质类名就结束了”。

它真正生效，依赖的是整条渲染链共同成立。

## 一个对象最终被渲染出来，最少要满足什么

把前面的内容压成最小检查表，可以先看这几件事：

1. 它已经在某个 `Scene3D` 的对象树里
2. 有相机能看到它
3. 它本身是可渲染对象，而不是纯空节点
4. 它有可用的 `Mesh`
5. 它的 `Renderer` 在正常参与渲染
6. 它绑定了可用的 `Material`
7. 如果材质依赖纹理，相关 `Texture` 已正确加载并设置
8. 如果是 PBR 材质，场景里的灯光和环境足以让表面特征表现出来

这八件事里，缺的往往不是“某个 API 会不会写”，而是链条中有一环没有接上。

## 一个更完整但仍然很短的例子

```ts
await Laya.init({ designWidth: 1334, designHeight: 750 });

const scene = new Laya.Scene3D();
Laya.stage.addChild(scene);

const camera = new Laya.Camera(0, 0.1, 100);
camera.transform.position = new Laya.Vector3(0, 1.2, 3);
camera.transform.lookAt(new Laya.Vector3(0, 0, 0), new Laya.Vector3(0, 1, 0));
scene.addChild(camera);

const light = new Laya.DirectionLight();
light.color = new Laya.Color(1, 1, 1, 1);
scene.addChild(light);

const sphere = scene.addChild(new Laya.MeshSprite3D(Laya.PrimitiveMesh.createSphere(0.5))) as Laya.MeshSprite3D;

const albedo = await Laya.loader.load("resources/3d/demo/albedo.png") as Laya.Texture2D;

const material = new Laya.PBRStandardMaterial();
material.albedoTexture = albedo;
material.metallic = 0.2;
material.smoothness = 0.6;

sphere.meshRenderer.material = material;
```

这条链可以直接按顺序读：

1. `Scene3D` 提供 3D 运行空间
2. `Camera` 提供观察视角
3. `DirectionLight` 提供主要光照
4. `MeshSprite3D` 持有可渲染的球体网格
5. `PBRStandardMaterial` 定义表面着色方式
6. `albedoTexture` 给材质提供基础颜色输入
7. `meshRenderer` 把这组信息接到渲染流程里

## 结论

一个 3D 对象被渲染出来，不是因为“场景里放了个模型”这么简单。

更准确地说，是下面这条链成立了：

1. 节点进入了 `Scene3D`
2. `Mesh` 定义了形状
3. `Renderer` 负责提交绘制
4. `Material` 定义着色规则
5. `Texture` 提供表面数据
6. 相机、灯光和环境共同参与最终显示

把这条链分清之后，再看材质、贴图、PBR、模型显示异常这类问题，就会容易定位得多。
