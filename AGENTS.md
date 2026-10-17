# 给 Agent 的说明

这是一个行业热点网站的框架：采集信源、用模型筛选和写作、归组事件、出日报，并通过网站、RSS、公开 API 和 MCP 对外提供。默认配置是一个 AI 行业的示例站。先读 README，再按任务读 `docs/` 里对应的文档。

## 最常见的任务：改成另一个行业

按 `docs/customize.md` 的顺序做。行业相关的一切都在 `industry/`：站名文案（`site.ts`）、分类标签（`taxonomy.ts`）、主题（`topics.json`）、示范信源（`sources.json`）、提示词（`prompts/`）、门槛（`selection.ts`）、模块开关（`features.ts`）、品牌（`brand/`）、条款页（`pages/`）。通常不需要改 `apps/` 和 `packages/`。

这些事要问使用者本人，不要替他决定：站名；要盯哪些信源；什么消息重要、什么是噪声；分类怎么分；条款和隐私说明的内容（`industry/pages/` 是模板，上线前需要他本人确认）。

改评分标准时保留原有结构（内容类型、五个维度加权、噪声压制、安全边界），替换的是“什么算重要”“什么算噪声”的例子。门槛要用使用者标注的样本重新校准（`docs/selection.md`），不要凭感觉改数字。

## 运行与检查

- Node.js 24 直接运行 TypeScript，后端没有构建步骤。npm workspaces：`apps/*`、`packages/*`、`industry`。
- 本机运行和 Docker 见 `docs/deploy.md`。
- 改完至少跑：
  ```bash
  npm run typecheck
  DATABASE_URL=postgres://127.0.0.1:5432/<名字>_test npm test   # 空库，名字必须以 _test 或 _ci 结尾，先 node scripts/migrate.ts
  npm run build -w @aihot/web && node --test apps/web/tests/*.test.ts
  node scripts/smoke.ts --base http://localhost:3000             # 站点跑起来以后
  ```
- `tests/` 里部分测试用的是示例行业的分类、标签和公司，改了 `industry/taxonomy.ts` 后把这些例子换成新行业的对应项。

## 要守住的规则

- 前端（`apps/web`）只通过 HTTP 读 `apps/api`，数据库、模型调用和密钥只在后端。
- 所有公开出口都从 `packages/backend/src/publication/` 这一个读取层读，新增公开出口也一样。
- 读者打开页面不触发模型调用；模型只在 worker 的任务里调用。
- 付费请求都经过回执（`providers/receipts.ts`）和预算熔断，不要绕开。
- 开发和测试时保持安全阀关闭：`COLLECT_ENABLED`、`MODEL_CALLS_ENABLED`、`FEISHU_*_ENABLED`、`INDEXNOW_SUBMIT_ENABLED`。测试不访问任何外部服务。
- 信源默认只展示摘要和原文链接（`site_fulltext` 关）；只有来源明确允许时才打开全文。
- 公开内容匿名，管理员和访客看到的一样；后台只允许管理员。
- 数据库迁移只做向后兼容的增量，新迁移按编号加在 `database/migrations/` 末尾。
- 不要提交 `.env`、密钥和 `.data/`。
- 不要使用 AIHOT 的名字和 Logo。

## 写代码

匹配周围代码的写法、命名和注释密度。选能清楚解决问题的简单方案，只定义正在使用的抽象。验证改动涉及的重要行为，不为简单的样式改动写测试。

## Cursor Cloud specific instructions

Cloud Agent 开机时会跑环境的 `start`：PostgreSQL、迁移、示范数据，以及三个开发进程。依赖已在 `install` 里装好，一般不用再执行 `npm ci`。

- Node.js 24.22 安装在 `/usr/local/bin`，并链接到 `/usr/local/cargo/bin`（该目录在 PATH 最前）。平台自带的 `/exec-daemon/node` 是 22，直接调用那个路径会低于仓库要求的 `>=24.11`。
- PostgreSQL 16 只监听本机，`127.0.0.1` 使用 trust。站点库是 `aihot`，测试库是 `aihot_test`。连接串：`postgres://ubuntu@127.0.0.1:5432/aihot`。测试用 `DATABASE_URL=postgres://ubuntu@127.0.0.1:5432/aihot_test npm test`（库已迁移）。
- 第一次启动会在仓库根目录生成 `.env`（已 gitignore）。采集和模型调用是关的：`COLLECT_ENABLED=false`、`MODEL_CALLS_ENABLED=false`，飞书和 IndexNow 也关着。`DEV_AUTH_ROLE=admin`，所以开发时打开 `/admin` 不用密码。这里没有真实的模型 API Key；要连模型时再把 key 写进 `.env`，并把 `MODEL_CALLS_ENABLED` 改为 `true`。
- 三个 tmux 会话：`aihot-api`（`npm run dev:api`，3001）、`aihot-worker`（`npm run dev:worker`）、`aihot-web`（`npm run dev:web`，http://127.0.0.1:3000）。日志分别在 `/tmp/aihot-api.log`、`/tmp/aihot-worker.log`、`/tmp/aihot-web.log`。网页把 `/api/*` 转到 3001。
- 模型榜页面在还没有一轮结果时返回 503，这是空库的正常状态。`node scripts/smoke.ts --base http://127.0.0.1:3000` 会把这种情况记为跳过。
