# OpenTUI 与 OpenCode 本地协作开发指南

## 项目关系

OpenTUI 是 OpenCode 的终端 UI 渲染层。OpenCode 通过 `@opentui/core` 和 `@opentui/solid` 包来构建其终端界面。

```
OpenCode (终端 AI 编程助手)
  └── @opentui/solid (SolidJS 绑定)
        └── @opentui/core (核心渲染引擎: TypeScript + Zig native)
```

当你需要同时修改 OpenTUI 和 OpenCode 的代码时（比如给 OpenCode 添加新的 UI 组件，或修复 OpenTUI 中的渲染 bug），需要将本地的 OpenTUI 链接到 OpenCode 项目中。

## 前置条件

- [Bun](https://bun.sh) >= 1.3.0
- [Zig](https://ziglang.org/learn/getting-started/) (仅在修改原生代码时需要)
- 本地克隆的 OpenTUI 仓库
- 本地克隆的 OpenCode 仓库（你的 fork）

## 目录结构示例

```
~/projects/
  ├── opentui/       # OpenTUI 仓库
  └── opencode/      # 你 fork 的 OpenCode 仓库
```

## 步骤一：准备 OpenTUI

```bash
cd ~/projects/opentui
bun install
```

如果你修改了 Zig 原生代码，还需要构建：

```bash
bun run build
```

> 仅修改 TypeScript 代码时不需要构建，Bun 会直接运行 `.ts` 源文件。

## 步骤二：准备 OpenCode

```bash
cd ~/projects/opencode
bun install
```

确保 `node_modules` 目录存在且包含 `.bun` 缓存目录。

## 步骤三：链接本地 OpenTUI

回到 OpenTUI 目录，运行链接脚本：

```bash
cd ~/projects/opentui
./scripts/link-opentui-dev.sh ~/projects/opencode --solid --subdeps
```

### 链接脚本参数

| 参数 | 说明 |
|------|------|
| (无额外参数) | 仅链接 `@opentui/core` + `yoga-layout` + `web-tree-sitter` |
| `--solid` | 额外链接 `@opentui/solid` + `solid-js` |
| `--react` | 额外链接 `@opentui/react` + `react` + `react-dom` + `react-reconciler` |

> OpenCode 使用 SolidJS 渲染器，所以通常需要 `--solid` 参数。

### 链接原理

脚本通过在 Bun 的缓存目录（`node_modules/.bun`）中创建 symlink 来实现链接。它会：

1. 在 OpenCode 的 `node_modules/.bun` 中查找匹配 `@opentui+core@*` 模式的缓存目录
2. 将缓存中的 `@opentui/core` 替换为指向本地 OpenTUI 源码的 symlink
3. 同样处理 peer dependencies（`yoga-layout`、`solid-js` 等），确保版本一致

这意味着 OpenCode 中所有引用 `@opentui/*` 的代码都会解析到你本地的 OpenTUI 源码。

## 开发工作流

### 场景一：仅修改 TypeScript

1. 在 OpenTUI 中修改 TypeScript 代码
2. 直接在 OpenCode 中运行/测试，改动立即生效（symlink + Bun 直接执行 `.ts`）

### 场景二：修改 Zig 原生代码

1. 在 OpenTUI 中修改 `packages/core/src/zig/` 下的 Zig 代码
2. 重新构建原生模块：
   ```bash
   cd ~/projects/opentui/packages/core
   bun run build:native      # 生产构建
   # 或
   bun run build:native:dev  # 开发构建（含调试信息，更快）
   ```
3. 在 OpenCode 中运行/测试

### 场景三：运行 OpenTUI 测试

```bash
cd ~/projects/opentui/packages/core

# TypeScript 测试
bun test

# Zig 原生测试
bun run test:native

# 过滤特定测试
bun run test:native -Dtest-filter="test name"
```

## 常见问题

### 链接后 OpenCode 报模块找不到

确认 OpenCode 已经运行过 `bun install`，且 `node_modules/.bun` 目录存在。链接脚本依赖 Bun 的缓存结构。

### 重新 `bun install` 后链接失效

每次在 OpenCode 中运行 `bun install` 都会重建 `node_modules/.bun` 缓存，需要重新执行链接脚本。

### 版本不匹配警告

OpenCode 的 `package.json` 中指定了 `@opentui/core` 的版本。如果你本地的 OpenTUI 版本号不同，可能会出现警告。这在开发阶段通常可以忽略。

### 调试 TUI 输出

OpenTUI 会捕获 `console.log` 输出。调试时可以：
- 使用反引号键（`` ` ``）切换内置控制台
- 通过环境变量控制调试行为（参考 `packages/core/docs/development.md`）

## 取消链接

删除 OpenCode 的 `node_modules` 并重新安装即可恢复到 npm 发布版本：

```bash
cd ~/projects/opencode
rm -rf node_modules
bun install
```
