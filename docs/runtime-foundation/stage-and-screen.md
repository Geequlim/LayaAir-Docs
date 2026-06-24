---
title: "舞台与屏幕适配"
order: 2
---

# 舞台是如何管理屏幕和坐标的

`Laya.init()` 完成后，`Laya.stage` 就成了运行时里和屏幕最直接相关的总入口。

它不只是显示树根节点，还同时决定这些事：

1. 项目用什么设计分辨率理解画面
2. 真实屏幕和设计尺寸之间怎么适配
3. 横屏、竖屏和画布对齐怎么处理
4. 鼠标、触摸这类输入最后落到什么舞台坐标

很多“为什么位置不对”“为什么两端留黑边”“为什么点击坐标和视觉位置不一样”的问题，本质上都和 `Stage` 的屏幕管理方式有关。

## 先看最小主链

最常见的舞台配置会直接放进 `Laya.init()`：

```ts
await Laya.init({
  designWidth: 1334,
  designHeight: 750,
  scaleMode: "showall",
  screenMode: "none",
  alignH: "center",
  alignV: "middle",
  backgroundColor: "#000000"
});
```

这里至少有两层含义：

1. `designWidth` 和 `designHeight` 定义的是项目要采用的设计坐标系
2. `scaleMode`、`screenMode`、`alignH`、`alignV` 决定这套坐标系怎样映射到真实设备屏幕

所以舞台适配的核心，不是“屏幕有多大”，而是“设计坐标和真实屏幕之间怎么换算”。

## `Laya.stage` 管的是哪一层坐标

先看一个最短例子：

```ts
await Laya.init({ designWidth: 1334, designHeight: 750, scaleMode: "showall" });

const box = new Laya.Sprite();
box.pos(100, 100);
box.size(200, 100);
box.graphics.drawRect(0, 0, 200, 100, "#ff6b6b");
Laya.stage.addChild(box);
```

这里的 `100, 100` 不是“浏览器窗口左上角往右下 100 像素”，而是“舞台坐标系里的 `x=100`、`y=100`”。

这点很关键：

1. 开发者平时写的大多数 `x`、`y`、`width`、`height`，都是舞台坐标
2. 舞台坐标通常优先受设计分辨率和缩放模式影响
3. 真实屏幕像素、浏览器 CSS 像素、设备 DPR，不是同一层概念

因此，先分清“舞台坐标”和“物理屏幕尺寸”，后面看适配才不会乱。

## 设计分辨率不等于真实屏幕分辨率

最容易混淆的一点是：

`designWidth` 和 `designHeight` 定的是设计尺寸，不是设备实际像素尺寸。

例如：

```ts
await Laya.init({
  designWidth: 1334,
  designHeight: 750,
  scaleMode: "showall"
});

console.log(Laya.stage.designWidth, Laya.stage.designHeight);
console.log(Laya.stage.width, Laya.stage.height);
```

这里要分开看：

1. `Laya.stage.designWidth` / `designHeight` 是初始化时给定的设计基准
2. `Laya.stage.width` / `height` 是当前适配后，运行时真正采用的舞台大小

在某些缩放模式下，这两组值会相同。

在另一些模式下，`stage.width` 或 `stage.height` 会被重新计算。

所以不要默认认为：

1. 设计宽高永远等于舞台宽高
2. 舞台宽高永远等于设备屏幕宽高

这三者经常不是一回事。

## 缩放模式决定舞台尺寸怎么变

`scaleMode` 是理解屏幕适配时最重要的入口。

## `noscale`：舞台保持设计宽高，不做适配缩放

```ts
await Laya.init({
  designWidth: 1334,
  designHeight: 750,
  scaleMode: "noscale"
});
```

这个模式下：

1. 舞台宽高就是设计宽高
2. 引擎不会为了适配屏幕而按比例缩放内容
3. 设备比设计尺寸大时，可能看到留白
4. 设备比设计尺寸小时，可能看到内容超出可视区域

它更像是“坚持设计尺寸原样显示”。

## `showall`：完整保留设计内容，但可能留边

```ts
await Laya.init({
  designWidth: 1334,
  designHeight: 750,
  scaleMode: "showall",
  alignH: "center",
  alignV: "middle"
});
```

这个模式下：

1. 舞台坐标系通常仍按设计宽高理解
2. 整个舞台会等比缩放，确保设计内容全部可见
3. 屏幕宽高比和设计比不一致时，会出现上下或左右空白
4. `alignH` 和 `alignV` 决定这些空白分布到哪边

