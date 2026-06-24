---
title: "引擎启动后的关键入口"
order: 1
---

# 引擎启动后，运行时里有哪些关键入口

当 `Laya.init()` 完成后，引擎才真正进入可用状态。但开发者后续不会一直围着 `Laya.init()` 本身写代码，而是会持续接触几个已经挂到 `Laya` 全局入口上的运行时对象。

最重要的入口通常只有这几类：

1. `Laya.stage`
2. `Laya.loader`
3. `Laya.timer`
4. `Laya.systemTimer`
5. `Laya.physicsTimer`

另外还有两组经常一起出现、但语义不同的全局信息：

1. `Laya.Config`：初始化前配置和全局运行参数
2. `LayaEnv`：当前运行环境状态

理解这几组对象的边界，比背更多 API 更重要。

## 先看最小可用状态

最小初始化代码可以先写成这样：

```ts
await Laya.init({
  designWidth: 1334,
  designHeight: 750,
  scaleMode: "showall"
});

const root = new Laya.Sprite();
Laya.stage.addChild(root);
```

这段代码说明了两件事：

1. `await Laya.init(...)` 之前，`Laya.stage` 这类入口还不能当成已完成初始化的运行时对象使用
2. `await Laya.init(...)` 之后，舞台、计时器、加载器这些全局入口才进入稳定可用状态

所以本文讨论的“运行时关键入口”，前提都是 `Laya.init()` 已经完成。

## `Laya.init()` 到底建起了什么

从引擎源码语义看，`Laya.init()` 至少会先建立这些全局对象：

1. `Laya.systemTimer = new Timer(false)`
2. `Laya.timer = new Timer(false)`
3. `Laya.physicsTimer = new Timer(false)`
4. `Laya.loader = new Loader()`
5. `Laya.stage = new Stage()`

然后才继续做：

1. 平台适配器初始化
2. 主画布与离屏画布创建
3. 渲染设备创建
4. `Stage` 尺寸、缩放、对齐、背景色应用
5. 2D 渲染、输入、声音等模块初始化
6. 如果 3D 模块已经装入运行时，再继续执行 3D 初始化

所以 `Laya.init()` 的直接结果，不是“某个场景已经打开”，而是“引擎的基础运行时入口已经建好”。

## `Laya` 是运行时入口集，不是单一对象

`Laya` 更接近一个全局入口集。

也就是说，后续高频访问通常不是“对 `Laya` 做什么”，而是“通过 `Laya` 取到对应的运行时对象”：

```ts
console.log(Laya.stage.width, Laya.stage.height);
console.log(Laya.timer.currFrame);
```

这类写法的重点是：

1. `Laya.stage` 是舞台对象引用
2. `Laya.loader` 是加载管理器引用
3. `Laya.timer` / `Laya.systemTimer` / `Laya.physicsTimer` 是三个不同职责的时钟引用

它们都共享同一个前提：引擎已经初始化。

## `Laya.stage` 是显示树和输入分发的总入口

`Laya.stage` 是初始化后最常接触的入口。

最小示例：

```ts
await Laya.init({ designWidth: 1334, designHeight: 750 });

const box = new Laya.Sprite();
box.size(200, 200);
Laya.stage.addChild(box);
```

这里的 `Laya.stage` 至少承担几类职责：

1. 作为显示对象树的根
2. 持有舞台尺寸、缩放、对齐、屏幕模式等全局显示状态
3. 作为很多全局输入事件的分发入口
4. 作为没有场景时的默认组件驱动宿主

因此，很多看似无关的能力最后都会落回 `Laya.stage`：

1. 把根节点挂到哪里
2. 以什么尺寸理解屏幕坐标
3. 键盘事件监听挂在哪里
4. 当前焦点对象是谁

如果你发现“对象创建了但画面没显示”，`Laya.stage` 往往是第一批该检查的入口之一。

## `Laya.loader` 是资源进入运行时的总入口

`Laya.loader` 是初始化后第二个高频入口。

最小示例：

```ts
await Laya.init({ designWidth: 1334, designHeight: 750 });

const logo = await Laya.loader.load("resources/ui/logo.png");
```

它的职责不是“只加载图片”，而是：

1. 根据 URL 和类型信息选择对应 loader
2. 处理缓存
3. 处理并发去重
4. 把原始资源解析成运行时对象或资源对象

