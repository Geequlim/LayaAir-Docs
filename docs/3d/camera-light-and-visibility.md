# 为什么这个 3D 场景能被正确看见

一个 3D 场景能不能“正确看见”，通常不是单一问题，而是几层条件同时成立：

1. 有相机在看
2. 相机看向了正确位置
3. 场景里有足够的光或环境信息
4. 背景和环境没有把画面读感弄错
5. 对象和相机的层、可见边界没有互相排掉

所以当你看到“场景是黑的”“模型不见了”“只能看到一部分”“背景不对”时，最好不要只盯某一个模型，而要把整条可见链一起看。

## 先看最小可见链

最短例子：

```ts
await Laya.init({ designWidth: 1334, designHeight: 750 });

const scene = new Laya.Scene3D();
Laya.stage.addChild(scene);

const camera = new Laya.Camera(0, 0.1, 100);
scene.addChild(camera);
camera.transform.position = new Laya.Vector3(0, 2, 6);
camera.transform.rotate(new Laya.Vector3(-15, 0, 0), true, false);

const light = new Laya.DirectionLight();
scene.addChild(light);

const box = new Laya.MeshSprite3D(Laya.PrimitiveMesh.createBox());
scene.addChild(box);
```

这段代码里，画面能出来至少依赖四件事：

1. `scene` 已经接入运行时
2. `camera` 已经在场景里，并且朝向盒子
3. `light` 让模型不至于完全发黑
4. `box` 处在相机可见范围内

只要这条链上少一环，结果就可能是“什么都没看到”，或者“看到了但不对”。

## `Camera` 决定的是“从哪里、以什么范围去看”

很多人会把相机理解成“开关显示的按钮”，但它更准确的作用是定义一次观察。

相机最常影响这些结果：

1. 画面从哪个位置观察场景
2. 画面朝向哪里
3. 哪些距离范围内的对象会进入视锥
4. 背景是纯色、天空还是只保留深度

先看一个最小例子：

```ts
camera.transform.position = new Laya.Vector3(0, 3, 8);
camera.transform.lookAt(new Laya.Vector3(0, 1, 0), new Laya.Vector3(0, 1, 0));
```

这里真正发生的是：

1. 相机站在 `(0, 3, 8)`
2. 相机朝 `(0, 1, 0)` 看
3. 只有落进它观察范围里的对象才会被渲染

所以模型“明明在场景里却看不见”，第一反应通常不该是资源坏了，而是：

1. 相机是不是没看向它
2. 模型是不是在相机后面
3. 模型是不是超出近平面或远平面

## 近平面和远平面决定了“多近和多远还能看见”

创建相机时常见写法是：

```ts
const camera = new Laya.Camera(0, 0.1, 100);
```

这里可以先把它理解成：

1. 小于 `0.1` 的太近内容不看
2. 大于 `100` 的太远内容不看

这就是可见距离边界的一部分。

例如一个对象离相机极近时，可能会被近平面裁掉；一个大地图上的远处建筑，也可能因为超出远平面而直接消失。

所以“走近就穿模一样消失”“远处物体整片没了”，先检查的通常不是材质，而是相机裁剪范围。

## `clearFlag` 决定背景如何被清掉

相机每帧渲染前，要先决定上一帧留下的背景怎么处理。

常见理解方式可以先记成：

1. 纯色清屏：背景就是一个颜色
2. 天空清屏：背景使用天空盒或天空穹顶
3. 只清深度：常见于多相机叠加场景

例如：

```ts
camera.clearFlag = Laya.CameraClearFlags.SolidColor;
camera.clearColor = new Laya.Color(0.12, 0.12, 0.16, 1);
```

如果你明明配了天空盒，但背景还是纯色，常见原因不是天空盒失效，而是相机清屏方式仍然是纯色。

## 光照决定的是“表面如何被读出来”

3D 里“能被看见”和“看起来正确”不是一回事。

有些对象即使已经被相机拍到了，如果场景没有合适的光照，结果仍然可能接近全黑，或者层次非常差。

最常见的第一盏灯通常是平行光：

```ts
const light = new Laya.DirectionLight();
scene.addChild(light);
light.transform.rotate(new Laya.Vector3(-45, 0, 0), true, false);
```

可以先把它理解成“来自远处的大方向光”，很像太阳光的简化版本。

它主要影响的是：

1. 哪些面被照亮
2. 明暗方向感是否成立
3. 阴影方向和空间层次是否自然

如果没有这类主光源，很多模型虽然还在，但体积感会很弱。

## 环境光不是主光方向，但它决定暗部是不是死黑

主光负责方向感，环境光更像在补整体基底亮度。

例如：

```ts
scene.ambientColor = new Laya.Color(0.35, 0.35, 0.35, 1);
```

这里的作用通常是：

1. 让背光面不至于完全漆黑
2. 让没有直接吃到主光的区域仍然保留一点可读性
3. 让整体明暗更接近真实环境

所以场景“不是完全看不见，但阴影面黑成一团”时，往往要同时看：

1. 主光方向是否合理
2. 环境光是否过低

