# 第一版文章 Backlog

## 说明

这份 backlog 用于把 `site-outline.md` 落成可执行的文章清单。

规则：

1. 标题优先使用问题句或任务句
2. 每篇只服务一个主问题
3. `阶段` 用于控制先后顺序，不等于最终导航顺序
4. `路径` 是建议的站内路径，不代表必须一次性全部创建文件

## P0：最短主线

### 开始使用

1. `docs/getting-started/about-this-docs-project.md`
标题：这套文档和官方文档是什么关系
阶段：P0

2. `docs/getting-started/standard-project.md`
标题：什么是标准 Laya 项目
阶段：P0

3. `docs/getting-started/project-startup.md`
标题：一个 Laya 项目是如何启动的
阶段：P0

### 项目工作流

4. `docs/project-workflow/layaair-cli.md`
标题：如何用 `layaair` CLI 管理项目
阶段：P0

5. `docs/project-workflow/asset-layout.md`
标题：项目里的资源应该放在哪里
阶段：P0

### 运行时基础

6. `docs/runtime-foundation/runtime-entry-points.md`
标题：引擎启动后，运行时里有哪些关键入口
阶段：P0

7. `docs/runtime-foundation/resource-loading.md`
标题：资源加载之后会经历什么
阶段：P0

## P1：日常开发主链路

### 项目工作流

8. `docs/project-workflow/debug-and-scripts.md`
标题：开发时如何调试和执行脚本
阶段：P1

9. `docs/project-workflow/build-output.md`
标题：项目构建后会得到什么
阶段：P1

### 运行时基础

10. `docs/runtime-foundation/stage-and-screen.md`
标题：舞台是如何管理屏幕和坐标的
阶段：P1

11. `docs/runtime-foundation/node-and-script.md`
标题：节点和脚本是如何一起工作的
阶段：P1

12. `docs/runtime-foundation/events-and-updates.md`
标题：事件、输入和帧更新是怎么协作的
阶段：P1

### 场景与预制体

13. `docs/scene-and-assets/scene-runtime.md`
标题：场景文件在运行时是怎么生效的
阶段：P1

14. `docs/scene-and-assets/prefab.md`
标题：预制体适合解决哪些问题
阶段：P1

15. `docs/scene-and-assets/script-binding.md`
标题：场景和预制体里的脚本是怎么绑定的
阶段：P1

16. `docs/scene-and-assets/scene-assets.md`
标题：场景资源该怎么预加载和回收
阶段：P1

### 2D 与 UI

17. `docs/2d-ui/common-2d-objects.md`
标题：常见 2D 对象应该怎么组织
阶段：P1

18. `docs/2d-ui/textures-text-and-atlases.md`
标题：文本、图片和图集应该怎么管理
阶段：P1

19. `docs/2d-ui/ui-structure.md`
标题：一个复杂 UI 应该怎么拆
阶段：P1

20. `docs/2d-ui/interaction-effects.md`
标题：交互效果应该交给谁来控制
阶段：P1

### 3D

21. `docs/3d/scene-setup.md`
标题：一个 3D 场景是怎么搭起来的
阶段：P1

22. `docs/3d/camera-light-and-visibility.md`
标题：为什么这个 3D 场景能被正确看见
阶段：P1

23. `docs/3d/rendering-objects.md`
标题：一个 3D 对象是怎么被渲染出来的
阶段：P1

24. `docs/3d/models-and-animations.md`
标题：模型和动画资源接入项目后该怎么用
阶段：P1

25. `docs/3d/interaction-and-picking.md`
标题：3D 交互为什么会命中这个对象
阶段：P1

## P2：扩展专题

26. `docs/2d-ui/ui2.md`
标题：什么时候需要关心 `ui2`
阶段：P2

27. `docs/engineering/asset-versioning-and-remote-assets.md`
标题：资源版本和远程资源该怎么规划
阶段：P2

28. `docs/engineering/cross-platform-release.md`
标题：多平台发布时最容易踩哪些坑
阶段：P2

29. `docs/engineering/plugins-and-build-extensions.md`
标题：什么时候需要看插件和构建扩展资料
阶段：P2

30. `docs/engineering/advanced-3d.md`
标题：3D 进阶能力应该从哪里开始
阶段：P2

31. `docs/engineering/faq.md`
标题：常见问题与排错索引
阶段：P2

## 建议执行顺序

建议采用 3 条并行主线推进：

1. 项目工作流主线：2, 3, 4, 5, 8, 9
2. 运行时与资源主线：6, 7, 10, 11, 12, 13, 16
3. 2D/3D 使用主线：17, 18, 19, 20, 21, 22, 23, 24, 25

这样可以同时积累：

1. 入门文档
2. 原理文档
3. 使用场景文档