这意味着，`Laya.loader` 是很多运行时能力能否成立的前置入口。

例如：

1. 图片能否变成 `Texture`
2. 场景资源能否变成层级对象
3. prefab 能否先被加载，再进一步实例化

所以“资源 API 写对了但结果不可用”，不一定是你调用姿势错了，也可能是对应的资源类型 loader 根本没有注册成功。

## 三个 Timer 不是一回事

很多人只记住 `Laya.timer`，但 `LayaAir 3.4` 在全局上实际暴露了三个计时器：

1. `Laya.timer`
2. `Laya.systemTimer`
3. `Laya.physicsTimer`

它们名字相近，但职责边界不同。

## `Laya.timer` 是主游戏时钟

`Laya.timer` 是开发者最常直接使用的时钟。

最小示例：

```ts
await Laya.init({ designWidth: 1334, designHeight: 750 });

Laya.timer.frameLoop(1, null, () => {
  // 每帧执行一次游戏逻辑
});
```

它主要承担：

1. 游戏主逻辑的帧循环
2. 场景、动画、缓动等常规运行时更新节奏
3. 受游戏时间尺度影响的时序控制

如果你写的是普通游戏逻辑更新、UI 轮询、简单节奏驱动，优先想到的通常应该是 `Laya.timer`。

## `Laya.systemTimer` 更偏引擎和系统级时序

`Laya.systemTimer` 在源码语义里更偏系统时钟。

最小示例：

```ts
await Laya.init({ designWidth: 1334, designHeight: 750 });

Laya.systemTimer.loop(1000, null, () => {
  // 更适合系统级轮询或不希望混入游戏时间语义的逻辑
});
```

从引擎内部使用情况看，它常被用于：

1. 平台适配层轮询
2. 字体、输入、缓存等系统辅助逻辑
3. 一些更靠近运行时基础设施的调度

对日常业务代码来说，不是完全不能用，而是不要把它和主游戏时钟混成同一个概念。

## `Laya.physicsTimer` 用于物理更新节奏

`Laya.physicsTimer` 的命名已经很直接，它对应物理相关更新。

最小示例：

```ts
await Laya.init({ designWidth: 1334, designHeight: 750 });

Laya.physicsTimer.frameLoop(1, null, () => {
  // 物理相关的逐帧处理
});
```

从源码实现看，2D 物理系统会把自己的更新挂到这个时钟上。

这背后的实际意义是：

1. 主游戏逻辑更新不等于物理更新
2. 物理节奏有独立入口，不应该随手混到任意计时器里

如果你后面在物理组件、碰撞回调、物理步进问题上继续深入，这条边界会很重要。

## `Config` 影响的是初始化前和全局运行方式

`Laya.Config` 不是运行时入口对象，但它会直接影响这些入口最终如何被创建和驱动。

最关键的一点只有一句话：需要修改时，优先在 `Laya.init()` 之前设置。

```ts
Laya.Config.useWebGL2 = true;
Laya.Config.isAntialias = false;
Laya.Config.fixedFrames = true;

await Laya.init({
  designWidth: 1334,
  designHeight: 750,
  backgroundColor: "#000000"
});
```

在这篇主题里，最值得先记住的是它影响哪些范围：

1. 渲染后端和画布相关能力，例如 `useWebGL2`、`isAntialias`、`isAlpha`
2. 运行节奏相关能力，例如 `FPS`、`fixedFrames`
3. 一些全局资源和平台行为，例如字体映射、音频缓存、资源释放默认策略

最常见误区是：

1. 先 `await Laya.init()`
2. 再去改 `Laya.Config.useWebGL2`、`Laya.Config.isAntialias` 之类初始化期参数
3. 最后发现结果没有按预期变化

原因不是 API 失效，而是时机错了。

## `LayaEnv` 描述的是当前运行环境，不是配置项

`LayaEnv` 和 `Laya.Config` 很容易被混淆，因为它们看起来都像“全局静态信息”。

但语义完全不同：

1. `Laya.Config` 是你设置给引擎的运行参数
2. `LayaEnv` 是引擎当前所处环境的状态描述

最常见的几个环境标记有：

1. `LayaEnv.isPreview`
2. `LayaEnv.isEditor`
3. `LayaEnv.isPlaying`
4. `LayaEnv.isConch`
5. `LayaEnv.isModernAPIs`

