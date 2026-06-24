# 模型和动画资源接入项目后该怎么用

很多人第一次把 3D 模型放进项目后，会把下面几件事混成一件事：

1. 模型文件是不是已经等于一个可直接操作的角色对象。
2. `Laya.loader.load()` 返回的是模型实例，还是资源定义。
3. 动画片段、`Animator`、状态机、脚本分别负责什么。
4. 模型资源、场景资源、预制体实例为什么看起来都和“节点树”有关。

把这几层分开后，模型和动画在运行时的使用方式其实很稳定。

## 先看最小主链

最常见的运行时写法不是手工一层层拼网格、材质、骨骼，而是先加载一个层级资源入口，再创建实例：

```ts
await Laya.init({ designWidth: 1334, designHeight: 750 });

const scene = new Laya.Scene3D();
Laya.stage.addChild(scene);

const camera = new Laya.Camera(0, 0.1, 100);
scene.addChild(camera);
camera.transform.position = new Laya.Vector3(0, 1.5, 4);
camera.transform.lookAt(new Laya.Vector3(0, 1, 0), new Laya.Vector3(0, 1, 0));

const light = new Laya.DirectionLight();
scene.addChild(light);

const rolePrefab = await Laya.loader.load("resources/3d/role/Role.lh") as Laya.Prefab;
const role = rolePrefab.create() as Laya.Sprite3D;
scene.addChild(role);

const animator = role.getComponent(Laya.Animator);
animator?.play("Idle");
```

这段代码里真正成立的是这条链：

1. `load()` 先把模型对应的层级资源和依赖资源准备好。
2. `create()` 再把这份资源定义变成当前这一份运行时节点树。
3. 节点树被加到 `Scene3D` 后，才真正进入 3D 场景。
4. `Animator` 再驱动这份实例里的骨骼或节点动画。

所以“模型接入项目后怎么用”，最常见答案不是“直接拿模型文件操作”，而是“加载层级资源，创建实例，再在实例上取组件和写逻辑”。

## 先分清模型资源和模型实例

模型资源本身不是已经活着的角色对象。

更准确地说，它更接近一份资源定义，里面通常会带着：

1. 节点层级。
2. `MeshSprite3D` 或 `SkinnedMeshSprite3D` 这类可渲染节点。
3. 材质、纹理、网格等依赖资源引用。
4. 骨骼、挂点、动画片段。
5. 可能还包括 `Animator` 这样的运行时组件配置。

而模型实例是 `create()` 之后拿到的当前对象树。

例如：

```ts
const rolePrefab = await Laya.loader.load("resources/3d/role/Role.lh") as Laya.Prefab;

const a = rolePrefab.create() as Laya.Sprite3D;
const b = rolePrefab.create() as Laya.Sprite3D;
```

这里共享的是同一份资源定义，不是同一个运行时对象。

所以：

1. 改 `a` 的位置，不会自动改 `b`。
2. 销毁 `a`，不等于把模型资源本身删掉。
3. 同一份模型资源可以在多个场景里创建多个实例。

## 模型资源在运行时常见是什么形态

很多项目里，美术源文件可能来自 `glTF`、`GLB`、`FBX` 等格式。

但业务代码真正面对的，通常不是“原始建模格式本身”，而是导入后可被运行时识别的层级资源和依赖资源。

从运行时角度看，常见形态通常是：

1. 一个模型入口资源。
2. 若干网格、材质、纹理等依赖资源。
3. 如果模型带动画，再附带动画片段、骨骼和 `Animator` 相关配置。

这也是为什么业务代码最常写的是：

```ts
const model = await Laya.loader.load("resources/3d/props/Crate.lh") as Laya.Prefab;
const crate = model.create() as Laya.Sprite3D;
scene.addChild(crate);
```

而不是手工去分别加载网格文件、材质文件、纹理文件，再自己把整棵层级重新拼出来。

后者不是不能做，而是更适合：

1. 程序化生成对象。
2. 只复用某个局部资源。
3. 需要完全自定义装配流程。

## 模型资源和场景资源不是一层东西

模型资源通常表达的是“场景里有一个什么对象”。

场景资源表达的则是“整个场景长什么样，以及里面有哪些对象”。

可以先用一句话区分：

1. `Scene3D` 更像运行时宿主。
2. 场景资源更像一整棵场景定义。
3. 模型资源更像可接入场景的一段对象树。

所以单独加载一个角色模型，并不会自动得到：

1. 相机。
2. 灯光。
3. 完整关卡环境。

它更常见的用法是被加入某个已经存在的 `Scene3D`：