这类项目里最常见的感受是：

1. 坐标比较稳定
2. 视觉上容易出现黑边或留白

如果你更关心“设计稿里的内容必须完整出现”，这是最常见的选择。

## `noborder`：铺满屏幕，但可能裁切

```ts
await Laya.init({
  designWidth: 1334,
  designHeight: 750,
  scaleMode: "noborder"
});
```

这个模式和 `showall` 一样也是等比缩放，但目标不同：

1. 它优先让内容铺满屏幕
2. 因为仍然保持比例，所以更长的一边可能超出屏幕
3. 超出部分会被裁掉

它的典型结果是：

1. 没有留边
2. 但设计稿边缘内容不一定都看得见

所以如果按钮、标题、血条被放在设计稿边缘，`noborder` 下就更容易出问题。

## `full`：舞台直接等于屏幕大小

```ts
await Laya.init({
  designWidth: 1334,
  designHeight: 750,
  scaleMode: "full"
});
```

这个模式要特别注意，因为它和前几种不一样。

它的关键不是“按设计尺寸缩放”，而是：

1. `Laya.stage.width` 直接变成当前屏幕宽度
2. `Laya.stage.height` 直接变成当前屏幕高度
3. 舞台坐标系本身就跟着屏幕变化

因此在 `full` 下：

1. 不同设备上的舞台宽高可能都不一样
2. 绝对坐标布局会更依赖实时计算
3. 更适合你自己按当前屏幕做动态排版

它不是“把 1334x750 缩放到全屏”，而是“让舞台本身变成当前屏幕尺寸”。

## `fixedwidth` 和 `fixedheight`：固定一边，另一边动态变化

```ts
await Laya.init({
  designWidth: 1334,
  designHeight: 750,
  scaleMode: "fixedwidth"
});
```

`fixedwidth` 的含义是：

1. 舞台宽度保持设计宽度
2. 舞台高度根据当前屏幕比例重新计算

对应地，`fixedheight` 则相反：

1. 舞台高度保持设计高度
2. 舞台宽度根据当前屏幕比例重新计算

这两种模式很适合“主布局必须守住一边，另一边允许扩展或收缩”的项目。

例如：

```ts
console.log(Laya.stage.width, Laya.stage.height);
```

在 `fixedwidth` 下，更应该预期的是：

1. `stage.width` 基本稳定
2. `stage.height` 可能随设备变化

而不是两者都固定。

## `fixedauto`：由引擎自动在两种固定模式间选择

```ts
await Laya.init({
  designWidth: 1334,
  designHeight: 750,
  scaleMode: "fixedauto"
});
```

它的语义可以简单理解成：

1. 如果当前屏幕更“窄”，就按 `fixedwidth` 处理
2. 如果当前屏幕更“宽”，就按 `fixedheight` 处理

所以这个模式的重点不是“尺寸固定”，而是“固定哪一边由当前屏幕比例决定”。

这意味着：

1. 某些设备上变的是高度
2. 另一些设备上变的是宽度

如果你发现同一段布局代码在不同设备上，变化方向都不一样，先看看是不是用了 `fixedauto`。

## 哪些模式最直接影响运行时坐标

可以把前面的模式按“舞台坐标系是否变化”简单分成两类。

第一类：舞台坐标通常保持设计宽高语义

1. `noscale`
2. `showall`
3. `noborder`

第二类：舞台宽高本身会变

1. `full`
2. `fixedwidth`
3. `fixedheight`
4. `fixedauto`

这条边界非常重要。

如果你的代码写的是：

```ts
button.pos(Laya.stage.width - 220, Laya.stage.height - 100);
```

那么：

1. 在第一类模式里，它更像是基于设计坐标摆放
2. 在第二类模式里，它会跟着当前设备实际适配后的舞台宽高变化

也就是说，同样是 `Laya.stage.width`，不同缩放模式下，它代表的运行时意义并不一样。

## `screenMode` 处理的是横竖屏方向，不是缩放策略

`screenMode` 很容易和 `scaleMode` 混在一起，但两者解决的是不同问题。

最小例子：

```ts
await Laya.init({
  designWidth: 1334,
  designHeight: 750,
  scaleMode: "showall",
  screenMode: "horizontal"
});
```

这里的 `screenMode` 只是在表达：

1. `none`：不强制横竖方向
2. `horizontal`：按横屏语义适配
3. `vertical`：按竖屏语义适配

它关心的是屏幕方向是否需要旋转。

`scaleMode` 关心的则是旋转之后，舞台与屏幕怎么换算。

