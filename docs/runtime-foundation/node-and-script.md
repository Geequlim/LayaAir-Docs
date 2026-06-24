---
title: "节点和脚本"
order: 3
---

# 节点和脚本是如何一起工作的

很多人第一次接触运行时时，会把“节点”“组件”“脚本”当成三套平行系统。

但在实际运行里，它们更像一条链：

1. `Node` 树负责组织对象层级
2. `Component` 负责把行为挂到节点上
3. `Script` 是最常用的行为组件形式
4. 激活、禁用、移除、销毁会一起驱动生命周期

如果这条链没分清，就很容易把下面几件事混成一件事：

1. 节点在不在树上
2. 节点是不是激活的
3. 组件是不是启用的
4. 脚本为什么有时执行、有时不执行
5. 场景或 prefab 里的脚本为什么加载后没有绑上

## 先看最小关系

先看一个最短例子：

```ts
const { regClass } = Laya;

@regClass()
class Rotator extends Laya.Script {
  onUpdate(): void {
    this.owner.rotation += 1;
  }
}

await Laya.init({ designWidth: 1334, designHeight: 750 });

const box = new Laya.Sprite();
Laya.stage.addChild(box);
box.addComponent(Rotator);
```

这段代码里有三层关系：

1. `box` 是节点
2. `Rotator` 是挂在节点上的脚本组件
3. `Laya.stage` 是整棵运行时节点树的根入口

所以真正执行 `onUpdate()` 的，不是“一个游离脚本”，而是“挂在某个节点上的组件脚本”。

## 节点树先解决的是“对象在哪里”

`Node` 树先回答的是层级关系，不是业务逻辑。

最小理解方式可以先记成：

1. 父节点决定子节点属于哪棵树
2. 节点进入有效层级后，才可能进入激活链
3. 脚本总是依附在某个节点上，而不是独立存在

例如：

```ts
const parent = new Laya.Sprite();
const child = new Laya.Sprite();

parent.addChild(child);
Laya.stage.addChild(parent);
```

这里的因果关系是：

1. `child` 先属于 `parent`
2. `parent` 再进入 `Laya.stage`
3. `child` 也随之进入这棵运行时树

所以很多脚本行为是否生效，前提都不是“脚本有没有写”，而是“脚本的宿主节点有没有真正进入有效层级”。

## `Component` 和 `Script` 不是两套无关体系

`Script` 本质上也是 `Component`。

更稳妥的理解是：

1. `Component` 是行为挂载基类
2. `Script` 是面向业务代码最常用的组件子类
3. `Script` 比普通 `Component` 多了一层事件和更新入口约定

所以这两句代码的共同点是“给节点挂组件”：

```ts
node.addComponent(SomeComponent);
node.addComponent(SomeScript);
```

差别在于，`Script` 会额外帮你接入常见运行时入口，例如：

1. `onStart`
2. `onUpdate`
3. `onLateUpdate`
4. `onMouseDown`
5. `onKeyDown`
6. 碰撞和触发器回调

这意味着，`Script` 不只是“能写生命周期的类”，而是“已经和节点、舞台事件、组件驱动器接好线的行为组件”。

## `active`、`activeInHierarchy`、`enabled` 不是同一个开关

这三个状态最容易混淆。

### `node.active`

表示节点自身想不想激活。

```ts
node.active = false;
```

这会让当前节点及其子树退出激活态。

### `node.activeInHierarchy`

表示节点在整条父链里是否真的处于激活状态。

也就是说，下面两种情况都会让它变成 `false`：

1. 节点自己 `active = false`
2. 某个父节点不激活，或节点已经从父树移除

所以“节点自己是 active”不等于“它现在真的在运行”。

### `component.enabled`

表示组件自己是否启用。

```ts
const script = node.addComponent(Rotator);
script.enabled = false;
```

这只会停掉当前组件，不会把整个节点树一起关掉。

最简单的边界可以先记成：

1. `active` 管节点自身
2. `activeInHierarchy` 管节点最终是否真正生效
3. `enabled` 管单个组件是否参与运行

## 脚本为什么会在启用时自动接上事件

`Script` 的一个关键特征，是它在启用时会把约定好的事件方法自动绑到运行时入口上。

例如你写了这些方法：

```ts
class PlayerInput extends Laya.Script {
  onMouseDown(): void {}
  onKeyDown(): void {}
}
```

运行时会在脚本启用时，把它们接到对应入口：

1. 鼠标相关事件接到 `owner`
2. 键盘相关事件接到 `Laya.stage`

而当脚本禁用时，这些绑定会一起被移除。

这也是为什么很多时候你并没有手写 `owner.on(...)` 或 `Laya.stage.on(...)`，脚本事件方法仍然能工作。

## 第一次进入激活链时，通常会发生什么

对脚本组件来说，第一次真正进入激活层级后，最常见的顺序可以先记成：

1. 节点进入 `activeInHierarchy`
2. 脚本的 `onAwake()` 执行一次
3. 脚本的 `onEnable()` 执行
4. 第一轮 `onUpdate()` 之前，脚本的 `onStart()` 执行一次
5. 后续每帧才进入 `onUpdate()` / `onLateUpdate()`

最小示例：

```ts
const { regClass } = Laya;

@regClass()
class LifeDemo extends Laya.Script {
  onAwake(): void { console.log("awake"); }
  onEnable(): void { console.log("enable"); }
  onStart(): void { console.log("start"); }
}
```

这里最值得先区分的是：

