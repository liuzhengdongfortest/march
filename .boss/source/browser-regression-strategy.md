# 浏览器回归策略

最后更新：2026-05-08

## 决策：保留 CDP smoke 作为快速 Chromium 回归

**为了谁**：服务于 [产品壳层需求](../requirements/product-shell.md)，保证本地 Web 的主路径能被快速、低依赖地验证。

**怎么设计**：现有 `npm run browser-smoke` 保持为 Chrome/Edge/Chromium 的快速 smoke。它直接启动浏览器，打开 `--remote-debugging-port`，用 Chrome DevTools Protocol 执行 DOM 断言和截图。

**为什么这样**：这条链路不需要额外 test runner，不下载浏览器，适合本机和 CI 的最小必跑；并且已经覆盖 diff、Agent board、私聊 board、订阅配置、窄屏布局等高价值路径。

## 决策：Firefox/WebKit 走 Playwright 适配层，不复刻 CDP

**为了谁**：服务于历史产品工程基线计划里“尚未覆盖 Firefox/WebKit 等非 Chromium 差异”的缺口；若恢复 Web 产品工程方向，应迁入新的 Roadmap。

**怎么设计**：

- 引入一个后续脚本 `browser-smoke-playwright.mjs` 或 `browser-smoke-engines.mjs`。
- 把现有 smoke 中的业务断言抽成可复用场景，底层 runner 提供 `evaluate`、`screenshot`、`setViewport`、`close` 等少量能力。
- CDP runner 继续支撑现有 `browser-smoke`。
- Playwright runner 使用同一套场景跑 `chromium`、`firefox`、`webkit`。

**为什么这样**：Firefox/WebKit 没有当前脚本依赖的 Chrome remote debugging 入口；Playwright 官方支持 Chromium、Firefox、WebKit 三类引擎，也支持 CI 中安装浏览器和系统依赖。用一个 runner 抽象复用业务断言，比维护两套 smoke 更稳。

## 当前能力边界

现有 `browser-smoke` 覆盖：

- 桌面布局无横向溢出。
- diff work block 展开、文件导航、分页、split view、批注/reply/resolve/delete。
- GitHub/Sentry/GitLab 订阅配置入口、GitHub 编辑、subscription 停用。
- Agent board 群聊/私聊快照、专用 SSE、增量状态。
- 窄屏布局和截图。

现有 `browser-smoke-matrix` 覆盖：

- 本机已安装的 Edge、Chrome、Chromium。
- CI 当前跑 Chrome 必选、Edge 可选。

现有 `browser-smoke-engines` 覆盖：

- Firefox/WebKit 桌面布局无横向溢出。
- diff work block 可展开，文件导航、diff 行和 split diff 可渲染。
- Agent board 可打开，per-agent SSE 可推送 `agent_board` 快照。
- provider account 列表、reconnect row 和 GitHub/GitLab/Sentry selector 可渲染，选择后写入 `auth:acct_*`。
- 窄屏布局无横向溢出。

未覆盖：

- Firefox Gecko 引擎下完整订阅创建/编辑业务路径。
- WebKit/Safari 类引擎下完整订阅创建/编辑和 EventSource 恢复业务路径。
- Playwright trace/report 级别的失败复现。

## Playwright 成本评估

- 依赖成本：需要新增 dev dependency，并在 CI 安装 browser binaries。
- 体积成本：Playwright 浏览器缓存通常是数百 MB 级别。
- CI 成本：`npx playwright install --with-deps` 会增加安装时间；Firefox/WebKit 初期建议非必选或 nightly。
- 语义成本：Playwright 的 WebKit 是跨平台 WebKit 构建，不等同于真实 macOS Safari，但能覆盖主要布局/DOM 引擎差异。
- 隐私成本：trace/report/screenshot 可能包含敏感数据；当前测试数据是本地 fixture，后续不能把真实 token 或用户内容写入 artifact。

## CI 策略

第一阶段：

- 必跑：现有 `browser-smoke` Chrome。
- 可选：现有 `browser-smoke` Edge。
- 已新增：`npm run browser-smoke-engines` 本地手动命令，不进 PR 必跑。

第二阶段：

- PR 必跑：Playwright `chromium` + `firefox`，WebKit 允许失败或只在 main/nightly 跑。
- artifact：失败时上传截图，不默认上传完整 trace。

第三阶段：

- 三引擎全部必跑：当 smoke 场景稳定且运行时间可接受后，再把 WebKit 从 optional 升级。

## 本地命令建议

当前 package scripts 已加入：

```json
{
  "browser-smoke": "node apps/server/src/browser-smoke.mjs",
  "browser-smoke-matrix": "node apps/server/src/browser-smoke-matrix.mjs",
  "browser-smoke-engines": "node apps/server/src/browser-smoke-engines.mjs"
}
```

`browser-smoke-engines` 默认只跑 `firefox,webkit`，并在缺少 Playwright 浏览器时给出 `npx playwright install firefox webkit` 安装提示，而不是自动下载。

## 下一刀建议

Playwright runner POC、Agent board 快照和 provider account selector 迁入已完成。下一步不要一次迁移所有断言，先评估是否把 Firefox 设为 optional CI，或再迁一条完整订阅创建/编辑场景。

## 官方约束来源

- Playwright 浏览器支持：<https://playwright.dev/docs/browsers>
- Playwright GitHub Actions / CI：<https://playwright.dev/docs/ci-intro>
- Playwright install CLI：<https://playwright.dev/docs/test-cli>
