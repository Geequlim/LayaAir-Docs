---
title: "预制体"
order: 2
---

# 预制体适合解决哪些问题

预制体最适合解决的，不是“少写几行 `new`”，而是“把会重复出现的一整段节点层级和默认配置，收敛成一个可复用资源”。

如果不先把这个边界讲清楚，就很容易把预制体误解成：

1. 一种特殊节点。
2. 一个已经在运行时存在的对象。
3. 只是给 UI 用的模板。

这些都不准确。

## 先用一句话理解预制体

预制体更接近：

“一份可重复实例化的层级资源定义”。

它保存的是：

1. 节点类型和父子层级。
2. 默认属性值。
3. 组件和脚本挂载关系。
4. 对贴图、材质、子预制体等资源的引用。

它本身不是已经活着的节点实例。

## 它和普通节点有什么区别

普通节点，比如 `new Laya.Sprite()`、`new Laya.Box()`、`new Laya.Scene3D()`，创建出来后就是当前这一份运行时实例。

例如：

```ts
const sp = new Laya.Sprite();
sp.pos(100, 100);
Laya.stage.addChild(sp);
```

这里的 `sp` 已经是运行时对象。你改它的位置、销毁它、把它移出树，处理的都是这一个实例。

预制体不是这回事。

例如：

```ts
const prefab = await Laya.loader.load("resources/prefab/Enemy.lh") as Laya.Prefab;
```

这时拿到的是“可实例化资源”，不是已经出现在场景里的敌人节点。

只有继续创建实例：

```ts
const enemy = prefab.create() as Laya.Sprite;
Laya.stage.addChild(enemy);
```

它才变成真正参与运行时的节点树。

所以两者最核心的区别是：

1. 普通节点是直接创建当前实例。
2. 预制体先拿到资源定义，再按这份定义创建实例。

## 预制体最适合解决哪些问题

### 1. 同一类层级结构会重复出现

这是预制体最典型的使用场景。

例如同一种敌人、同一种按钮单元、同一种掉落物、同一种 3D 装饰物，会重复很多次出现。

这类对象的共同点通常是：

1. 节点层级基本一致。
2. 组件组合基本一致。
3. 默认资源引用基本一致。
4. 每次只改少量参数。

这时比起每次都手写一整段创建代码，把它做成一个 prefab 更稳。

### 2. 结构想复用，参数想局部变化

很多对象不是“完全一样”，而是“骨架一样，少量值不同”。

例如：

1. 同一种血条 prefab，不同实例显示不同血量。
2. 同一种敌人 prefab，不同实例速度不同。
3. 同一种奖励面板 prefab，不同实例填不同文案或图标。

最小例子：

```ts
const prefab = await Laya.loader.load("resources/prefab/Hud.lh") as Laya.Prefab;

const hud1 = prefab.create() as Laya.Sprite;
const hud2 = prefab.create() as Laya.Sprite;

hud1.pos(0, 0);
hud2.pos(0, 120);

Laya.stage.addChild(hud1);
Laya.stage.addChild(hud2);
```

这里复用的是层级结构，变化的是实例级状态。

### 3. 资源引用关系希望跟着结构一起走

预制体的价值不只是“省代码”，还包括把结构和依赖资源一起组织起来。

例如一个敌人 prefab 可能同时带出：

1. 自己的贴图。
2. 挂载的脚本。
3. 子节点上的特效。
4. 子 prefab。

这样业务代码只需要关心入口资源：

```ts
const prefab = await Laya.loader.load("resources/prefab/Enemy.lh") as Laya.Prefab;
const enemy = prefab.create();
```

而不用在每次创建时都重新手工拼装完整依赖链。

### 4. 美术、交互、层级组织需要稳定协作边界

当一个对象的重点不只是逻辑，还包括：

1. 节点命名。
2. 子节点层级。
3. 资源引用。
4. 默认组件配置。

那么 prefab 往往比纯代码创建更适合长期维护。

因为它表达的是“这个对象默认应该长什么样”，而不是“运行时临时怎么拼出来”。

### 5. 同一套组织方式要同时用于 2D、UI、3D

预制体不是只给 UI 用。

只要对象本质上是“可复用的层级资源”，它就可以是 2D、UI 或 3D。

它们的共同点都在这里：

1. 都先是资源定义。
2. 都要在运行时通过创建步骤变成实例。
3. 都适合复用稳定结构，而不是替代所有动态创建。

## `load()` 和 `create()` 分别解决什么

这是理解 prefab 时最重要的一条边界。

### `load()` 解决“把 prefab 资源准备好”

```ts
const prefab = await Laya.loader.load("resources/prefab/Enemy.lh") as Laya.Prefab;
```

这里完成的是：

1. 加载 prefab 数据。
2. 解析它依赖的资源。
3. 得到一个可重复实例化的资源对象。

这时还没有敌人节点加入舞台。

### `create()` 解决“把资源定义变成当前实例”

```ts
const enemy = prefab.create() as Laya.Sprite;
Laya.stage.addChild(enemy);
```

这里完成的是：

1. 按 prefab 数据创建节点树。
2. 恢复组件、脚本和引用关系。
3. 得到这一份实际运行时对象。

所以对 prefab 来说：

1. `load()` 拿到的是模板资源。
2. `create()` 拿到的是活的实例。

## 为什么说它是“可复用层级资源”，不是“共享同一个节点”

