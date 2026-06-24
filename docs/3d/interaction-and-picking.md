# 为什么这个 3D 交互会命中这个对象

很多人第一次看 3D 交互时，会把“看见了这个对象”和“点中了这个对象”当成同一件事。

但运行时里，它们其实是两条不同链路：

1. 渲染链决定它有没有被画出来
2. 交互链决定输入最终命中了谁

所以一个对象出现在画面里，不代表它一定能被点中；反过来，一个对象之所以被点中，也不是因为“图片上看起来点到了它”，而是因为一整条检测链成立了。

## 先看最小命中链

最短例子：

```ts
await Laya.init({ designWidth: 1334, designHeight: 750 });

const scene = new Laya.Scene3D();
Laya.stage.addChild(scene);

const camera = new Laya.Camera(0, 0.1, 100);
scene.addChild(camera);
camera.transform.position = new Laya.Vector3(0, 2, 6);
camera.transform.lookAt(new Laya.Vector3(0, 0.5, 0), new Laya.Vector3(0, 1, 0));

const box = new Laya.MeshSprite3D(Laya.PrimitiveMesh.createBox());
box.transform.position = new Laya.Vector3(0, 0.5, 0);
scene.addChild(box);

const collider = box.addComponent(Laya.PhysicsCollider);
collider.colliderShape = new Laya.BoxColliderShape(1, 1, 1);

const ray = new Laya.Ray(new Laya.Vector3(), new Laya.Vector3());
const point = new Laya.Vector2();
const hit = new Laya.HitResult();

Laya.stage.on(Laya.Event.MOUSE_DOWN, null, () => {
  point.setValue(Laya.stage.mouseX, Laya.stage.mouseY);
  camera.viewportPointToRay(point, ray);

  if (scene.physicsSimulation.rayCast(ray, hit)) {
    console.log("命中对象：", hit.collider.owner.name || hit.collider.owner);
  }
});
```

这段代码里，真正发生的事情是：

1. 鼠标位置先落在 2D 屏幕坐标上
2. 相机把这个屏幕点换算成一条 3D 射线
3. 物理系统用这条射线去测试场景里的碰撞体
4. 第一个被打中的碰撞体，成为这次命中结果

所以“为什么命中了它”，本质上是在问：

1. 这条射线是怎么来的
2. 它打到了哪个碰撞体
3. 业务脚本又怎么使用了这个结果

## 命中的不是渲染网格，而是碰撞体

3D 里最容易混淆的一点是：

1. `MeshSprite3D` 负责显示
2. `PhysicsCollider`、`Rigidbody3D` 这类组件负责物理检测

也就是说，射线拾取默认不是在问“画面上哪个三角形看起来被点到了”，而是在问“这条射线先碰到了哪个物理碰撞体”。

例如：

```ts
const box = new Laya.MeshSprite3D(Laya.PrimitiveMesh.createBox());
scene.addChild(box);

const collider = box.addComponent(Laya.PhysicsCollider);
collider.colliderShape = new Laya.SphereColliderShape(1);
```

这里看起来是一个盒子，但用于交互检测的却是球形碰撞范围。

所以当你发现：

1. 明明点在模型边缘也能命中
2. 视觉上点到了模型，结果却没命中

先看的通常不是材质和贴图，而是碰撞体形状是不是和你想要的交互范围一致。

## 没有碰撞体，很多 3D 点击逻辑就没有命中基础

如果对象没有进入物理检测链，射线就没有东西可打。

最常见的几种情况是：

1. 节点有模型，但没加碰撞组件
2. 加了碰撞组件，但没设置 `colliderShape`
3. 业务脚本写了 `onMouseDown()`，但宿主 3D 节点本身没有可命中的碰撞体

可以先建立一个很稳的判断：

1. “看得见”主要看渲染链
2. “点得到”主要看拾取链
3. 拾取链里最关键的实体通常是碰撞体

## `PhysicsCollider` 和 `Rigidbody3D` 都能成为被命中的对象

从射线检测角度看，重点不是它是不是动态刚体，而是它是不是一个可参与物理查询的碰撞体。

常见理解可以先记成：

1. `PhysicsCollider` 常用来做静态场景物体、地面、机关触发区
2. `Rigidbody3D` 常用来做会受物理模拟影响的动态物体
3. 只要有有效碰撞形状，它们都可能成为射线命中结果

例如地面通常会这样配置：