## 天空盒和环境设置决定的是“场景外部世界给了什么背景和气氛”

天空盒不只是拿来“把背景变好看”。

它至少影响两层感受：

1. 相机看到的远处背景是什么
2. 整个场景是否有明确的环境氛围

例如：

```ts
scene.skyRenderer.mesh = Laya.SkyDome.instance;
scene.skyRenderer.material = new Laya.SkyProceduralMaterial();
camera.clearFlag = Laya.CameraClearFlags.Sky;
```

这几句合起来表达的才是：

1. 场景有天空渲染器内容
2. 相机允许按天空方式清背景

少任何一边，都不会得到预期天空效果。

可以把背景环境先理解成一个边界条件：

1. 它不直接替代主光和材质
2. 但它会强烈影响画面的空间感、气氛和“是不是像在一个完整世界里”

## 阴影决定的是空间接触感，不是“有没有显示出来”

很多初学者会把“没有阴影”理解成“灯光没生效”，其实不完全对。

对象有没有被渲染出来，和阴影不是同一层问题。

阴影更主要解决的是：

1. 物体和地面是否像真的接触在一起
2. 前后层次是否更容易读出来
3. 光源方向是否更有说服力

所以阴影没开时，常见结果是“画面发飘”“像贴上去的”，而不是对象完全消失。

## `layer` 和 `cullingMask` 决定“谁允许被这台相机看到”

3D 场景里还有一条非常常见的可见边界：层过滤。

可以先把它理解成两边配合：

1. 对象在某个层上
2. 相机只看某些层

例如：

```ts
box.layer = 1;
camera.cullingMask = 1 << 1;
```

这时表达的是：

1. 盒子被放进第 `1` 层
2. 相机只渲染第 `1` 层内容

如果对象层和相机掩码对不上，结果通常就是：

1. 对象明明存在
2. 位置也对
3. 光也正常
4. 但这台相机就是看不见它

这也是做这些需求时最常用的边界：

1. 主场景和 UI 3D 分层
2. 小地图相机只看特定对象
3. 特效相机只补一部分层

## `active` 和渲染开关决定对象有没有参与这次显示

除了层过滤，还有更直接的可见边界：对象自己是不是启用状态。

例如：

```ts
box.active = false;
```

这类开关影响的是对象是否还参与场景树和后续流程。

所以一个对象看不见，排查顺序里也要包含：

1. 节点是不是已经被禁用
2. 它的父节点是不是被禁用

因为父节点失活后，子节点通常也不会继续按正常可见链参与渲染。

## 真正的“可见边界”通常是几层一起生效

一个 3D 对象最终能不能被看见，常常同时受这些边界约束：

1. 它在不在场景树里
2. 它是不是启用状态
3. 它是不是落在相机朝向和视锥里
4. 它有没有被近平面和远平面裁掉
5. 它的层是否被当前相机允许渲染

所以“我明明把模型加进场景了为什么还是没看到”这句话，通常还缺了至少一半信息。

真正要问的是：

1. 被哪台相机看
2. 相机从哪里看
3. 相机允许看哪些层
4. 光照和环境是否足以把它读出来

## 一个更接近真实项目的最小配置

```ts
await Laya.init({ designWidth: 1334, designHeight: 750 });

const scene = new Laya.Scene3D();
Laya.stage.addChild(scene);

scene.ambientColor = new Laya.Color(0.3, 0.3, 0.3, 1);
scene.skyRenderer.mesh = Laya.SkyDome.instance;
scene.skyRenderer.material = new Laya.SkyProceduralMaterial();

const camera = new Laya.Camera(0, 0.1, 100);
scene.addChild(camera);
camera.clearFlag = Laya.CameraClearFlags.Sky;
camera.transform.position = new Laya.Vector3(0, 2, 6);
camera.transform.lookAt(new Laya.Vector3(0, 1, 0), new Laya.Vector3(0, 1, 0));

const light = new Laya.DirectionLight();
scene.addChild(light);
light.transform.rotate(new Laya.Vector3(-45, 0, 0), true, false);

const box = new Laya.MeshSprite3D(Laya.PrimitiveMesh.createBox());
box.transform.position = new Laya.Vector3(0, 0.5, 0);
scene.addChild(box);
```

这个例子里，每一层职责都比较清楚：

1. `camera` 决定观察位置、观察方向和背景清除方式
2. `light` 决定主光方向
3. `ambientColor` 决定暗部基础亮度
4. `skyRenderer` 和天空材质决定环境背景
5. `box` 只负责作为被看的对象进入场景

## 场景看不对时，最稳的排查顺序

如果只记一套排查顺序，可以先按这条走：

1. 先确认场景和对象是否真的已经加入 `Scene3D`
2. 再确认对象和父节点是否处于启用状态
3. 再确认相机位置、朝向、近平面和远平面
4. 再确认对象层和 `camera.cullingMask` 是否匹配
5. 再确认主光、环境光和背景环境是否合理

把这几层边界分清之后，很多“3D 场景为什么没显示对”的问题，都会从模糊问题变成可以逐项排掉的具体问题。
