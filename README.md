# LayaAir 文档

围绕 LayaAir 3.4 引擎，以代码优先、结构优先的方式重新组织的中文技术文档。

## 这份文档适合谁

- 不想依赖 IDE 界面、希望用代码和命令行完成开发的工程师
- 需要稳定检索和理解 LayaAir 知识的 Agent / LLM
- 想系统理解引擎运行时机制而非面板操作的开发者

## 文档结构

| 章节 | 内容 |
|---|---|
| [开始使用](./docs/getting-started/) | 项目结构、启动方式、核心对象关系 |
| [项目工作流](./docs/project-workflow/) | CLI 管理、资源组织、调试、构建 |
| [运行时基础](./docs/runtime-foundation/) | 引擎入口、舞台、节点脚本、事件更新、资源加载 |
| [场景与预制体](./docs/scene-and-assets/) | 场景运行时模型、预制体、脚本绑定、资源回收 |
| [2D 与 UI](./docs/2d-ui/) | 显示对象组织、资源管理、UI 拆分、交互效果 |
| [3D 开发](./docs/3d/) | 场景搭建、渲染链路、模型动画、交互拾取 |
| [工程化与扩展](./docs/engineering/) | 资源版本、多平台发布、插件扩展、进阶入口 |

## 本地构建

```bash
yarn install
yarn tiny-docs -c configs.yaml publish dist
```

构建产物输出到 `dist`。

## 在线访问

发布到 GitHub Pages：

```
https://geequlim.github.io/LayaAir-Docs/
```

## 写作规划

文档写作规划和取材策略见 [writing-plan](./writing-plan/) 目录。
