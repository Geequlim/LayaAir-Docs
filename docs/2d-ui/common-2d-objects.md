# 常见 2D 对象应该怎么组织

一个 2D 画面看起来像一张完整界面，运行时里其实是一棵显示对象树。

真正需要先分清的，不是“该用哪个控件”，而是“这个对象在这一层到底负责什么”：

1. 谁负责当容器
2. 谁负责显示图片
3. 谁负责显示文字
4. 谁负责占位和排版
5. 谁负责跟着父节点一起移动、缩放、隐藏

如果这些职责没有拆开，后面就很容易出现这些问题：

1. 一改父节点位置，整组内容都跟着乱跑
2. 明明只想换图，却把交互范围和排版结构一起改掉
3. 文本、底图、图标都直接堆在同一层，后面很难复用

## 先建立最小心智模型

先看一个最短例子：

```ts
const panel = new Laya.Box();
panel.pos(40, 40);
panel.size(320, 180);

const bg = new Laya.Image("resources/ui/panel.png");
bg.size(320, 180);

const title = new Laya.Label("背包");
title.pos(24, 20);

panel.addChild(bg);
panel.addChild(title);
Laya.stage.addChild(panel);
```

这里最重要的不是 API 数量，而是角色分工：

1. `panel` 负责当这一组内容的父容器
2. `bg` 负责显示背景图
3. `title` 负责显示标题文字
4. `Laya.stage` 是整棵 2D 显示树的根入口

所以组织 2D 对象时，优先想的是“按角色分层”，不是“先把所有东西都创建出来再摆位置”。

## `Sprite` 更像通用显示节点

`Sprite` 最适合拿来做通用显示对象，或者做一层轻量容器。

它常见的职责有三类：

1. 画基础图形
2. 挂贴图或子节点
3. 作为一组内容的公共父节点

例如：

```ts
const marker = new Laya.Sprite();
marker.graphics.drawCircle(0, 0, 8, "#ff6b6b");
marker.pos(200, 120);
Laya.stage.addChild(marker);
```

这个例子里，`Sprite` 更像一个“我需要一个能显示、能移动、能进层级树的对象”。

如果一个对象的重点是：

1. 自己要画点东西
2. 以后可能继续挂子节点
3. 不需要明显的 UI 布局语义

那通常先想到 `Sprite`。

## `Box` 更像结构容器

`Box` 更适合表达“这里是一块 UI 结构区域”。

它和 `Sprite` 都能挂子节点，但在组织界面时，`Box` 的语义通常更明确：

1. 这一层主要是为了装内容
2. 这一层通常会关心宽高和区域边界
3. 子节点的位置更像是在这块区域里排布

例如一个头像块：

```ts
const card = new Laya.Box();
card.size(220, 80);

const icon = new Laya.Image("resources/ui/avatar.png");
icon.pos(0, 0);

const nameText = new Laya.Label("Knight");
nameText.pos(96, 12);

card.addChild(icon);
card.addChild(nameText);
```

这里如果后面整块卡片要移动、隐藏、缩放，改 `card` 就够了。

所以一个实用判断是：

1. 偏通用显示节点，优先想 `Sprite`
2. 偏界面分组容器，优先想 `Box`

这不是硬性限制，而是更稳定的组织习惯。

## `Image` 负责把图片放进显示树

`Image` 最核心的职责很单纯：显示图片。

例如：

```ts
const icon = new Laya.Image("resources/ui/item-sword.png");
icon.pos(16, 16);
parent.addChild(icon);
```

组织上要注意的一点是：

不要因为一张图当前看起来就是一个按钮背景，就把“图片”和“按钮整块结构”理解成同一个对象。

更稳妥的拆法通常是：

1. 外层容器负责整块区域
2. `Image` 负责底图、图标、装饰图
3. 文本单独用 `Label`

这样后面换图、换文案、加角标时，不会互相缠在一起。

## `Label` 负责文字，不负责整块 UI

`Label` 的职责也应该尽量保持单一：显示文字。

例如：

```ts
const price = new Laya.Label("128");
price.color = "#ffffff";
price.fontSize = 28;
price.pos(120, 18);
parent.addChild(price);
```

如果一个区域里既有标题、数值、描述，通常也不要把它们硬塞进同一个文本对象里处理全部布局。

更常见的组织方式是：

1. 标题一个 `Label`
2. 数值一个 `Label`
3. 描述一个 `Label`
4. 它们共同挂在同一个父容器下

