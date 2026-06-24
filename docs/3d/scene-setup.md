---
title: "3D 场景搭建"
order: 1
---

# 一个 3D 场景是怎么搭起来的

很多人第一次接触 3D 运行时时，会把下面几件事混成一件事：

1. `Scene3D` 是什么
2. `Sprite3D` 是什么
3. `Transform3D` 到底管什么
4. 场景层级为什么会影响位置和旋转
5. 脚本通常该挂在哪一层

如果把这几条先分开，3D 场景的组织方式其实很稳定。

## 先看最小主链

一个最小的 3D 场景，通常至少有这几层：

1. `Scene3D` 作为 3D 场景根
2. `Camera` 负责把场景渲染出来
3. `DirectionLight` 或其他光源负责提供光照
4. 若干 `Sprite3D` 负责承载具体对象

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

这段代码里最重要的不是 API 数量，而是层级关系已经成立了：

1. 舞台里有一个 `Scene3D`
2. `Scene3D` 下面有相机、光和模型节点
3. 模型节点进入这棵树后，才真正成为场景的一部分

## `Scene3D` 先解决的是“这是一棵 3D 运行树”

`Scene3D` 可以先把它理解成 3D 内容的场景根节点。

它最重要的职责通常是：

1. 作为 3D 节点树的根
2. 承载场景里的相机、灯光、模型、特效等对象
3. 让这些对象进入同一个 3D 更新和渲染上下文

所以这句代码：

```ts
Laya.stage.addChild(scene);
```

表达的不是“把一个普通节点显示出来”，而是“把一整棵 3D 场景树接入当前运行时”。

如果只有单个 `MeshSprite3D`，却没有把它放进某个 `Scene3D`，那它就还没有完整的 3D 场景宿主。

## `Sprite3D` 先解决的是“场景里有什么对象”

在 3D 里，`Sprite3D` 更接近通用节点基类。

可以先把它理解成：

1. 一个能进入 3D 层级树的对象
2. 一个能持有 3D 变换的对象
3. 一个可以继续挂子节点和组件的对象

很多常见 3D 对象，本质上都属于这条链上的节点，例如：

1. `MeshSprite3D`
2. `Camera`
3. `DirectionLight`
4. `SkinnedMeshSprite3D`

也就是说，虽然它们职责不同，但在层级组织上都还是“3D 节点”。

例如：

```ts
const root = new Laya.Sprite3D();
const weapon = new Laya.MeshSprite3D(Laya.PrimitiveMesh.createBox());

scene.addChild(root);
root.addChild(weapon);
```

这里更关键的不是 `weapon` 是什么模型，而是它已经成为 `root` 的子节点。

## `Transform3D` 管的是节点怎么放进空间里

3D 节点进入层级树后，位置、旋转、缩放主要由 `transform` 管。

最常用的就是 `Transform3D` 这一组信息：

1. `position`
2. `rotation` 或旋转方法
3. `scale`

例如：

```ts
box.transform.position = new Laya.Vector3(0, 0.5, 0);
box.transform.setWorldLossyScale(new Laya.Vector3(1, 2, 1));
box.transform.rotate(new Laya.Vector3(0, 45, 0), false, false);
```

可以先建立一个稳定认识：

1. `Sprite3D` 负责“节点存在于树里”
2. `Transform3D` 负责“节点在空间中的状态”
3. 场景层级会继续影响这个空间状态的最终结果

也就是说，很多时候不是对象本身坐标错了，而是父节点的变换一起叠上去了。

## 3D 层级先解决的是“谁跟着谁一起变”

3D 场景里的父子关系，首先不是为了分类，而是为了建立变换继承关系。

最小例子：

```ts
const player = new Laya.Sprite3D();
const weapon = new Laya.MeshSprite3D(Laya.PrimitiveMesh.createBox());

scene.addChild(player);
player.addChild(weapon);

player.transform.position = new Laya.Vector3(2, 0, 0);
weapon.transform.localPosition = new Laya.Vector3(0.5, 1, 0);
```

这里的因果关系是：

1. `weapon` 的局部位置是相对 `player` 来看的
2. `player` 移动时，`weapon` 会一起跟着移动
3. `player` 旋转时，`weapon` 也会跟着改变朝向关系

所以父子层级的第一职责，不是“整理目录”，而是“组织变换关系”。

这也是为什么角色、挂点、武器、特效，通常不会全部直接平铺挂在 `Scene3D` 根下。

## 一个常见的 3D 场景拆法

从运行时组织看，一个场景经常会拆成几层：

1. 场景根 `Scene3D`
2. 环境节点，例如相机和灯光
3. 关卡根节点
4. 动态角色节点
5. 特效或交互节点

例如：

```ts
const levelRoot = new Laya.Sprite3D();
const actorRoot = new Laya.Sprite3D();
const fxRoot = new Laya.Sprite3D();

scene.addChild(levelRoot);
scene.addChild(actorRoot);
scene.addChild(fxRoot);
```

这种拆法的价值通常是：

1. 静态内容和动态内容更容易分开管理
2. 一整组对象需要隐藏、销毁或搬迁时，可以直接操作父节点
3. 后面接 prefab、角色生成器、战斗逻辑时，层级边界会更清楚

## 脚本通常挂在哪些地方

脚本本质上仍然是挂在节点上的组件。

在 3D 里，常见做法通常有三类：

1. 场景级脚本挂在 `Scene3D` 或场景根节点上
2. 对象级脚本挂在角色、相机、机关、特效这些具体 `Sprite3D` 节点上
3. 局部行为脚本挂在真正需要该行为的子节点上

例如：

```ts
class AutoSpin extends Laya.Script {
  onUpdate(): void {
    const owner = this.owner as Laya.Sprite3D;
    owner.transform.rotate(new Laya.Vector3(0, 1, 0), false, false);
  }
}

box.addComponent(AutoSpin);
```

这里更合理的挂法，是直接把脚本挂到要旋转的 `box` 上，而不是挂到完全无关的场景根上再去间接控制它。

可以先记住一个简单判断：

1. 管整场景流程的，挂场景根
2. 管某个对象行为的，挂对象节点
3. 管某个局部挂点或子部件的，挂对应子节点

## 为什么脚本挂载位置会影响理解成本

因为脚本宿主节点，通常就是这段逻辑的作用边界。

例如：

1. 相机跟随脚本，通常挂在相机节点或相机控制根节点上
2. 角色移动脚本，通常挂在角色根节点上
3. 武器挥动或枪口特效脚本，通常挂在武器节点或挂点节点上

这样看层级时，通常能直接读出：

1. 谁在控制谁
2. 谁会跟着谁一起移动
3. 节点禁用或销毁时，哪段行为会一起停掉

## 把这几层关系串起来

如果把一个 3D 场景最常见的组织方式压缩成一句话，可以先记成：

1. 用 `Scene3D` 建立 3D 场景根
2. 用 `Sprite3D` 组织相机、灯光、模型和角色这些节点
3. 用 `Transform3D` 管每个节点在 3D 空间里的位置、旋转和缩放
4. 用父子层级表达变换继承关系
5. 把脚本挂到最贴近它作用边界的节点上

先把这五条关系分清，后面再看 prefab、动画、物理、角色控制和场景加载时，理解成本会低很多。
