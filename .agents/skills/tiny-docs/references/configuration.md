---
title: 配置说明
order: 2
---

# 配置说明

`tiny-docs` 的主要配置位于 `configs.yaml` 的 `DocsModule` 节点下。

如果你需要先理解这些字段背后的概念模型，请先阅读 [文档技能](../SKILL.md)。

## 最小配置

```yaml
DocsModule:
  workspace: ../..
  title: 我的项目文档
  publish:
    routePrefix: /
```

## 常用字段

### workspace

- 文档扫描根目录
- 相对当前配置文件所在目录解析
- `spaces` 中的目录通常都位于这个工作区内

### assetsDirs

- 静态资源目录列表
- 用于 logo、图片、样式、脚本等页面依赖资源
- 相对路径相对配置文件目录解析

```yaml
DocsModule:
  assetsDirs:
    - ../node_modules/@tinyaxis/tiny-docs/assets
    - ./assets
```

### 站点外观

| 字段              | 作用     | 说明                     |
| ----------------- | -------- | ------------------------ |
| `title`           | 站点标题 | 显示在页面标题和站点头部 |
| `tagline`         | 副标题   | 可选的站点说明文案       |
| `logo`            | 站点图标 | 通常指向静态资源路径     |
| `copyright`       | 版权信息 | 页脚显示文案             |
| `showLineNumbers` | 显示行号 | 控制代码块是否显示行号   |

```yaml
DocsModule:
  title: Tiny Docs 示例
  tagline: 项目文档站点
  logo: /assets/dev/docs/img/icon.png
  showLineNumbers: false
```

### 路由与发布

| 字段                        | 作用                 | 说明                               |
| --------------------------- | -------------------- | ---------------------------------- |
| `routePrefix`               | 开发时文档站路径前缀 | 控制页面访问路径                   |
| `staticRoutePrefix`         | 开发时静态资源前缀   | 控制脚本、样式等静态资源路径       |
| `publish.routePrefix`       | 发布后的站点路径前缀 | 静态站部署子路径时使用             |
| `publish.staticRoutePrefix` | 发布后的静态资源前缀 | 发布产物中的资源路径前缀           |
| `publish.llm.*`             | LLM 发布附加产物     | 控制是否生成 `llms.txt` 与 `llms/` |

```yaml
DocsModule:
  routePrefix: /documentation
  publish:
    routePrefix: /
    llm:
      enabled: true
      includeFrontmatter: true
```

### publish.llm

`publish.llm` 用来控制面向 Agent / LLM 的 Markdown 镜像发布。

启用后，发布目录默认会额外生成：

- `llms.txt`：Markdown 索引入口
- `llms/`：按文档路由镜像出来的 Markdown 文件目录

```yaml
DocsModule:
  publish:
    llm:
      enabled: true
      includeFrontmatter: true
      mermaid:
        output: source
```

| 字段                 | 作用                                 | 说明          |
| -------------------- | ------------------------------------ | ------------- |
| `enabled`            | 是否启用 LLM 发布                    | 默认 `false`  |
| `includeFrontmatter` | 是否保留 front matter                | 默认 `true`   |
| `mermaid.output`     | Mermaid 在 LLM Markdown 中的输出方式 | 默认 `source` |

`mermaid.output` 支持：

- `source`：保留原始 Mermaid 代码块，不进行额外转换
- `wireframe`：只输出线框图，适合需要纯文本视觉预览时使用
- `both`：同时输出原始 Mermaid 代码块和线框图

如果 Mermaid 类型不支持或线框图生成失败，会保留原始 Mermaid 代码块，避免信息丢失。

`llm` 路径约定固定为：

- `llms.txt`
- `llms/`

### navigator.buttons

顶部导航只看 `navigator.buttons`，不会从 `pages` 自动推导。

```yaml
DocsModule:
  navigator:
    buttons:
      - title: 首页
        url: /
        match: equals
      - title: 文档
        url: /getting-started
        match: startsWith
        prefix: /
      - title: About
        url: /about
        match: equals
```

### spaces

`spaces` 用来声明文档按什么分组展示，以及扫描哪些目录。

```yaml
DocsModule:
  spaces:
    - title: 开发文档
      dirs:
        - path: docs
          title: 快速开始
          linkPrefix: ""
    - title: Agent
      agentSkillNoOrphanDirs: true
      dirs:
        - path: .agents/skills
          title: skills
          agentSkillNoOrphanDirs: space
      useOrphanDocsAsDirs: true
      noOrphanDirs: true
      order: 100
```

