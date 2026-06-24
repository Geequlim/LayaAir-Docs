# 一个 Laya 项目是如何启动的

`LayaAir 3.4` 里的“项目启动”至少要分成两层看：

1. 引擎启动
2. 项目入口启动

很多混淆都来自把这两层当成一件事。

`Laya.init()` 负责把引擎运行时建起来。场景打开、脚本执行、进入首页，这些属于项目自己的入口逻辑。

## 先分清两层启动

最小链路通常是这样：

```ts
await Laya.init({
  designWidth: 1334,
  designHeight: 750,
  scaleMode: "showall"
});

await Laya.Scene.open("scenes/Main.ls");
```

这段代码里：

1. `Laya.init(...)` 是引擎启动
2. `Laya.Scene.open(...)` 是项目入口进入第一个场景

如果只执行 `Laya.init()`，引擎已经可用，但你的项目还没有真正进入业务内容。

如果只想象“打开了一个场景，所以项目启动了”，也不准确，因为场景打开依赖的前提是引擎已经完成初始化。

## 引擎启动时做了什么

从引擎入口语义看，`Laya.init()` 做的是运行时基础设施初始化，而不是“打开某个场景”。

它会建立至少这些关键对象：

1. `Laya.stage`
2. `Laya.loader`
3. `Laya.timer`
4. `Laya.systemTimer`
5. `Laya.physicsTimer`

同时它还会继续完成：

1. 平台适配器初始化
2. 主画布与离屏画布创建
3. 渲染设备创建
4. `Stage` 创建并应用舞台配置
5. 2D 渲染、输入、声音等运行时模块初始化
6. 如果已装配 3D 模块，再继续执行 3D 初始化

所以 `Laya.init()` 的结果不是“首页出现了”，而是“引擎已经具备创建舞台、加载资源、驱动更新循环的能力”。

## 初始化前就该定下来的配置

有些配置必须放在 `Laya.init()` 之前，因为它们会影响初始化过程本身。

```ts
Laya.Config.useWebGL2 = true;
Laya.Config.isAntialias = false;
Laya.Config.fixedFrames = true;

await Laya.init({
  designWidth: 1334,
  designHeight: 750,
  scaleMode: "showall",
  screenMode: "none",
  backgroundColor: "#000000"
});

await Laya.Scene.open("scenes/Main.ls");
```

这里有两类配置：

1. `Laya.Config.*` 影响引擎初始化和全局运行方式
2. `Laya.init({...})` 里的 `stageConfig` 直接参与 `Stage` 创建

这也是一个常见误区：在 `Laya.init()` 之后再改初始化前配置，往往不会得到预期结果。

## 项目入口常见有哪几种形式

引擎启动完成后，项目一般会用以下几种方式进入内容。

### 1. 直接创建代码入口对象

适合纯代码组织的最小项目。

```ts
await Laya.init({ designWidth: 1334, designHeight: 750 });

const root = new Laya.Sprite();
Laya.stage.addChild(root);
```

这时项目入口就是这段脚本本身。它不依赖启动场景。

### 2. 打开启动场景

这是标准项目里最常见的项目入口形式。

```ts
await Laya.init({ designWidth: 1334, designHeight: 750 });
await Laya.Scene.open("scenes/Main.ls");
```

这里的关键点不是“场景文件天生就是入口”，而是“你的入口脚本选择把第一个运行内容交给 `Laya.Scene.open()`”。

从运行时语义看，`Laya.Scene.open(url)` 先加载场景资源，再在内部调用场景实例的 `scene.open()`。真正把场景挂进运行时树的，是实例级的 `open()` 过程：

1. 按需要关闭其他场景
2. 把当前场景加入场景根节点
3. 如果场景带有 3D 部分，再把对应 `Scene3D` 加到舞台
4. 调用场景的 `onOpened(param)`

也就是说，打开场景是项目入口层动作，不是引擎初始化动作。

### 3. 先执行入口脚本，再由脚本决定进入哪个场景

