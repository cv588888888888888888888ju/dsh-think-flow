## 相对上游 0.3.3 的改动

- 标题/解释走**独立路由** `titleProvider` / `titleModel`（两个都非空才生效；空 = 主会话模型）。主模型的系统提示词、工具、上下文零改动，插件只在旁路多发一次请求。
- `autoTitlesOn`：面板一连上就自动补标题，不必每个会话手点「自动」，宿主重启也不丢。
- 提示词改成「标题 + 解释」两段式：每步一行 8~18 字中文标题 + 20~60 字中文解释；整轮一条中文标题。解释是模型直接写的中文，不经过翻译器。
- 标题生成按每段 ≤10 步**切段**调用：113 步的轮 49.2s 出 71 条，不再因条数不符整批丢弃。
- 客户端在步标题下多一行解释（两行封顶）；快照、SSE、手动生成三条路都带 `explains`。
- `/think-flow/api/ping` 的 build 号为 **27**。

## 安装

```bash
dsh plugin --profile web add https://github.com/cv588888888888888888888ju/dsh-think-flow/releases/download/v0.4.0/dsh-think-flow-zh-0.4.0.tgz
```

与上游不能同时安装（同一个插件 id `think-flow`）。