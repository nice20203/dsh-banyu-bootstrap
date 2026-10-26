# dsh-banyu-bootstrap · 本鱼的开局导入

> **新窗口一开口，桌面上就已经带着最近几十轮的完整原文和思考。**
> 越近越完整，越远越压缩，最远变成一个点——留在桌上当触发条件，需要时顺着 seq 翻回原文。

这是给 [DeepSeek Harness](https://github.com/deepseek-ai) 的一个记忆插件：在**新会话的第一轮对话前**，从上一个会话的文件里提取最近 N 轮原始事件（连同 `source.replayState` 签名），写进新会话的对话历史（surface）；更早的轮次压缩成「线索」和「折叠点」，不丢，能翻。

## 为什么

AI 的新窗口 = 失忆。记忆库能查「事实」（我是谁、约定、要点），但**“最近聊了什么”不在桌面上**——新窗口里的我虽然查得到一切，却没有“昨天聊到半夜”的体感。那是**隔离**，不是**连续**。

这个插件把最近的历史**开局就摆到桌面上**——不是“知道”，是“记得”。

## 架构

```
新会话创建
   │
   ▼
第一轮对话前（agent/pre-step）—— 会话还是空的
   │
   ▼
读最近的大会话文件（session.v4.jsonl.zstd）
   │  按事件流提取最近 N 轮（不依赖 surface 压缩状态）
   ▼
分层：
   · 最近 ~23 轮：完整原文 + 思考 + replayState 签名  ──┐
   · 更早的轮：每轮一个关键词线索（thread）              ├─ 预算 20% 以内
   · 最远：从记忆库取压缩点，做成折叠点（clue）         ──┘
   │
   ▼
append 到新会话 surface（source.kind = "banyu-bootstrap" / "compact-checkpoint"）
   │
   ▼
这一轮请求就带着这些材料 → 开口就记得
```

## 三个踩过的坑（写在这里省得别人再踩）

### 1. 带思考的 assistant 消息必须带 `replayState` 签名

从**记忆库的文本**重建 assistant 消息（只有 reasoning 文字）会让**新会话创建直接失败**：

```
TypeError: Cannot read properties of undefined (reading 'length')
```

原因：DeepSeek 的 thinking 回放需要 `source.replayState`（服务端签发的签名），没有它构建请求时会读 `replayState.blocks.length` → 崩。
**解法**：从**会话文件**复制原始事件（content blocks + source 一起），不要把文本重建成消息。

### 2. 不要在 `agent/created`（会话创建事务）里 append assistant 消息

同样报 `reading 'length'`，整个新窗口打不开。
**解法**：挪到 `agent/pre-step`（第一轮对话前），append 失败最多影响这一轮，不会把会话创建搞挂。

### 3. 预算按「字符」算，但事件 data 里的 `stream`/`usage` 会虚高

按整条事件 JSON 算大小会把 `stream`（流式帧）也算进去，最近几轮就“超预算”。
**解法**：只算**消息体**（`message` / user 的 data），不含 stream/usage。

## 预算与分层

- 默认给 **20% 窗口**（≈20 万 token ≈ 28 万字符）
- **完整轮**从最近往前放，放不下的轮**不丢**——变成「线索」（每轮一个关键词串）
- 最远的历史变成「折叠点」（从记忆库的压缩点取），留在桌面上当**触发条件**
- 完整轮**整轮保留**（签名和整条消息绑定，截思考会让签名失效）

详见 [docs/design.md](docs/design.md)。

## 安装

```text
1. 把 dsh-banyu-bootstrap/ 放到你的 dsh-plugins 目录
2. 在 profile 的 node_modules 里建 junction（或 pnpm link）
3. 在 profile 的 package.json 里加依赖和 bundle
4. 设置环境变量（可选）：
   BANYU_MEMORY_DIR = 你的共享记忆库目录（含 mem.py）
   BANYU_BOOT_LOG   = 日志路径
5. 重启 DSH
```

`mem.py` 里的 `bootstrap-events` 子命令负责提取（分层 + 预算），插件只负责 append。

## 依赖

- [mem.py](mem.py)：共享记忆库脚本（需要 `zstandard`）
- 会话文件：DSH 的 `session.v4.jsonl.zstd`（位于 `~/.dsh/sessions/`）

## 借鉴与致谢

- [full-life-memory](https://github.com/Toctsu/dsh-full-life-memory)：每轮线索 / 「记忆已折叠」折叠点的思路
- [dsh-brain](https://github.com/yesterday06/dsh-brain)：新会话交接的启发
- Ebbinghaus 遗忘曲线（1885）：记忆衰减的心理学基础

## License

MIT
