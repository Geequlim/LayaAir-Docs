# 事件、输入和帧更新是怎么协作的

很多运行时问题看起来像三件事：

1. 点击为什么没有触发
2. 逻辑为什么每帧都在跑
3. 缓动、计时器、脚本更新为什么会互相影响

但从运行时角度看，它们其实是一条连续链：

1. 输入先进入 `Laya.stage`
2. 事件再分发到具体对象或沿父链传播
3. 脚本、计时器、缓动在后续帧里继续推进状态
4. 生命周期变化又会决定哪些监听和更新还有效

如果这条链没分清，就很容易把下面几件事混成一件事：

1. 事件是发给 `owner` 还是发给 `Laya.stage`
2. 点击一次和每帧更新一次到底有什么本质区别
3. `Laya.timer`、`Laya.Tween`、`onUpdate()` 分别该负责什么
4. 节点停用、移除后，哪些逻辑会停，哪些不会停

## 先看最小主链

先看一段最短代码：

```ts
const { regClass } = Laya;

@regClass()
class InputDemo extends Laya.Script {
  onMouseDown(): void {
    this.owner.scale(0.9, 0.9);
  }

  onMouseUp(): void {
    this.owner.scale(1, 1);
  }

  onUpdate(): void {
    this.owner.rotation += 1;
  }
}

await Laya.init({ designWidth: 1334, designHeight: 750 });

const box = new Laya.Sprite();
box.size(200, 200);
box.graphics.drawRect(0, 0, 200, 200, "#4dabf7");
box.pos(300, 200);
Laya.stage.addChild(box);
box.addComponent(InputDemo);
```

这段代码里至少同时发生了三件事：

1. `onMouseDown()` 和 `onMouseUp()` 处理离散输入事件
2. `onUpdate()` 处理持续帧更新
3. 三者都依附在同一个 `owner` 节点上运行

所以运行时不是“事件系统”和“更新系统”各跑各的，而是共同作用在同一批节点和组件上。

## 事件先解决的是“谁通知谁”

最小理解方式可以先记成：

1. 监听方先用 `on()` 注册回调
2. 某个对象后续调用 `event()` 发出事件
3. 运行时把回调执行起来

例如：

```ts
const panel = new Laya.Sprite();

panel.on("ready", null, () => {
  console.log("panel ready");
});

panel.event("ready");
```

这里最关键的边界是：

1. 事件名只是一个约定
2. 监听挂在哪个对象上，决定了谁能收到通知
3. 事件本身不等于输入，它也可以是业务对象主动发出的消息

所以不要把“事件”直接等同于“鼠标点击”。点击只是事件来源之一。

## 输入不是直接掉到节点上，而是先进入舞台

对鼠标、触摸、键盘这类输入来说，`Laya.stage` 是总入口。

先看一个最短示例：

```ts
Laya.stage.on(Laya.Event.CLICK, null, () => {
  console.log(Laya.stage.mouseX, Laya.stage.mouseY);
});
```

这说明至少有两件事先成立了：

1. 输入先被舞台接收
2. 输入位置已经被换算成当前舞台坐标

后续如果这是一次指向显示对象的输入，运行时通常还会继续做：

1. 根据当前舞台坐标做命中判断
2. 找到被命中的显示对象
3. 把事件分发给这个对象
4. 再沿父节点链继续传播

所以更准确的理解是：

输入先进入 `Stage`，再进入显示树。

## `owner` 事件和 `Laya.stage` 事件不是一个边界

这条边界在脚本里尤其重要。

可以先把常见事件分成两类。

第一类：更像“某个对象被点了、按了、移入了”

1. `Laya.Event.CLICK`
2. `Laya.Event.MOUSE_DOWN`
3. `Laya.Event.MOUSE_UP`
4. `Laya.Event.MOUSE_MOVE`

这类事件更适合挂在具体对象上：

```ts
button.on(Laya.Event.CLICK, null, () => {
  console.log("click button");
});
```

第二类：更像“整个运行时状态变了”

1. 键盘输入
2. 舞台尺寸变化
3. 焦点变化

这类事件更适合挂在 `Laya.stage` 上：

```ts
Laya.stage.on(Laya.Event.KEY_DOWN, null, (e: Laya.Event) => {
  console.log(e.keyCode);
});

Laya.stage.on(Laya.Event.RESIZE, null, () => {
  console.log(Laya.stage.width, Laya.stage.height);
});
```

最实用的判断方式是：

1. 只关心某个节点本身，优先监听 `owner`
2. 关心整个舞台或全局输入，优先监听 `Laya.stage`

## 为什么脚本事件方法看起来像“自动接好了线”

如果你写的是 `Laya.Script`，很多时候不需要手动 `on()`，因为脚本约定方法会在启用时自动接入对应入口。

例如：

```ts
const { regClass } = Laya;

@regClass()
class PlayerInput extends Laya.Script {
  onMouseDown(): void {}
  onMouseUp(): void {}
  onKeyDown(): void {}
}
```

这里更接近的运行时语义是：

