---
title: "交互效果"
order: 4
---

# 交互效果应该交给谁来控制

UI 交互效果最容易乱的地方，不是不会写动画，而是把几类完全不同的职责混在一起：

1. 结构负责把节点挂进显示树。
2. 状态负责记录当前到底是什么样。
3. 反馈效果负责把变化表现出来。
4. `Laya.Tween` 负责一段时间里的平滑过渡。
5. `Laya.timer` 负责延时和节拍。
6. 脚本生命周期负责这些逻辑什么时候接入、什么时候退出。

如果这六层边界不清楚，常见结果就是：

1. 为了做按压效果，直接把整块结构改乱。
2. 业务状态偷偷塞进 Tween 回调里。
3. 节点已经停用，外部计时器还在继续改它。
4. 一个按钮的短反馈，最后变成一串很难收尾的临时逻辑。

## 结构先回答“谁跟着谁一起动”

结构层先解决的是显示树关系，不是这次要不要播效果。

更实用的分法通常是：

1. 外层容器负责点击区域和整块层级。
2. 中间容器负责真正要动的部分。
3. `Image`、`Label` 负责图片和文字槽位。

最短例子：

```ts
const button = new Laya.Box();
button.size(220, 80);

const content = new Laya.Box();
content.size(220, 80);

const bg = new Laya.Image("resources/ui/button.png");
const label = new Laya.Label("购买");
label.pos(78, 22);

button.addChild(content);
content.addChild(bg);
content.addChild(label);
```

这里更稳的边界是：

1. `button` 负责整块按钮结构和命中区域。
2. `content` 负责按压缩放、弹起这类局部反馈。
3. `bg` 和 `label` 只是被显示出来，不负责决定效果逻辑。

如果整块按钮都需要一起缩放，改 `content` 通常比直接改所有子节点更稳。

## 状态先回答“它现在是什么”

交互里真正应该被保存的，通常不是“它刚刚播了哪个 Tween”，而是当前状态。

常见适合保留为状态的内容：

1. 是否选中。
2. 是否禁用。
3. 是否展开。
4. 当前倒计时还剩多少秒。
5. 当前是否正在加载。

常见不适合拿来当业务真相的内容：

1. 当前 `alpha` 是不是 0。
2. 当前 `scaleX` 有没有变小。
3. 某个 Tween 是否刚刚播完。

例如“按钮不能点了”，更接近状态：

```ts
let disabled = true;

label.text = disabled ? "冷却中" : "购买";
content.alpha = disabled ? 0.6 : 1;
```

这里真正的业务结论是 `disabled`，不是 `alpha = 0.6`。

## `Laya.Tween` 负责“怎么平滑变过去”

`Laya.Tween` 最适合做的是属性插值：

1. 缩放。
2. 位移。
3. 透明度。
4. 旋转。

例如按钮按下和弹起：

```ts
Laya.Tween.to(content, { scaleX: 0.96, scaleY: 0.96 }, 80);
Laya.Tween.to(content, { scaleX: 1, scaleY: 1 }, 80);
```

它更像在回答：

1. 哪个对象要变。
2. 变到什么值。
3. 用多久变过去。

它不适合负责这些事情：

1. 倒计时每秒减一。
2. 页面级业务分支判断。
3. 对象启用和停用时的绑定清理。

一个实用判断是：

如果重点是“中间过程要顺一点”，优先想 `Laya.Tween`；如果重点是“什么时候再做一次”，通常不是 Tween 的职责。

## `Laya.timer` 负责“多久做一次”

`Laya.timer` 管的是时序和节拍，不是插值过程。

最常见的三类用途：

1. `once()` 做一次延时。
2. `loop()` 做按时间重复。
3. `frameLoop()` 做按帧重复。

最短例子：

```ts
Laya.timer.once(200, null, () => {
  panel.visible = false;
});

Laya.timer.loop(1000, null, () => {
  remainSeconds -= 1;
  timeText.text = String(remainSeconds);
});
```

这里更准确的理解是：