| 字段                     | 作用                 | 说明                                            |
| ------------------------ | -------------------- | ----------------------------------------------- |
| `title`                  | space 名称           | 展示在侧边栏分组标题上                          |
| `dirs[]`                 | 扫描入口             | 每个条目描述一个目录入口                        |
| `linkPrefix`             | 链接前缀             | 覆盖该入口默认生成的路由前缀                    |
| `useOrphanDocsAsDirs`    | 孤立文档提升         | 只有单一文档时是否提升为目录入口                |
| `noOrphanDirs`           | 孤立目录提升         | 目录只包含一个子目录时是否提升子目录            |
| `agentSkillNoOrphanDirs` | Agent Skill 目录提升 | 是否折叠 Agent Skill 根节点下唯一的无链接子目录 |
| `order`                  | 排序值               | 数值越小越靠前                                  |

`dirs[]` 条目还支持这些字段：

| 字段                     | 作用                 | 说明                                                       |
| ------------------------ | -------------------- | ---------------------------------------------------------- |
| `path`                   | 目录路径             | 相对 `workspace` 或绝对路径                                |
| `title`                  | 入口标题             | 作为目录入口的展示标题                                     |
| `context`                | 关联上下文           | 供运行时挂载自定义页面等场景使用                           |
| `linkPrefix`             | 链接前缀             | 覆盖该入口默认生成的路由前缀                               |
| `agentSkillNoOrphanDirs` | Agent Skill 目录提升 | `on` / `off` / `space`；默认 `space`，表示服从所属 `space` |

### Agent Skill 根目录折叠

`agentSkillNoOrphanDirs` 只影响 `/.agents/skills/<skill>` 这一层目录。

典型场景是 skill 根目录下只有一个无链接子目录，例如 `references/`。开启后，这一层会被自动拍平：

- sidebar 中不再额外显示 `references`
- 面包屑路径会直接从 `Agent Skills / <skill>` 进入最终文档

配置优先级如下：

1. `dirs[].agentSkillNoOrphanDirs: 'on'` 强制开启
2. `dirs[].agentSkillNoOrphanDirs: 'off'` 强制关闭
3. `dirs[].agentSkillNoOrphanDirs: 'space'` 或不写时，服从 `space.agentSkillNoOrphanDirs`

示例：

```yaml
DocsModule:
  spaces:
    - title: 框架模块
      agentSkillNoOrphanDirs: true
      dirs:
        - path: packages/foundation/modules/docs
          title: docs
          agentSkillNoOrphanDirs: space
        - path: packages/foundation/modules/other-module
          title: other-module
          agentSkillNoOrphanDirs: off
```

### remaps

如果某些文档不在 `spaces` 的目录中，但希望被纳入文档站，可以使用 `remaps`。

```yaml
DocsModule:
  remaps:
    getting-started:
      title: 快速开始
      target: README
      space: 开发文档
      order: -1
```

### search

`search` 控制文档站的内置搜索能力。

```yaml
DocsModule:
  search:
    enabled: true
    placeholder: 搜索文档
    shortcut: Ctrl K
    initialText: 输入关键词开始搜索
    searchingText: 正在搜索...
    emptyText: 没有找到相关结果
    errorText: 搜索暂时不可用，请稍后重试
```

| 字段            | 作用           | 说明                                                    |
| --------------- | -------------- | ------------------------------------------------------- |
| `enabled`       | 是否启用搜索   | 默认 `true`；关闭后不会渲染搜索入口，也不会生成搜索索引 |
| `placeholder`   | 输入框提示文案 | 显示在顶部按钮和搜索弹窗输入框中                        |
| `shortcut`      | 快捷键提示     | 显示在顶部按钮右侧                                      |
| `initialText`   | 初始状态文案   | 搜索弹窗首次打开时的说明文字                            |
| `searchingText` | 搜索中提示文案 | 搜索请求进行中时显示                                    |
| `emptyText`     | 无结果提示文案 | 没有匹配结果时显示                                      |
| `errorText`     | 失败提示文案   | 搜索索引不可用或请求失败时显示                          |

搜索在开发态和发布态的行为不同：

- 开发态会使用本地搜索索引预热，索引准备好后才允许真正发起搜索
- 发布时会生成 Pagefind 索引文件，随静态文档一起部署

### pages

自定义页面统一放在 `DocsModule.pages`。页面机制、导航关系和挂载方式请看 [自定义页面](custom-pages/).