```ts
const plane = new Laya.MeshSprite3D(Laya.PrimitiveMesh.createPlane(10, 10));
scene.addChild(plane);

const collider = plane.addComponent(Laya.PhysicsCollider);
collider.colliderShape = new Laya.BoxColliderShape(10, 0.01, 10);
```

而一个会掉落的箱子更常见的是：

```ts
const box = new Laya.MeshSprite3D(Laya.PrimitiveMesh.createBox());
scene.addChild(box);

const rigidbody = box.addComponent(Laya.Rigidbody3D);
rigidbody.colliderShape = new Laya.BoxColliderShape(1, 1, 1);
```

## 触发器解决的是“进入范围”，不是“发生实体阻挡”

`isTrigger` 很容易被误解成“不能被检测到”。

它更准确的含义是：这个碰撞体主要用于触发事件，而不是产生普通的实体碰撞响应。

例如：

```ts
const area = new Laya.Sprite3D();
scene.addChild(area);

const collider = area.addComponent(Laya.PhysicsCollider);
collider.colliderShape = new Laya.BoxColliderShape(3, 2, 3);
collider.isTrigger = true;
```

这类对象通常用来表达：

1. 进入门口范围时弹提示
2. 走进补给区时恢复状态
3. 进入机关区域时启动后续逻辑

所以要分清两类结果：

1. 射线命中结果：这次点击或指向打到了谁
2. 触发器事件结果：某个物体进入、停留、离开了哪个区域

它们都和碰撞体有关，但不是同一层逻辑。

## `onTriggerEnter()` 处理的是范围进入，不是屏幕点击

例如：

```ts
const { regClass } = Laya;

@regClass()
class AreaHint extends Laya.Script {
  onTriggerEnter(other: any): void {
    console.log("进入触发区：", other.owner.name);
  }
}

area.addComponent(AreaHint);
```

这段逻辑的含义不是“鼠标点中了 `area`”，而是“别的碰撞体进入了 `area` 的触发范围”。

所以如果你的需求是点击选中，就要优先看射线拾取；如果你的需求是角色走进范围自动触发，就要优先看触发器事件。

## 屏幕点为什么会变成一条 3D 射线

鼠标和触摸最开始给你的，是屏幕上的一个 2D 点。

例如：

```ts
const point = new Laya.Vector2(Laya.stage.mouseX, Laya.stage.mouseY);
```

但 3D 世界里，单独一个屏幕点并没有深度信息。它只告诉你“用户在屏幕上点了这里”，并没有直接告诉你“用户想选中 3D 世界里的哪个位置”。

所以相机要做一次换算：

```ts
const ray = new Laya.Ray(new Laya.Vector3(), new Laya.Vector3());
camera.viewportPointToRay(point, ray);
```

换算后的含义可以先记成：

1. 射线起点来自当前相机
2. 射线方向穿过这个屏幕点对应的观察方向
3. 后续射线会沿这个方向去测试 3D 世界

这就是“屏幕点击”和“3D 命中对象”之间的桥。

## 为什么同一个屏幕点，换一台相机会命中不同对象

因为射线不是从屏幕自己长出来的，而是从当前相机的观察关系里推出来的。

只要下面任一条件变化，命中结果都可能变：

1. 相机位置变了
2. 相机朝向变了
3. 相机视口变了
4. 前方遮挡关系变了

例如两个相机都看向场景，但位置不同。屏幕中心点对第一台相机可能对应地面，对第二台相机可能对应角色头顶。

所以“我点的是同一块屏幕区域，为什么命中对象变了”，第一反应通常应该是看相机，而不是先怀疑输入坐标错了。

## 射线返回的通常是“第一个命中的碰撞体”

最常见的拾取写法是：

```ts
const hit = new Laya.HitResult();

if (scene.physicsSimulation.rayCast(ray, hit)) {
  console.log(hit.collider.owner.name);
}
```

这里可以先理解成：

1. 物理系统沿射线向前查找
2. 返回最近的那个命中结果
3. 结果里会带上碰撞体、命中点、法线等信息

所以当前面有遮挡时，被选中的通常不是你“想点的后面那个对象”，而是这条射线先遇到的前景碰撞体。

这也是为什么地面、透明挡板、隐藏交互区之类的碰撞配置，都会直接影响拾取结果。

## 需要整条穿透结果时，用 `rayCastAll()`

如果你不是只想知道“最前面是谁”，而是想拿到整条射线穿过的对象列表，可以用：

```ts
const hits: Laya.HitResult[] = [];
scene.physicsSimulation.rayCastAll(ray, hits);
```

这类结果常用于：