因为同一个 prefab 可以创建很多次，而每次创建出来的实例彼此独立。

例如：

```ts
const prefab = await Laya.loader.load("resources/prefab/Coin.lh") as Laya.Prefab;

const a = prefab.create() as Laya.Sprite;
const b = prefab.create() as Laya.Sprite;

a.x = 100;
b.x = 200;
```

改 `a.x` 不会自动改 `b.x`。

因为共享的是“创建依据”，不是“运行中的同一个节点对象”。

## 复用和覆盖的边界在哪里

预制体适合复用“公共默认值”，实例适合承接“当前这一次的变化”。

可以先用一个简单判断：

### 适合放进 prefab 的内容

1. 大多数实例都相同的层级结构。
2. 大多数实例都相同的组件组合。
3. 大多数实例都相同的默认贴图、材质、尺寸、锚点。
4. 需要被重复创建的默认节点命名和引用关系。

### 适合留到实例阶段再改的内容

1. 当前实例的位置。
2. 当前实例的数值状态。
3. 当前实例的文案、图标、血量、速度。
4. 只对这一份实例生效的临时逻辑。

最需要记住的一点是：

运行时对实例做的修改，默认只影响这一个实例，不会反向改写 prefab 资源本身，也不会自动影响其他实例。

例如：

```ts
const prefab = await Laya.loader.load("resources/prefab/Enemy.lh") as Laya.Prefab;
const enemy = prefab.create() as Laya.Sprite;

enemy.pos(300, 200);
enemy.scale(1.5, 1.5);
```

这里改的是这一个 `enemy`，不是所有由该 prefab 创建出来的对象。

## 什么时候 prefab 不合适

预制体不是所有对象创建问题的默认答案。

下面几类情况，直接代码创建通常更合适。

### 1. 对象只会出现一次，而且结构很简单

例如只创建一个纯代码调试标记：

```ts
const marker = new Laya.Sprite();
marker.graphics.drawCircle(0, 0, 10, "#ff0000");
Laya.stage.addChild(marker);
```

如果只是一次性、结构很薄，做成 prefab 反而增加资源管理成本。

### 2. 结构本身主要由运行时数据决定

例如排行榜条目数量、技能节点树、程序化地图块，节点数量和层级常常要根据数据动态生成。

这类情况的核心问题不是“复用固定层级”，而是“按数据构造结构”，所以代码创建往往更直接。

### 3. 同类对象差异大到已经没有稳定公共骨架

如果所谓“同一类对象”只是名字像，实际层级、组件、资源依赖都差很多，那么继续强行共用一个 prefab，通常只会让覆盖越来越混乱。

更合适的做法通常是：

1. 拆成多个 prefab。
2. 或者改成由代码分别创建。

### 4. 你真正要复用的是行为，不是层级

如果复用重点在算法、状态机、通用逻辑，而不是节点树本身，那么应该优先抽代码，而不是先抽 prefab。

prefab 解决的是“结构复用”，不是所有层面的复用。

## prefab 和直接代码创建，不是谁替代谁

更稳妥的理解方式是：

1. 直接代码创建，适合临时、轻量、强动态、强计算生成的对象。
2. prefab 创建，适合稳定、重复、带层级和默认资源配置的对象。

真实项目里，两者经常一起用。

例如：

1. 用 prefab 创建一个敌人基础结构。
2. 再用代码给这一份敌人实例写入出生点、速度、目标。

最小例子：

```ts
const prefab = await Laya.loader.load("resources/prefab/Enemy.lh") as Laya.Prefab;
const enemy = prefab.create() as Laya.Sprite;

enemy.pos(spawnX, spawnY);
Laya.stage.addChild(enemy);
```

这里 prefab 负责“敌人默认长什么样”，代码负责“这次把它放到哪里、给它什么状态”。

## 和场景的边界也要分开

场景和 prefab 都是层级资源，但职责不一样。

可以先这样记：

1. 场景更像一段较完整的运行内容入口。
2. prefab 更像场景里的可重复局部单元。

通常会直接打开场景：

```ts
await Laya.Scene.open("scenes/Main.ls");
```

而 prefab 更常见的路径是：

```ts
const prefab = await Laya.loader.load("resources/prefab/Enemy.lh") as Laya.Prefab;
const enemy = prefab.create();
```

所以不要把 prefab 理解成“小号场景”，也不要把场景理解成“只能创建一次的大 prefab”。

## 常见误区

### 误区 1：prefab 加载完就已经出现在运行时里

不是。`load()` 只是拿到可实例化资源，`create()` 之后才有实际节点实例。

### 误区 2：prefab 就是某个节点对象本身

不是。prefab 是创建依据，节点实例是创建结果。

### 误区 3：改一个实例，就等于改了 prefab

不是。运行时改实例，通常只影响当前实例。

### 误区 4：能用 prefab 的地方就不该写 `new`

也不是。简单对象、一次性对象、强动态对象，直接代码创建更合适。

## 结论

预制体最适合解决的是“重复层级结构的复用问题”。

可以把它稳定地理解成四句话：

1. prefab 是可重复实例化的层级资源，不是普通节点实例。
2. `load()` 拿到的是 prefab 资源，`create()` 拿到的才是运行时对象。
3. prefab 适合沉淀公共结构和默认配置，实例适合承接当前这一次的局部变化。
4. 结构稳定且会重复出现时优先考虑 prefab，结构简单或强动态时直接代码创建通常更合适。
