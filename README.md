# OpenDesign 本地 OpenCode 超时修复 + 禁用自动升级（专用版）

## 问题

使用本地 `opencode` 作为 AI 服务时会超时，主要原因：

1. **Agent 连接测试超时**：打包版启动时会检测本地 Agent，默认 45s 超时时间对本地 opencode 较慢的冷启动不够。
2. **聊天运行不活动看门狗超时**：守护进程默认 10 分钟无新输出就判定为卡死并终止运行。本地 opencode 长时间思考/工具调用时可能长时间无 stdout。
3. **自动升级风险**：专用定制版不希望自动升级覆盖本地修改。

## 源码修改

### 1. Agent 连接测试超时支持环境变量
文件：[`apps/daemon/src/connectionTest.ts`](apps/daemon/src/connectionTest.ts)

```diff
- const DEFAULT_AGENT_TIMEOUT_MS = 45_000;
+ const DEFAULT_AGENT_TIMEOUT_MS = (() => {
+   const raw = process.env.OD_CONNECTION_TEST_AGENT_TIMEOUT_MS;
+   if (raw === undefined || raw === '') return 45_000;
+   const n = Number(raw);
+   const MAX = 2_147_483_647;
+   if (Number.isSafeInteger(n) && n > 0 && n <= MAX) return n;
+   console.warn(
+     `connection-test: ignoring OD_CONNECTION_TEST_AGENT_TIMEOUT_MS=${JSON.stringify(raw)} (must be a positive integer between 1 and ${MAX} ms); using 45_000ms`,
+   );
+   return 45_000;
+ })();
```

### 2. 为本地 OpenCode 单独增加不活动超时
文件：[`apps/daemon/src/runtimes/defs/opencode.ts`](apps/daemon/src/runtimes/defs/opencode.ts)

在 `opencodeAgentDef` 最后增加：

```diff
     externalMcpInjection: 'opencode-env-content',
+    inactivityTimeoutMs: (() => {
+      const raw = process.env.OD_OPENCODE_INACTIVITY_TIMEOUT_MS;
+      if (raw === undefined || raw === '') return 60 * 60 * 1000; // 默认 60 分钟
+      const n = Number(raw);
+      if (Number.isSafeInteger(n) && n >= 0) return Math.min(24 * 60 * 60 * 1000, n);
+      return 60 * 60 * 1000;
+    })(),
 } satisfies RuntimeAgentDef;
```

### 3. 聊天运行不活动超时支持环境变量
文件：[`apps/daemon/src/runtimes/chat-run-lifecycle.ts`](apps/daemon/src/runtimes/chat-run-lifecycle.ts)

```diff
- const DEFAULT_CHAT_RUN_INACTIVITY_TIMEOUT_MS = 10 * 60 * 1000;
+ const DEFAULT_CHAT_RUN_INACTIVITY_TIMEOUT_MS = (() => {
+   const raw = process.env.OD_CHAT_RUN_INACTIVITY_TIMEOUT_MS;
+   if (raw === undefined || raw === '') return 10 * 60 * 1000; // 默认 10 分钟
+   const n = Number(raw);
+   if (Number.isSafeInteger(n) && n >= 0) return Math.min(24 * 60 * 60 * 1000, n);
+   return 10 * 60 * 1000;
+ })();
```

## 打包与定制

1. 构建并打包 macOS DMG：
```bash
OD_CONNECTION_TEST_AGENT_TIMEOUT_MS=300000 OD_OPENCODE_INACTIVITY_TIMEOUT_MS=7200000 pnpm tools-pack mac build --to dmg
```

2. 安装 DMG 到 `/Applications/Open Design Fixed.app`

3. 禁用自动升级（已在打包产物内修改）  
   修改打包后的 `Contents/Resources/app/prebundled/packaged-main.mjs`：

```js
const defaultEnabled = false;
const enabled = false;
```

将更新检测强制禁用，避免专用版被自动升级覆盖。

4. 替换内置 opencode 为本地新版  
   将本地 `opencode`（如 1.19.0）复制到：
```
Open Design Fixed.app/Contents/Resources/open-design/bin/libexec/opencode/opencode
```
并同步为 `opencode-cli`。

## 运行

推荐使用环境变量启动：

```bash
launchctl setenv OD_CONNECTION_TEST_AGENT_TIMEOUT_MS 300000
launchctl setenv OD_OPENCODE_INACTIVITY_TIMEOUT_MS 7200000
launchctl setenv OD_CHAT_RUN_INACTIVITY_TIMEOUT_MS 7200000
open /Applications/Open\ Design\ Fixed.app
```

或直接命令行启动：

```bash
OD_CONNECTION_TEST_AGENT_TIMEOUT_MS=300000 \
OD_OPENCODE_INACTIVITY_TIMEOUT_MS=7200000 \
OD_CHAT_RUN_INACTIVITY_TIMEOUT_MS=7200000 \
/Applications/Open\ Design\ Fixed.app/Contents/MacOS/Open\ Design
```

## 环境变量说明

| 变量 | 默认值 | 说明 |
|---|---|---|
| `OD_CONNECTION_TEST_AGENT_TIMEOUT_MS` | 45000 | Agent 连接测试超时（毫秒），适配本地 opencode 冷启动较慢的情况 |
| `OD_OPENCODE_INACTIVITY_TIMEOUT_MS` | 3600000（60分钟） | 本地 opencode 运行时不活动看门狗超时（毫秒），针对长时间思考 |
| `OD_CHAT_RUN_INACTIVITY_TIMEOUT_MS` | 600000（10分钟） | 全局聊天运行不活动看门狗超时（毫秒），兜底用 |