这更接近真实项目。

```ts
await Laya.init({ designWidth: 1334, designHeight: 750 });

const firstScene = needTutorial ? "scenes/Tutorial.ls" : "scenes/Main.ls";
await Laya.Scene.open(firstScene);
```

这时真正的入口不是某个固定场景文件，而是“决定首个运行内容的那段脚本”。

所以不要把示例工程里的某个 `Main.ts`、`GameConfig.ts` 或固定文件名，当成标准项目必须遵守的入口规范。规范的核心是启动链，而不是样例文件名。

## 启动场景和启动脚本分别负责什么

可以把两者的职责简单分开：

1. 启动脚本负责初始化顺序、运行配置、首个内容分发
2. 启动场景负责承载第一个可见的运行时对象树

启动脚本通常处理这些事：

1. 设置 `Laya.Config`
2. 调用 `Laya.init()`
3. 决定是否打开统计、调试、日志或错误捕获
4. 决定进入哪个场景，或直接创建对象

启动场景通常处理这些事：

1. 提供首页或首屏的节点层级
2. 挂载脚本组件
3. 持有它依赖的场景资源
4. 在 `onOpened` 之后接收入口参数

把这两类职责分开后，很多问题会更容易定位：

1. 画面没出来，先看是否完成 `Laya.init()`，以及是否真的进入了入口逻辑
2. 场景打开了但行为不对，再看场景脚本、参数和资源依赖

## 预览、运行、构建看到的入口行为为什么可能不同

同一个项目，在不同执行方式下，读者看到的“入口”不一定完全一致。

### 预览

预览模式下，运行时环境通常会带有“当前是预览”的上下文。`LayaEnv.isPreview` 可以帮助区分这类环境。

这意味着：

1. 某些调试逻辑可能只在预览时打开
2. 预览工具可能直接让你进入某个场景或当前测试内容
3. 你看到的第一个画面，不一定等于正式发布时的首页链路

因此，不能假设“当前场景预览”一定完整走过了你的标准启动脚本。

### 正式运行

正式运行时，更应该以你写下的启动链为准：

1. 先设置初始化前配置
2. 再执行 `Laya.init()`
3. 再进入首个场景或首个代码入口

如果你依赖某些只在预览环境成立的行为，发布后就可能出现偏差。

### 构建

构建本身不是运行入口，它只是把项目整理成目标平台需要的产物。

但构建会影响你最终看到的入口行为，因为它会改变至少两件事：

1. 运行环境从预览环境切到正式环境
2. 资源是否可被正确找到，取决于发布产物里的路径和收集规则

所以“构建后首页打不开”未必是 `Laya.init()` 有问题，也可能是项目入口脚本引用的场景或资源没有按预期进入产物。

## 一条更接近真实项目的启动链

把前面的关系合起来，可以把标准启动链理解成：

```ts
async function bootstrap() {
  Laya.Config.useWebGL2 = true;
  Laya.Config.fixedFrames = true;

  await Laya.init({
    designWidth: 1334,
    designHeight: 750,
    scaleMode: "showall",
    backgroundColor: "#1e1e1e"
  });

  await Laya.Scene.open("scenes/Boot.ls");
}

bootstrap();
```

其中每一步的角色分别是：

1. `Laya.Config.*` 决定初始化前的全局运行参数
2. `Laya.init()` 建立引擎运行时
3. `Laya.Scene.open("scenes/Boot.ls")` 让项目进入第一个场景

如果后面还要切换到 `Main.ls`、`Battle.ls` 或别的内容，那已经属于场景流转，不属于引擎启动。

## 最容易混淆的四件事

1. `Laya.init()` 是引擎启动，不是打开首页
2. 打开场景是项目入口层动作，不是初始化引擎
3. 当前场景预览不一定完整经过标准启动脚本
4. 示例工程的入口文件名不是标准项目规范本身

理解这条边界后，再看场景、脚本、资源和构建问题，会清楚很多。
