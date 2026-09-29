# Changelog

本项目遵循 [语义化版本](https://semver.org/lang/zh-CN/)。

## [0.1.1] — 2026-09-29

### 修复

这一版修的是**插件根本加载不起来**的三个问题。0.1.0 虽已发布，但源码在
任何 Node 上都会在 import 阶段抛 `SyntaxError`——也就是说，此前任何按文档
执行 `dsh plugin add` 的人，装上的是一个不工作的插件。

- **`randomBytes` 从 `node:fs` 导入**：它属于 `node:crypto`。
  `import { randomBytes } from "node:fs"` 是语法错误，模块加载即失败：
  `The requested module 'node:fs' does not provide an export named 'randomBytes'`。
  注意 `node --check` **不会**报这个错——它只做语法解析，不解析 ESM 的
  具名导出，所以纯语法检查无法发现。
- **`inject` 里服务名大小写写错**：写的是 `webserver`，而宿主服务名为
  `webServer`。即使前一个问题修好，插件也不会被注入，路由不会注册。
- **`ctx.effect` 用法错误**：`ctx.effect` 要求回调**返回**一个 disposer，
  原写法 `ctx.effect(() => unlinkSync(path))` 会在启动时就把密钥文件删掉、
  且什么都没注册——密钥文件因此活过进程生命周期。改为
  `ctx.effect(() => () => unlinkSync(path))`。
- **`SECRET_PATH` 改为惰性求值**：原先在模块加载时求值，冻结了当时的
  `DSH_HOME`；若之后在另一个 home 下重新激活，密钥会写到客户端找不到的
  地方。
- **`lib/types/index.d.ts` 的 `BRIDGE_PATH` 类型与实际值不符**：声明为
  `"/api/local-bridge/auth"`，实际是 `"/local-bridge/auth"`。

### 新增

- **测试套件**（`test/index.test.js`，28 项，`node:test`）—— 此前的 144 行
  插件代码没有任何测试。覆盖：三道防护各自真的会挡（禁用任一守卫，测试即
  失败）、密钥每次启动重新生成、密钥文件 0600 且在预置为 0644 时收紧、
  停止时删除、拒绝时不铸造任何 URL、铸造失败返回 500 而非崩溃。
  测试全部使用临时 `DSH_HOME`，不触碰真实安装。
- `package.json` 加 `"test"` 脚本。

### 验证

对每个修复都做了变异测试（把修复改回原样，确认测试变红）：

| 变异 | 结果 |
| --- | --- |
| 改回 `node:fs` 导入 | 28 项全红 |
| 改回 `webserver` | 1 项红 |
| 改回 `() => unlinkSync(...)` | 22 项红 |
| 停用 loopback 守卫 | 6 项红 |
| 削弱密钥比较 | 3 项红 |

## [0.1.0] — 2026-09-27

初始版本：在 DSH 进程内注册 `/local-bridge/auth`，让本机 CLI 通过
loopback + 每次启动的 32 字节共享密钥自行取得带 token 的 URL。

⚠️ 见 0.1.1：**这个版本无法加载**，请直接升级。