1. `once()` 负责 200ms 后再收尾一次。
2. `loop()` 负责每秒推进一次倒计时状态。
3. 文本怎么从旧值过渡到新值，不是 `timer` 的重点。

所以：

1. 平滑位移、渐隐渐现，优先用 `Laya.Tween`。
2. 延时关闭、轮询刷新、倒计时跳秒，优先用 `Laya.timer`。

## 脚本生命周期负责“什么时候接上，什么时候停掉”

很多交互效果并不是写出来就结束了，还要考虑它跟着谁生效、跟着谁退出。

对挂在节点上的 UI 脚本，更稳的边界通常是：

1. `onAwake()` 做一次性查找和初始化。
2. `onEnable()` 绑定事件，启动这次启用期需要的逻辑。
3. `onDisable()` 解绑事件，清掉这次启用期的 Tween 和 timer。

最短例子：

```ts
class PressEffect extends Laya.Script {
  onEnable(): void {
    this.owner.on(Laya.Event.MOUSE_DOWN, this, this.onDown);
    this.owner.on(Laya.Event.MOUSE_UP, this, this.onUp);
  }

  onDisable(): void {
    this.owner.off(Laya.Event.MOUSE_DOWN, this, this.onDown);
    this.owner.off(Laya.Event.MOUSE_UP, this, this.onUp);
    Laya.Tween.clearAll(this.owner);
    Laya.timer.clearAll(this);
  }

  onDown(): void {
    Laya.Tween.to(this.owner, { scaleX: 0.96, scaleY: 0.96 }, 80);
  }

  onUp(): void {
    Laya.Tween.to(this.owner, { scaleX: 1, scaleY: 1 }, 80);
  }
}
```

这里最重要的不是方法名本身，而是归属关系：

1. 监听跟着脚本启停。
2. 缓动跟着效果目标清理。
3. 计时器跟着当前脚本实例清理。

如果一个效果天然属于某个节点，就不要把它长期挂成无归属的全局逻辑。

## 交互反馈效果只负责“把状态变化表现出来”

交互反馈本身更像表现层结果，不应该反过来当业务控制器。

几个常见例子：

1. 按压反馈：缩一下、暗一下、弹一下。
2. 选中反馈：高亮、描边、位移一点点。
3. 出现和消失：淡入、上浮、缩放进入。
4. 错误提示：抖动、闪红、短暂文本提示。

这些效果更适合建立在状态变化之上。

例如页签切换时，更稳的顺序通常是：

1. 先改当前选中页签状态。
2. 再刷新旧页签和新页签的显示。
3. 最后给新页签补一个很短的强调反馈。

而不是：

1. 先播一串动画。
2. 等动画播完再猜现在是不是选中了。

同样，错误提示也更适合这样理解：

1. “输入不合法”是状态判断结果。
2. 抖动和变红只是把这个结果表现出来。

## 一个更稳的分工链

处理一个常见按钮交互时，可以先按这条链分工：

1. 结构决定按钮根节点、内容节点、图标和文字槽位。
2. 状态决定它当前是可点、禁用还是选中。
3. 事件负责启动这一次交互流程。
4. `Laya.Tween` 负责 100ms 到 300ms 的短过渡。
5. `Laya.timer` 负责延时收尾或倒计时跳秒。
6. 脚本生命周期负责绑定、暂停、清理。

如果还想再压缩成一句话，可以先这样记：

1. 结构管位置和层级。
2. 状态管当前真相。
3. Tween 管怎么顺滑地变。
4. timer 管什么时候再做。
5. 生命周期管什么时候开始和结束。
6. 反馈效果管让玩家看见这次变化。

## 结论

交互效果最稳的做法，通常不是把所有逻辑都塞进一个点击回调里，而是先把职责拆开：

1. 用结构留住稳定层级。
2. 用状态留住当前结果。
3. 用 `Laya.Tween` 处理短时过渡。
4. 用 `Laya.timer` 处理延时和节拍。
5. 用脚本生命周期管理接入和退出。
6. 让 UI 反馈效果只负责表现，不代替业务真相。

把这几层分开后，按钮按压、页签切换、弹窗出现、倒计时刷新这类交互，通常都会清楚很多。