1. `onMouseDown()`、`onMouseUp()` 这类鼠标事件接到 `owner`
2. `onKeyDown()` 这类键盘事件接到 `Laya.stage`
3. 脚本禁用或离开激活链后，这些入口也会一起停掉

所以脚本事件方法的重点不是“少写几行绑定代码”，而是它已经帮你把对象边界和舞台边界区分好了。

## 事件分发解决的是瞬时通知，不是持续推进

这也是事件最容易被滥用的地方。

例如：

```ts
button.on(Laya.Event.CLICK, null, () => {
  console.log("buy");
});
```

这里更像：

1. 某件事发生了一次
2. 回调被执行一次
3. 执行完就结束

但下面这类逻辑不是一次事件能解决的：

1. 角色持续移动
2. 倒计时每秒变化
3. 物体在 0.3 秒内平滑位移
4. HUD 需要每帧跟随目标位置

这些都属于“状态持续推进”，需要后续更新机制参与。

## 每帧更新先解决的是“状态怎么持续变化”

最常见的逐帧入口有两类：

1. `Laya.timer.frameLoop()`
2. 脚本的 `onUpdate()` / `onLateUpdate()`

先看脚本更新：

```ts
@regClass()
class Rotator extends Laya.Script {
  onUpdate(): void {
    this.owner.rotation += 1;
  }
}
```

这说明：

1. 逻辑是跟着脚本和节点生命周期跑的
2. 只要脚本启用、节点处于有效激活链，它就会持续更新

再看 `frameLoop`：

```ts
Laya.timer.frameLoop(1, null, () => {
  console.log("tick", Laya.timer.currFrame);
});
```

这说明：

1. 它也是逐帧执行
2. 但它不天然依附某个脚本方法名
3. 更像显式注册一段帧循环任务

所以两者都能“每帧跑一次”，但挂接位置不一样。

## `onUpdate()` 和 `frameLoop()` 怎么选

最简单的经验是：

1. 某段逻辑天然属于某个节点行为，优先放 `onUpdate()`
2. 某段逻辑更像独立调度任务，优先放 `Laya.timer.frameLoop()`

例如角色朝向、血条跟随、状态机推进，更适合放脚本更新里：

```ts
@regClass()
class FollowTarget extends Laya.Script {
  target!: Laya.Sprite;

  onLateUpdate(): void {
    this.owner.pos(this.target.x, this.target.y - 80);
  }
}
```

而全局倒计时、轮询刷新、统一节拍，更适合显式计时器：

```ts
Laya.timer.loop(1000, null, () => {
  console.log("1 second");
});
```

关键不是“哪个更高级”，而是：

1. 一个偏组件生命周期
2. 一个偏全局调度

## `onLateUpdate()` 的重点不是更晚一点，而是跟随前面的结果

`onUpdate()` 和 `onLateUpdate()` 最容易被写成一样。

更实用的区分方式是：

1. `onUpdate()` 先推进对象自己的主逻辑
2. `onLateUpdate()` 再根据别的对象本帧结果做跟随或收尾

例如：

```ts
@regClass()
class CameraLike extends Laya.Script {
  target!: Laya.Sprite;

  onLateUpdate(): void {
    this.owner.x = this.target.x;
  }
}
```

这类“等目标先动完，我再跟上”的逻辑，通常更适合放到 `onLateUpdate()`。

## `Laya.timer` 解决的是节奏调度，不只是延时

很多人第一次只记住 `once()` 或 `loop()`，但 `Timer` 的本质更接近“统一调度时序任务”。

最常见的几种写法分别对应不同节奏：

```ts
Laya.timer.once(300, null, () => {
  console.log("delay");
});

Laya.timer.loop(500, null, () => {
  console.log("half second");
});

Laya.timer.frameLoop(1, null, () => {
  console.log("every frame");
});
```

可以先这样理解：

1. `once()` 适合一次性延时
2. `loop()` 适合按时间间隔重复执行
3. `frameLoop()` 适合按帧重复执行

所以 `Timer` 管的是“多久执行一次”，不是“值怎么平滑变化”。

## `Laya.Tween` 解决的是插值变化，不是业务调度

缓动最适合处理“从当前值平滑过渡到目标值”。

例如：

```ts
Laya.Tween.to(panel, { x: 800, alpha: 0.5 }, 300);
```

这段代码更像在表达：

1. 目标属性是什么
2. 用多长时间过去
3. 中间过程由缓动系统持续推进

所以它和 `Laya.timer` 的边界很明确：

1. `Laya.timer` 负责什么时候触发
2. `Laya.Tween` 负责一段时间里怎么变过去

如果把大量业务判断塞进 Tween 回调里，后面通常会越来越难维护。

## 一个常见协作链：事件触发，Tween 执行动画，Timer 或更新收尾

真实代码里，这几套机制经常是串起来用的。

例如：

```ts
button.on(Laya.Event.CLICK, null, () => {
  Laya.Tween.to(panel, { y: 100 }, 200);
  Laya.timer.once(200, null, () => {
    panel.visible = false;
  });
});
```