1. 需要自己决定忽略哪些遮挡物
2. 需要同时处理地面和角色等多层命中
3. 想做穿透型检测或调试显示

但大多数点击选中场景里，`rayCast()` 已经够用了，因为业务更关心当前最前面的有效目标。

## `HitResult` 里真正有价值的是“命中了谁”和“命中在哪里”

最常用的几个信息通常是：

1. `hit.collider`：命中的碰撞体
2. `hit.collider.owner`：碰撞体所属节点
3. `hit.point`：命中世界坐标
4. `hit.normal`：命中面的法线方向

例如：

```ts
if (scene.physicsSimulation.rayCast(ray, hit)) {
  const target = hit.collider.owner as Laya.Sprite3D;
  target.transform.position = hit.point;
}
```

虽然这个例子很短，但它已经表达出一个关键边界：

1. 射线检测负责产生命中结果
2. 脚本逻辑负责决定拿这个结果做什么

## 脚本不是“自动懂你要点击谁”，而是在消费命中结果

运行时里常见有两种配合方式。

第一种是你自己统一监听输入，再做射线检测：

```ts
Laya.stage.on(Laya.Event.MOUSE_DOWN, null, () => {
  point.setValue(Laya.stage.mouseX, Laya.stage.mouseY);
  camera.viewportPointToRay(point, ray);

  if (scene.physicsSimulation.rayCast(ray, hit)) {
    const target = hit.collider.owner as Laya.Sprite3D;
    target.active = false;
  }
});
```

这类写法的好处是：

1. 交互入口集中
2. 更容易统一过滤目标
3. 更适合做选中、高亮、地面寻路这类全局逻辑

第二种是把脚本事件方法挂在对象自己身上：

```ts
const { regClass } = Laya;

@regClass()
class Clickable extends Laya.Script {
  onMouseDown(): void {
    console.log("点击了", this.owner.name);
  }
}

box.addComponent(Clickable);
```

这类写法本质上仍然依赖命中结果，只是运行时帮你把命中的输入事件路由到了对应 `owner`。

所以它不是绕开了拾取，而是把“命中后怎么分发给脚本”这一步自动做了。

## `onMouseDown()`、`onTriggerEnter()`、`onCollisionEnter()` 不是一回事

这三个名字很像，但来源完全不同：

1. `onMouseDown()`：来自输入命中
2. `onTriggerEnter()`：来自触发范围进入
3. `onCollisionEnter()`：来自实体碰撞接触

如果把它们混成一种“交互事件”，就会很容易写出边界不清的逻辑。

最简单的区分方式可以先记成：

1. 点一下物体，看 `onMouseDown()`
2. 走进区域，看 `onTriggerEnter()`
3. 两个实体真正撞上，看 `onCollisionEnter()`

## 为什么脚本逻辑会改变“最终看起来命中了谁”

很多时候，物理命中结果只是第一步，真正的业务目标还要再过一层脚本判断。

例如：

1. 命中了角色的武器子节点，但业务想选中整个人物根节点
2. 命中了地面，但业务只允许选中敌人层
3. 命中了遮挡板，但业务要忽略它继续找后面的可交互对象

最短处理方式通常像这样：

```ts
if (scene.physicsSimulation.rayCast(ray, hit)) {
  let target = hit.collider.owner as Laya.Sprite3D;

  while (target.parent && target.parent instanceof Laya.Sprite3D && target.name !== "UnitRoot") {
    target = target.parent as Laya.Sprite3D;
  }

  console.log("最终交互目标：", target.name);
}
```

这里的重点不是这段规则本身，而是：

1. 射线先给你原始命中对象
2. 脚本再把它映射成真正的业务对象

所以“为什么最后选中了这个对象”，有时答案不是物理层，而是你后面又做了一次归类、过滤或提升。

## 交互排查时，最稳的是按链路往后看

如果一个 3D 对象命中不对，通常按下面顺序最稳：

1. 先确认输入点是不是正确拿到了 `Laya.stage.mouseX`、`Laya.stage.mouseY`
2. 再确认是不是用正确相机调用了 `camera.viewportPointToRay()`
3. 再确认目标对象是否真的有碰撞组件和碰撞形状
4. 再确认前面有没有别的碰撞体先把射线挡住
5. 再确认你后续脚本有没有把原始命中结果重定向到别的对象
6. 如果是范围触发需求，再检查 `isTrigger` 和触发脚本，而不是只盯点击事件

把这条链拆开之后，“为什么会命中这个对象”通常就不再是玄学问题，而是可以逐层定位的运行时结果。