这样文本变化时，局部调整更可控。

## 显示层级先解决的是“谁跟着谁”

显示树里的父子关系，首先决定的是变换继承关系。

先看一个最小例子：

```ts
const hud = new Laya.Box();
hud.pos(60, 40);

const icon = new Laya.Image("resources/ui/star.png");
icon.pos(0, 0);

const count = new Laya.Label("x12");
count.pos(48, 10);

hud.addChild(icon);
hud.addChild(count);
Laya.stage.addChild(hud);
```

这里的因果关系是：

1. `icon` 和 `count` 的位置，都是相对 `hud` 来看
2. `hud` 挪到别处，两个子节点会一起跟着走
3. `hud` 隐藏、缩放、旋转，子节点也会一起受影响

所以父子层级首先不是“目录结构”，而是“变换关系和组织关系”。

这也是为什么很多界面不该把所有对象都直接 `addChild` 到 `Laya.stage`。

## 常见的 2D 层级拆法

一个界面通常可以先拆成几层：

1. 页面根节点
2. 背景层
3. 内容层
4. 前景装饰层
5. 弹出层或提示层

例如：

```ts
const page = new Laya.Box();
const backgroundLayer = new Laya.Sprite();
const contentLayer = new Laya.Box();
const popupLayer = new Laya.Box();

page.addChild(backgroundLayer);
page.addChild(contentLayer);
page.addChild(popupLayer);
Laya.stage.addChild(page);
```

这种拆法的价值在于：

1. 同层对象更容易统一管理
2. 需要整体隐藏某一层时，改父节点就够了
3. 后面加动画、加遮罩、加弹窗时，不容易把原有结构搅乱

## 变换基础先记三件事

组织 2D 对象时，最常用的变换通常就这几类：

1. `x`、`y` 或 `pos()` 决定位置
2. `scaleX`、`scaleY` 决定缩放
3. `rotation` 决定旋转

例如：

```ts
const icon = new Laya.Image("resources/ui/coin.png");
icon.pos(200, 80);
icon.scale(1.2, 1.2);
icon.rotation = 15;
Laya.stage.addChild(icon);
```

这里要先建立一个稳定认识：

1. 子节点的位置默认是相对父节点
2. 父节点的缩放和旋转会继续影响子节点
3. 调整一个容器，通常比逐个调整内部对象更省事

所以当一组对象要整体移动或整体缩放时，优先改它们共同的父节点。

## 锚点会直接影响“围着哪里变换”

缩放和旋转不是总围着左上角理解，关键还要看锚点。

例如：

```ts
const badge = new Laya.Image("resources/ui/badge.png");
badge.size(64, 64);
badge.anchorX = 0.5;
badge.anchorY = 0.5;
badge.pos(300, 160);
badge.rotation = 30;
Laya.stage.addChild(badge);
```

这里的重点是：

1. 锚点决定对象以自身哪个参考点参与旋转和缩放
2. 以中心点做动效时，通常会把锚点设到 `0.5, 0.5`
3. 如果不先统一锚点，很多 UI 动效看起来就会像在“甩出去”

组织层级时也要顺手想清楚：

是整块容器需要绕中心变化，还是只有里面某个图标需要绕中心变化。

## 一个对象只做一类主职责，结构会稳很多

2D 场景和 UI 最常见的失控方式，就是让一个节点同时承担太多角色。

例如下面这种拆法通常更稳：

1. 页面根节点负责整页开关
2. 面板容器负责区域位置和尺寸
3. 背景 `Image` 负责底图
4. 图标 `Image` 负责视觉元素
5. 文字 `Label` 负责文案
6. 需要整体运动的部分，再单独包一层父节点

这样做的直接好处是：

1. 换资源时不容易碰坏结构
2. 做动效时边界清楚
3. 复用某一块 UI 时更容易整体搬走

## 先按“对象角色”组织，再按“视觉效果”微调

如果只记一个经验，可以先记这条：

1. `Sprite` 常拿来做通用显示节点或轻量容器
2. `Box` 常拿来做界面分组和结构容器
3. `Image` 负责图片显示
4. `Label` 负责文字显示
5. 父子层级决定谁跟着谁一起变换
6. 位置、缩放、旋转、锚点先在容器层想清楚，再做局部细节

把这套角色边界先分开后，一个 2D 场景或 UI 界面即使继续变复杂，也更容易保持结构清楚。
