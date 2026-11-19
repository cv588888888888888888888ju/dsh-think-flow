# 中文增强 fork：相对上游 0.3.3 改了什么

本仓库是 [yfwu2020/dsh-think-flow](https://github.com/yfwu2020/dsh-think-flow) 0.3.3 的 fork。
目标：**在不影响主模型工作的前提下，让另一个模型维护可视化的状态** ——

- 主模型的系统提示词、工具清单、上下文**一个字节都不加**；
- 插件自己另发一次请求，把每一步的英文思考压成「中文标题 + 中文解释」，把整轮压成一条中文标题；
- 走哪个模型由配置决定，因此可以把这件旁路活交给免费/小模型。

## 为什么 fork

上游的 `resolveRoute()` 固定取 `agentDefaultModel`（主会话模型）来给主会话的思考起标题 ——
每轮都拿主模型的额度去写小标题，而且用户还得每开一个会话手点一次「自动」。

本 fork 把标题生成**拆成独立路由**（`titleProvider` / `titleModel`），并让「有人看面板」这件事
自动臂上标题生成（`autoTitlesOn`）。

## 改了什么（按文件）

宿主侧（TypeScript，`src/` → `lib/` 由 `npm run build` 重建）：

### `src/host/config.ts`
- 新增 `titleProvider` / `titleModel`：两者都非空才生效，空 = 沿用主模型（上游行为）。
- 新增 `autoTitlesOn`（默认 `false`）：面板一连上就自动臂上标题，不必手点，宿主重启也不丢。
- `Config`（schemastery）与 `resolveConfig()` 两处都加（缺一处则程序化调用拿不到默认值）。

### `src/titles.ts`
- `MAX_EXPLAIN_CHARS = 240`：单条解释的字数上限（客户端本来也只显示两行）。
- `TITLE_SYSTEM` 重写：输出从 `{"titles":[...]}` 改成
  `{"steps":[{"title","explain"}],"notes":{...},"turnTitle"}`。
  标题 8~18 汉字照旧；`explain` 20~60 汉字，要求说「在查什么 / 为什么这么查 / 得出了什么」，
  是标题的补充而不是复述；标题**不夹英文单词**（专有名词除外）。
- `parseTitles()`：读 `data.steps`（对象数组），旧的 `titles:["…"]` 仍然兼容（此时 explains 全空）；
  **条数必须相等**这条硬约束没变 —— 错位的标题比没有标题更糟。

### `src/host/title-service.ts`
- `resolveRoute()`：先看 `titleProvider`+`titleModel`，没配才退回 `agentDefaultModel`。
- `generateBatch()` **按 `MAX_STEPS_PER_CALL = 10` 分段**：一轮 100+ 步时一次性要上百条标题，
  模型必然少给，条数校验一挡就整轮都没有。分段后每段各自校验，某段失败只丢那几步。
  工具说明只随第一段发、整轮输出只随最后一段发。
- `mergeIntoCache()`：缓存每步存 `{title, explain, print}`（`print` 是内容指纹，判失效用）。
- `generateTitles()`：命中缓存时把 `explain` 一起收回；返回值带 `explains`。
- `pushTitles()`：SSE `titles` 消息带上 `explains`（空则不推）。
- 新增 `armAuto(sessionId)`：`autoTitlesOn` 为真时，面板 SSE 一连上就自动开启标题。
- `titleable()` / `titleInputOf()` / `toolMaterialOf()`：**没有思考、只有工具调用的步**也纳入标题生成，
  素材换成工具名 + 参数（`run_code` 的 `code` / `description` 前 240 字）。
  以前这种步只有一行原始参数（`run_code {"code":…`），读者看不懂它在干什么。
- `generateBatch()` **分段重试**：每段最多 3 次，间隔 600ms / 1800ms；某段三次都失败才丢那几步。
- 自动节拍器**失败退避**：连续失败按 30s → 60s → … → 300s 退避，不再每 5s 撞一次免费通道。
- **分段流式出标题**：`generateBatch()` 收 `onPart` 回调，每段成功就立刻落缓存 + 推 SSE；
  自动与手动两条路都走它。以前是整轮算完才一次性推送，100+ 步的轮次要空白四五十秒。

### `src/host/snapshots.ts`
- `turnSnapshot()` 增加 `explains`（按步号索引的对象，和 `titles` 一样要指纹对得上才下发）。

### `src/host/routes.ts`
- SSE 订阅成功后调用 `titles.armAuto(sessionId)`。
- `/api/ping` 的 `build` 24 → **29**（重启后用它确认新代码在跑）。

### `src/host/types.ts`
- 标题缓存条目 `steps[key]` 增加可选 `explain`。

客户端（`src/client/` → `lib/client.js` 由 esbuild 打包）：

### `src/client/components/step.js`
- `createStepRenderer` 接收 `explainsRef`；有解释时在步骤行下面加一行 `.tf-step-explain`
  （两行封顶，悬停看全文）。

### `src/client/panel.js`
- 新增 `explainsRef`；快照 / SSE `titles` / `POST /titles` 三条路都写入。

### `src/client/styles.css`
- 增补 `.tf-step-explain` 规则（以及「进行中」状态的着色）。

### `tests/test-routes.mjs`
- 一条断言跟着提示词字段改名（`titles` → `steps`）。

## 怎么用

```yaml
# profile 的 cordis.patch.yml，覆盖 id: think-flow 那一行
- id: think-flow
  config:
    titleProvider: opencode-go-free   # 任意 OpenAI 兼容 provider
    titleModel: hy4-flash-free
    titleReasoningEffort: 'off'       # 免费通道别填 'none'，会被直接拒绝
    autoTitlesOn: true
```

两个键留空 = 完全回到上游行为（用主会话模型起标题）。

## 实测（真实会话，force 重算）

| 轮次步数 | 耗时 | 结果 |
| --- | --- | --- |
| 113 步 | 49.2s | 71 条标题（当时只给带思考的步） |
| 24 步（含 7 个无思考的工具步） | 14.0s | 23/23，无思考的步也拿到了标题 + 解释 |
| 同轮，流式（3 段） | 11.9s | SSE 上 4 条 `titles`，间隔约 4.4s / 4.4s / 2.6s —— 边算边出 |
| 20 步 | 13.7s | 20/20 |
| 11 步 | 10.2s | 11/11 |
| 7 步 | 4.0–5.6s | 7/7 |

对照：同一批轮次用大模型（longcat-2.5-free）20 步要 70s；换回「一次性整批」时 113 步轮直接整轮失败。

## 构建与自检

```bash
npm ci
npm run build      # tsc → lib/host, lib/titles.js…；esbuild → lib/client.js
npm test           # 900+ 断言
```

本仓库的 `src/` 与 `lib/` 是同步的：`lib` 是构建产物，不要手改；
改功能请改 `src/` 再 `npm run build`。

## 已知副作用

- 失败退避期间该会话的标题会停更（最长 5 分钟），期间步照常在面板上滚，只是没有中文标题。
- 无思考的工具步也要调模型，轮次很长时调用次数会略增（每段多算几步，仍在 10 步/段的批量里）。

## 与上游的关系

- 上游版本：0.3.3（MIT）。
- 本 fork 版本：0.4.2，包名 `dsh-think-flow-zh`，插件 id 仍是 `think-flow`。
- 与上游**不能同时安装**（同一个插件 id）。