```ts
const enemyPrefab = await Laya.loader.load("resources/3d/enemy/Enemy.lh") as Laya.Prefab;
const enemy = enemyPrefab.create() as Laya.Sprite3D;
scene.addChild(enemy);
```

这里 `enemy` 是场景中的一个对象，不是整个场景。

## 模型资源和预制体实例也要分开看

很多导入后的 3D 模型，在运行时使用方式上就很像 3D 预制体：

1. 先加载资源定义。
2. 再创建实例。
3. 最后把实例接入当前场景。

最重要的边界仍然是：

1. 资源文件保存的是“怎么构造这棵树”。
2. 实例对象才是“当前这一次真正活着的树”。

所以运行时逻辑通常应该写在实例这一层，例如：

```ts
const npc = rolePrefab.create() as Laya.Sprite3D;
npc.transform.position = new Laya.Vector3(2, 0, 0);
scene.addChild(npc);
```

这里改的是这一个 `npc`，不是资源定义本身。

## 动画资源先解决的是“怎么动”

如果模型带动画，资源里保存的通常不是“角色现在正在播放哪一段”，而是“有哪些可用于播放的动画数据”。

这些动画数据更接近：

1. 骨骼或节点随时间变化的关键帧。
2. 可被 `Animator` 读取和评估的片段。
3. 可被状态机组织成待机、跑步、攻击这类状态。

也就是说：

1. 动画片段负责提供运动数据。
2. `Animator` 负责在实例上计算和应用这些数据。
3. 状态机负责管理状态切换。
4. 脚本负责决定什么时候该切状态、该响应什么业务事件。

## `Animator`、状态机、脚本的边界

这三层最容易混淆。

可以先把职责拆成这样：

### `Animator`

它首先是一个运行时组件。

它负责：

1. 持有动画播放能力。
2. 在每帧评估当前动画结果。
3. 把结果写回骨骼或节点变换。

如果没有 `Animator` 这一层，动画数据本身不会自动让模型动起来。

### 状态机

状态机负责的是动画状态之间的组织关系，例如：

1. 待机切跑步。
2. 跑步切攻击。
3. 攻击播完回待机。

它更关心的是：

1. 当前处于哪个状态。
2. 什么时候允许切换。
3. 切换时如何过渡和混合。

所以状态机管理的是“播哪段、怎么切”，不是角色完整业务逻辑本身。

### 脚本

脚本更适合处理的是游戏语义：

1. 角色是否收到移动输入。
2. 是否进入受击、死亡、交互状态。
3. 技能命中、AI 决策、网络同步等业务条件。

脚本通常不该去替代状态机做整套动画过渡管理。

更稳妥的协作方式是：

1. 脚本决定业务状态。
2. 状态机把业务状态映射成动画状态。
3. `Animator` 负责真正播放和评估。

## 一个常见协作拆法

角色逻辑经常会拆成下面三层：

1. 模型实例负责承载节点、骨骼、挂点和渲染结果。
2. `Animator` 和状态机负责动画播放表现。
3. 脚本负责输入、AI、战斗和数值逻辑。

最短例子：

```ts
const role = rolePrefab.create() as Laya.Sprite3D;
scene.addChild(role);

const animator = role.getComponent(Laya.Animator);

if (isMoving) {
  animator?.play("Run");
} else {
  animator?.play("Idle");
}
```

这个例子虽然简单，但边界是清楚的：

1. `isMoving` 是业务条件。
2. `play("Run")` 和 `play("Idle")` 是动画控制。
3. 角色是否真的能显示、能移动、能受击，仍然是脚本和场景系统共同决定的。

## 什么时候该改资源，什么时候该改实例

这是模型接入项目后最常见的实际判断。

适合在资源侧稳定下来的内容通常有：

1. 默认层级结构。
2. 骨骼和挂点命名。
3. 默认材质和网格引用。
4. 默认动画集合。

适合在实例侧按本次运行再决定的内容通常有：

1. 出生位置。
2. 当前朝向。
3. 当前播放哪段动画。
4. 当前武器、血量、状态、目标。

如果把这条边界弄清楚，模型资源、动画控制和脚本逻辑就不会互相挤在一起。

## 先记住这条主线

模型和动画接入项目后，最常见的运行时主线可以压缩成一句话：

先加载模型对应的层级资源，再创建实例，把实例加入 `Scene3D`，然后在实例上通过 `Animator`、状态机和脚本分别接管表现与业务。

只要一直分清“资源定义”和“运行时实例”不是一回事，后面的角色生成、动画切换、挂点逻辑和资源复用都会清楚很多。