这段代码的主链是：

1. 点击事件负责启动流程
2. Tween 负责 200ms 内的平滑移动
3. Timer 负责在动画结束后做一次状态切换

这比把全部逻辑都塞进 `onUpdate()` 更清晰。

## 生命周期会反过来决定哪些事件和更新还有效

事件、更新、缓动不是永远存在的，它们都受节点和组件状态影响。

例如一个脚本宿主节点被停用后，最直接的影响通常是：

1. 脚本事件方法不再继续响应
2. `onUpdate()`、`onLateUpdate()` 不再执行
3. 依附这个节点行为的逻辑要么暂停，要么需要你显式清理

例如：

```ts
node.active = false;
```

这不是“画面先隐藏一下”这么简单，而是让这个节点子树离开激活链。

同样，只有组件被停掉时：

```ts
script.enabled = false;
```

更接近的是：

1. 节点还在
2. 别的组件可能仍在运行
3. 只有当前脚本退出事件和更新驱动

所以定位问题时，要先分清是节点退出了，还是组件退出了。

## 为什么定时器和缓动也要讲“归属”

虽然 `Timer` 和 `Tween` 都可以全局启动，但业务上仍然要关心它们属于谁。

例如：

```ts
Laya.timer.loop(1000, this, this.tick);
Laya.Tween.to(this.owner, { alpha: 0 }, 300);
```

这里至少要先想清楚：

1. 这段循环是跟某个脚本走，还是跟整个游戏走
2. 这个缓动目标对象后面会不会被移除或销毁
3. 当对象不再参与运行时，这些任务还应不应该继续

最稳妥的习惯不是“反正能跑就行”，而是让调度边界和对象边界尽量一致。

## 一段更接近真实使用的最小示例

```ts
const { regClass } = Laya;

@regClass()
class CardItem extends Laya.Script {
  onEnable(): void {
    this.owner.on(Laya.Event.CLICK, this, this.onClick);
  }

  onDisable(): void {
    this.owner.off(Laya.Event.CLICK, this, this.onClick);
  }

  onClick(): void {
    Laya.Tween.to(this.owner, { scaleX: 1.1, scaleY: 1.1 }, 100);
    Laya.timer.once(100, this, () => {
      Laya.Tween.to(this.owner, { scaleX: 1, scaleY: 1 }, 100);
    });
  }

  onUpdate(): void {
    this.owner.rotation += 0.2;
  }
}

await Laya.init({ designWidth: 1334, designHeight: 750 });

const card = new Laya.Sprite();
card.size(180, 240);
card.graphics.drawRect(0, 0, 180, 240, "#ffd43b");
card.pos(400, 180);
Laya.stage.addChild(card);
card.addComponent(CardItem);
```

这段代码里每一块的角色分别是：

1. `owner.on(Laya.Event.CLICK, ...)`：监听对象自身点击
2. `Laya.Tween.to(...)`：让点击反馈平滑出现
3. `Laya.timer.once(...)`：控制第二段动作的触发时机
4. `onUpdate()`：持续推进旋转效果
5. `onEnable()` / `onDisable()`：让监听跟着组件启停

它们不是彼此替代关系，而是各自负责不同层的运行时协作。

## 什么时候该优先用哪一种机制

可以先用最短规则判断：

1. 某件事发生一次，优先用事件
2. 某段逻辑要持续推进，优先用 `onUpdate()` 或 `frameLoop()`
3. 某个状态要按时间间隔触发，优先用 `Laya.timer`
4. 某组属性要平滑过渡，优先用 `Laya.Tween`
5. 某段逻辑天然依附节点，优先放脚本生命周期里管理

如果把这几种机制混着乱用，最常见的结果就是：

1. 一次事件里偷偷启动多个长期循环
2. 短动画被写成复杂的每帧判断
3. 对象已经停用，外部计时器还在继续改它的状态

## 最容易混淆的五件事

1. 输入先进入 `Laya.stage`，不是直接落到节点上
2. 鼠标触摸事件更偏 `owner`，键盘和 `resize` 更偏 `Laya.stage`
3. 事件负责通知一次，更新负责持续推进
4. `Laya.timer` 管时序调度，`Laya.Tween` 管插值过渡
5. 节点和组件的激活状态，会反过来影响事件和更新是否继续生效

## 结论

对 `LayaAir 3.4` 来说，事件、输入和帧更新协作的关键，不是多记几个 API 名字，而是先建立一条稳定关系：

1. 用 `Laya.stage` 接住全局输入和舞台级事件
2. 用 `owner` 接住对象级输入和交互反馈
3. 用 `onUpdate()` / `onLateUpdate()` 处理依附节点的持续逻辑
4. 用 `Laya.timer` 处理节奏和延时
5. 用 `Laya.Tween` 处理属性随时间的平滑变化
6. 用生命周期控制这些机制什么时候接入、暂停和退出

把这六层边界先建立起来，后面再看交互、动画、UI 跟随和状态同步问题时，会清楚很多。