1. `onAwake()` 只在首次激活时执行一次
2. `onEnable()` 会在每次重新启用时执行
3. `onStart()` 只在第一次进入更新前执行一次

因此，初始化逻辑放错位置，就会直接造成行为偏差。

## `onAwake()` 和 `onEnable()` 最容易放错地方

一个很实用的判断方式是：

1. 只需要做一次的初始化，优先放 `onAwake()`
2. 每次重新启用都要重做的逻辑，放 `onEnable()`

例如：

```ts
@regClass()
class Enemy extends Laya.Script {
  onAwake(): void {
    this.owner.name = "enemy";
  }

  onEnable(): void {
    this.owner.rotation = 0;
  }
}
```

这里更符合语义的是：

1. 名字只需初始化一次
2. 旋转角度可能每次重新出现都要重置

如果把第二类逻辑误放到 `onAwake()`，节点再次启用时就不会重新执行。

## 移除、停用、禁用，看起来像“消失”，但不是一回事

运行时里最常见的三种“停掉它”的方式分别是：

```ts
node.active = false;
script.enabled = false;
parent.removeChild(node);
```

它们的共同点是：都可能让脚本停止工作。

但语义不同：

1. `node.active = false`：停掉整个节点及其子树
2. `script.enabled = false`：只停掉这一个组件
3. `removeChild(node)`：把节点从当前树上移走，节点本身还活着

这里最关键的一点是：

`removeChild()` 不是 `destroy()`。

被移除的节点后面仍然可以再加回别的父节点里，再次进入激活链。

## 重新启用时，哪些生命周期会重跑

如果对象没有被销毁，只是离开激活链后又回来，最常见的结果是：

1. `onAwake()` 不会再执行
2. `onStart()` 不会再执行
3. `onEnable()` 会再次执行
4. `onUpdate()` 会继续恢复

例如：

```ts
node.active = false;
node.active = true;
```

对已经跑过一次的脚本来说，更接近“重新启用”，不是“重新创建一个新对象”。

## `destroy()` 的边界是“对象不再可用”

真正不可逆的边界是销毁。

```ts
node.destroy();
```

这和 `removeChild()` 的差别很大：

1. `removeChild()` 只是离开当前父树
2. `destroy()` 会让节点进入不可再用状态
3. 节点上的组件也会一起进入销毁流程
4. 子节点默认也会一起销毁

对脚本组件来说，销毁链通常至少包含两段：

1. 如果当前处于启用态，会先经历 `onDisable()`
2. 之后再进入 `onDestroy()`

这里有一个很值得记住的细节：

组件的 `onDestroy()` 属于销毁阶段调用，不要把它简单理解成“和 `destroy()` 同一行代码里立刻完成的普通函数调用”。

在文档理解上，更稳妥的结论是：

1. `destroy()` 一旦调用，对象就应该视为结束生命周期
2. 后续清理逻辑应该放到 `onDestroy()`
3. 不要指望销毁后的节点或组件还能重新启用

## 一条更接近真实使用的生命周期链

把前面的关系串起来，一个节点和脚本常见会经过这条链：

1. 创建节点
2. 给节点添加组件或脚本，先触发 `onAdded()`
3. 节点进入激活层级后，进入 `onAwake()` / `onEnable()`
4. 首次更新前进入 `onStart()`
5. 运行中持续进入 `onUpdate()`
6. 节点被停用、移除或组件被禁用时进入 `onDisable()`
7. 节点或组件真正销毁时进入 `onDestroy()`

如果把这条链记清楚，很多“为什么脚本没执行”“为什么再次显示时逻辑没重置”“为什么对象池复用后状态怪异”的问题就会容易定位很多。

## 类注册真正影响的是“按数据恢复类”这条边界

`@regClass()` 最容易被误解成“写脚本就必须加，不加完全不能跑”。实际上，直接代码引用（如 `node.addComponent(Rotator)`）不依赖注册；它真正解决的是场景、prefab 和 `runtime` 在反序列化阶段“按名字还原成真实类”的硬边界——类没注册成功，资源能加载、节点树能创建，但脚本绑定或 `runtime` 替换会失败。

类注册和“类是否被构建保留”也要分开看：即使源码里写了 `@regClass()`，如果这段代码没有进入最终运行时产物，反序列化同样无法恢复。完整的注册原理、两条路径的对比和常见失败原因，见 [script-binding.md](../scene-and-assets/script-binding.md)。

## 最容易混淆的五件事

1. `Script` 不是脱离节点独立运行的逻辑体，它首先是 `Component`
2. `active`、`activeInHierarchy`、`enabled` 分别控制不同层级的开关
3. `onAwake()`、`onEnable()`、`onStart()` 都不是一回事
4. `removeChild()` 只是移出树，`destroy()` 才是生命周期终点
5. `@regClass()` 主要影响按数据恢复脚本和 `runtime` 类的边界

## 结论

节点和脚本一起工作的关键，不是多记几个生命周期名字，而是先建立这条主线：

1. 用 `Node` 树组织对象层级
2. 用 `Component` 把行为挂到节点上
3. 用 `Script` 接入更新、事件和碰撞等常见运行时入口
4. 用 `active` / `activeInHierarchy` / `enabled` 分清激活边界
5. 用 `destroy()` 区分“暂时离开运行”和“真正结束生命周期”
6. 用类注册处理场景、prefab 和 `runtime` 的反序列化绑定边界

把这六条边界先建立起来，后面再看事件、场景、prefab 和对象池问题时，会清楚很多。