例如：

```ts
if (LayaEnv.isPreview) {
  console.log("当前运行在预览环境");
}
```

在这篇主题里，只需要先抓住它们的大致用途：

1. `isPreview` 区分预览与正式产品环境
2. `isEditor` / `isPlaying` 区分编辑器上下文和播放状态
3. `isConch` 区分是否运行在原生平台
4. `isModernAPIs` 区分是否使用现代图形 API 后端

因此，`LayaEnv` 更适合用来分支判断当前运行上下文，而不是拿来替代初始化配置。

## 为什么有时“代码写对了，功能还是不可用”

这类问题在 `LayaAir 3.4` 里很常见，而且往往不是语法错误，而是运行时入口背后的模块没有准备好。

最常见的原因有两类。

### 1. 对应模块没有完成注册或初始化

引擎不是把所有功能都硬编码进 `Laya.init()` 本体里。

很多能力依赖模块侧的注册、副作用导入或初始化回调，例如：

1. 某些资源类型需要先注册对应 loader
2. 某些类需要先注册到类表里，序列化资源才能正确还原
3. 某些子系统会通过 `Laya.addInitCallback()` 或 `Laya.addAfterInitCallback()` 接入初始化链

这也是为什么“全局 `Laya` 对象已经可访问”不等于“所有模块都已经可用”。

举个最短的理解方式：

```ts
await Laya.init({ designWidth: 1334, designHeight: 750 });

// 代码形式没问题，但某类资源或功能是否真的可用，
// 还取决于对应模块是否已经把 loader、类注册或初始化逻辑接进运行时。
```

这里真正要记住的是因果关系：

1. API 看得见
2. 不代表模块一定完成注册
3. 模块没注册，对应能力就可能不可用

### 2. 用错了初始化前后边界

另一类问题不是模块缺失，而是时机错误：

1. 需要初始化前设置的 `Config`，被放到了初始化后
2. 需要 `await Laya.init()` 之后访问的入口，被提前访问
3. 需要运行环境判断的逻辑，被错误地当成配置处理

这三种问题的外观经常很像“功能失灵”，但根因并不一样。

## 一段更接近真实使用的最小示例

下面这段代码把本文提到的几个核心入口放在一起：

```ts
Laya.Config.fixedFrames = true;

await Laya.init({
  designWidth: 1334,
  designHeight: 750,
  scaleMode: "showall"
});

const root = new Laya.Sprite();
Laya.stage.addChild(root);

const logo = await Laya.loader.load("resources/ui/logo.png");

Laya.timer.frameLoop(1, root, () => {
  root.rotation += 1;
});

if (LayaEnv.isPreview) {
  console.log("preview mode");
}
```

这里每一块的角色分别是：

1. `Laya.Config.fixedFrames`：初始化前决定运行节奏策略
2. `Laya.init(...)`：建立运行时基础设施
3. `Laya.stage`：承载显示树
4. `Laya.loader`：把资源带进运行时
5. `Laya.timer`：驱动主逻辑更新
6. `LayaEnv.isPreview`：识别当前环境

如果把这几个入口的边界弄清楚，后面的舞台适配、资源加载、脚本生命周期、物理和场景问题都会更容易理解。

## 最容易混淆的五件事

1. `Laya.init()` 建立的是运行时基础设施，不是直接打开业务内容
2. `Laya.stage`、`Laya.loader`、`Laya.timer` 是初始化后的长期入口
3. `Laya.timer`、`Laya.systemTimer`、`Laya.physicsTimer` 不是同一个时钟
4. `Laya.Config` 是配置，`LayaEnv` 是环境状态
5. API 存在不等于对应模块已经完成注册和初始化

## 结论

对 `LayaAir 3.4` 来说，`Laya.init()` 之后最值得先掌握的不是更多零散 API，而是几条稳定的运行时入口关系：

1. 用 `Laya.stage` 进入舞台和显示树
2. 用 `Laya.loader` 让资源进入运行时
3. 用 `Laya.timer` / `Laya.systemTimer` / `Laya.physicsTimer` 区分不同时序职责
4. 用 `Laya.Config` 把握初始化前配置边界
5. 用 `LayaEnv` 判断当前运行环境

把这五条边界先建立起来，后面再看场景、组件、生命周期和资源系统时，理解成本会低很多。
