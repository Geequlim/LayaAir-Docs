---
title: "如何用 layaair CLI 管理项目"
order: 1
---

# 如何用 `layaair` CLI 管理项目

这里讨论的是当前项目采用的 `layaair` 命令行工作流：安装、创建项目、预览、构建、校验资源，以及执行注册脚本。

如果你希望只靠命令行完成日常工程操作，可以把它理解成一条固定主线：

```bash
layaair create
layaair run
layaair build
layaair validate
```

需要注意的是，当前 CLI 的语义和旧的命令行流程不是一回事。这里的命令以 `layaair` 为准，不沿用旧流程里的参数和习惯。

## 安装与版本管理

`layaair` 的安装分成两层：

1. dispatcher：负责接收命令和分发版本
2. runtime：真正执行 `create`、`run`、`build`、`validate` 的 CLI 版本

只装 dispatcher 还不能正常使用 CLI，还需要至少安装一个 runtime 版本。

```bash
curl -fsSL https://raw.githubusercontent.com/layabox/layaair-cli/master/install.sh | bash
~/.layaair/layaair install 3.4.0
```

安装后，可以先检查本机已有版本：

```bash
layaair list
```

查看当前激活版本：

```bash
layaair --version
```

如果你需要明确指定某次命令使用 3.4.0，可以直接写在命令前：

```bash
layaair --version=3.4.0 build web -p .
```

通常 dispatcher 会自动选择合适版本；当传入 `--project` 时，它也会结合项目信息推断应使用的 CLI 版本。但在写脚本或 CI 时，显式写出 `--version=3.4.0` 更稳妥。

## 创建项目

创建项目前，先列出当前 CLI 实际可用的模板名，不要凭记忆手写。

```bash
layaair create -l
```

这个列表是当前 CLI 的真实结果，里面可能同时包含内置模板、本地缓存模板和云端模板。选中模板后再创建项目：

```bash
layaair create MyGame -t "2D empty project"
```

如果你只想用默认模板，最短写法就是：

```bash
layaair create MyGame
```

有两个行为要特别注意：

1. `create` 成功后，项目就已经创建完成，不需要自动接着跑 `build` 或 `run`
2. 默认不会额外再创建一层子目录，项目文件会直接写入目标位置

如果你要做自动化创建流程，建议始终把模板查询和项目创建分成两步，而不是把模板名写死在文档或脚本里长期不动。

## 预览项目

`layaair run` 在不带 `--script` 时，会启动内置预览服务器。这是现代 CLI 的默认预览方式，不需要另外准备外部 Web 服务器。

```bash
layaair run -p .
```

也可以省略 `run`，因为直接执行 `layaair` 默认就是这个子命令：

```bash
layaair -p .
```

启动后，CLI 会输出预览地址。服务端口读取自项目设置，因此同一个项目通常会保持稳定的本地预览入口。

如果你只是想确认项目能否启动，`run` 就够了；它不是构建命令，也不会替代正式发布。

## 构建项目

构建前，先查询当前项目支持哪些平台名：

```bash
layaair build -l
```

或者在指定项目时查询：

```bash
layaair build -p . -l
```

平台名应以 CLI 实时输出为准。确认后再执行构建：

```bash
layaair build web -p .
```

这里的 `web` 是平台名位置参数。虽然也可以通过参数形式传入平台，但日常使用里直接写位置参数更直观。

如果你需要把产物输出到指定目录，再额外加输出参数；但最重要的习惯仍然是先查平台，再构建，避免把旧版本或旧教程里的平台名直接搬过来。

## 校验资源

`validate` 用于校验项目资源文件，适合在批量改资源、改场景、改预制体之后快速检查问题。

最小示例：

```bash
layaair validate assets/main.lh assets/player.lprefab -p .
```

如果你已经在项目目录里执行命令，仍然建议带上 `-p .`，这样脚本语义更明确，也更适合复制到自动化流程中。

`validate` 的定位不是构建，也不是预览。它更像是一次独立的资源一致性检查，应该在资源变更频繁时尽早运行，而不是等到构建失败后再回头排查。

## 执行注册脚本

`run --script` 适合执行项目里已经注册过的类上的静态方法。它最常见的用途不是启动游戏，而是跑一段工程脚本，例如导出数据、批处理资源、生成额外文件。

先定义一个可被 CLI 调用的注册类：

```ts
@IEditorEnv.regClass()
export class MyExporter {
  static async run(output: string): Promise<void> {
    // ...
  }
}
```

再从命令行执行这个静态方法：

```bash
layaair run -p . --script=MyExporter.run --script-args="dist/data.json"
```

这里有三个关键点：

1. `--script` 指向的是 `类名.静态方法名`
2. 目标类需要先完成注册
3. `--script-args` 是一整段参数字符串，CLI 会再把它拆成位置参数

如果脚本代码不在现有项目资源里，也可以临时追加一个 TypeScript 文件参与本次运行：

```bash
layaair run -p . --script=AX.test --script-file=/tmp/a.ts
```

`--script-file` 只影响这一次 CLI 执行，不会把这个文件正式纳入项目资源数据库。

## 推荐的命令行工作流

日常开发里，比较顺手的顺序通常是：

1. `layaair create -l`，确认模板
2. `layaair create ...`，创建项目
3. `layaair run -p .`，本地预览
4. `layaair validate ... -p .`，校验关键资源
5. `layaair build -l`，确认平台名
6. `layaair build <platform> -p .`，正式构建

如果项目里有自己的工程脚本，再把 `layaair run --script ...` 插入到这条链路中。

## 常见误区

1. 只安装 dispatcher，就以为 `layaair` 已经可用
2. 不先执行 `create -l` 或 `build -l`，直接凭记忆写模板名和平台名
3. 把旧的命令行流程当成现在的 `layaair` CLI
4. 把 `run`、`build`、`validate` 混成同一件事
5. 用 `run --script` 调实例方法，或者调用未注册的类

对 LayaAir 3.4 来说，最稳妥的做法很简单：模板先查、平台先查、版本尽量写明、每个子命令只做它自己的事。