所以不要把这两件事混成一句“横屏适配”。更准确地说：

1. `screenMode` 先决定是否按横屏或竖屏语义处理屏幕
2. `scaleMode` 再决定采用哪种尺寸映射方式

## 对齐只移动画布，不改舞台坐标原点语义

`alignH` 和 `alignV` 经常被误解成“改变舞台坐标原点”。

更准确地说，它们主要处理的是：

1. 当适配后出现多余空白时，画布贴左、居中还是贴右
2. 画布贴上、居中还是贴下

例如：

```ts
await Laya.init({
  designWidth: 1334,
  designHeight: 750,
  scaleMode: "showall",
  alignH: "right",
  alignV: "bottom"
});
```

这时更应该理解成：

1. 舞台内容还是那套舞台坐标
2. 只是整块画布被放到了屏幕的右下位置

因此，对齐最直接影响的是“内容在物理屏幕上的停靠位置”，而不是“你写代码时使用的对象局部坐标规则”。

## 为什么输入坐标看起来能自动对上

很多人第一次接触适配时会疑惑：

画面明明被缩放、留边甚至旋转了，为什么 `Laya.stage.mouseX`、`mouseY` 还能直接用？

原因是 `Stage` 会把输入位置换算回当前舞台坐标系。

最小例子：

```ts
Laya.stage.on(Laya.Event.CLICK, null, () => {
  console.log(Laya.stage.mouseX, Laya.stage.mouseY);
});
```

这里输出的不是原始屏幕像素，而是舞台坐标。

所以在大多数情况下：

1. 你看到的点击位置会和当前舞台坐标系对应
2. 即使画面被整体缩放，输入通常仍然能落到正确对象上

这也是为什么 `showall` 留边后，按钮点击通常仍然是对的。因为输入系统关心的是“换算后的舞台坐标”，不是“原始物理像素”。

## 什么时候该监听 `resize`

并不是所有缩放模式都要求你在窗口变化时重排布局。

但只要你的布局依赖 `Laya.stage.width`、`Laya.stage.height`，就应该重视 `resize`。

例如：

```ts
function layout() {
  panel.pos(Laya.stage.width - panel.width - 20, 20);
}

Laya.stage.on(Laya.Event.RESIZE, null, layout);
layout();
```

这在下面几类模式里尤其重要：

1. `full`
2. `fixedwidth`
3. `fixedheight`
4. `fixedauto`

因为这些模式下，舞台尺寸本身更可能跟着设备变化。

## 一段更接近真实项目的最小示例

```ts
await Laya.init({
  designWidth: 1334,
  designHeight: 750,
  scaleMode: "showall",
  screenMode: "horizontal",
  alignH: "center",
  alignV: "middle",
  backgroundColor: "#1e1e1e"
});

const hud = new Laya.Sprite();
hud.graphics.drawRect(0, 0, 240, 80, "#4dabf7");
hud.pos(Laya.stage.width - 260, 20);
Laya.stage.addChild(hud);

Laya.stage.on(Laya.Event.RESIZE, null, () => {
  hud.pos(Laya.stage.width - 260, 20);
});
```

这段代码背后的含义是：

1. 项目按 `1334x750` 作为设计坐标理解画面
2. 运行时要求按横屏语义处理屏幕
3. 内容完整显示，允许留边
4. 画布在留边时居中
5. HUD 位置按当前舞台宽度重新计算

## 最容易混淆的五件事

1. 设计分辨率不是设备真实分辨率
2. `scaleMode` 处理的是尺寸映射，`screenMode` 处理的是横竖方向
3. `showall` 和 `noborder` 都是等比缩放，但一个保内容完整，一个优先铺满屏幕
4. `alignH`、`alignV` 主要影响画布停靠位置，不是重新定义对象局部坐标
5. `Laya.stage.width`、`height` 在不同缩放模式下，语义并不完全一样

## 结论

理解 `Stage` 的关键，不是背更多适配参数，而是先建立一条稳定关系：

1. 用 `designWidth`、`designHeight` 定义设计坐标基准
2. 用 `scaleMode` 决定舞台尺寸和屏幕之间怎么换算
3. 用 `screenMode` 决定横竖屏语义
4. 用 `alignH`、`alignV` 处理留白区域里的画布停靠方式
5. 最终用 `Laya.stage.width`、`height`、`mouseX`、`mouseY` 理解运行时实际坐标

把这五层分开之后，再看布局、输入和多端适配问题，会清楚很多。
