# Jev Ultrafast ⚡

> [!NOTE]
> **本文是 [browser-use/jev-ultrafast](https://github.com/browser-use/jev-ultrafast) 的 NeoHorse-Jev-4B 改编复刻。**
> 决策模型已替换为 `tokenrhythm.studio` 上的 NeoHorse-Jev-4B，因此本复刻不需要
> Browser Use Cloud 账号，也不需要任何付费第三方决策服务。它同时作为同名 ZCode 插件
> `neohorse-jev-browser-use` 的技能发布。上游采用 MIT 许可，本复刻保持相同许可。
> 具体差异见 [NeoHorse-Jev-4B 改编说明](#neohorse-jev-4b-改编说明)。
> 英文自述见 [README_EN.md](README_EN.md)。

> [!IMPORTANT]
> **Browser Use Cloud 等待列表已开放。** 可抢先体验云端极速浏览器智能体。
> **[加入等待列表 →](https://browser-use.com/ultrafast?utm_source=github&utm_medium=readme&utm_campaign=jev-ultrafast)**

**一个拥有动态索引动作空间的浏览器智能体。**

给它一个目标。[NeoHorse-Jev-4B](https://tokenrhythm.studio) 决策模型选择操作和元素；
只有操作是 `TYPE_TEXT` 时，才由一个小语言模型生成要输入的文本。

决策循环见 [agent.py](jev_ultrafast/agent.py)，原子 DOM 快照见
[snapshot.js](jev_ultrafast/snapshot.js)，决策头见
[model.py](jev_ultrafast/model.py)。本仓库不附演示录像和实测数据——
见[证据与边界](#证据与边界)。

## 动作空间

每次观测都会生成一张新的元素表：

```text
[1] button    Change ticket type · Round trip
[2] combobox  Where from?        · San Francisco
[3] combobox  Where to?          · empty
[4] textbox   Departure          · empty
...
```

支持的操作有 `CLICK`、`TYPE_TEXT`、`SELECT`、`SCROLL_UP`、`SCROLL_DOWN`、`WAIT`、`DONE`、`BLOCKED`。只会提供受支持的操作和目标。

```text
                      one NeoHorse-Jev-4B request
                     ┌───────────────────────────┐
page → element table → operation                 │
                     │ click_target              │
                     │ type_text_target          │
                     │ select_target, if present │
                     └─────────────┬─────────────┘
                         use the matching target
                                   │
                    CLICK [7] ─────┤──→ browser
                TYPE_TEXT [3] ─────┘
                          ↓
                   small LLM → text → browser
```

目标问题是投机性的：如果操作是 `CLICK`，只有 `click_target` 可以执行。两个决策，**一次网络往返**。每个目标头只包含相容的元素。原生下拉选项携带观测到的元素/选项索引。

策略中没有站点专用的动作脚本或预填字段。Flights 示例只提供目标并独立验证结果。

## NeoHorse-Jev-4B 改编说明

本副本向 `https://tokenrhythm.studio/v1/decision` 发送决策请求，模型固定为
`NeoHorse-Jev-4B`，平台密钥从 `NEO_HORSE_API_KEY` 读取，替换了上游的 TypeSafe
端点。仅允许向该主机发起 https 请求；主机解析到私有、环回、链路本地或保留地址时
一律拒绝。平台在 choice 答案中不返回 `confidence`，因此按可选处理。密钥只从环境
变量读取；切勿放入提示词、请求文件或受版本控制的文件。

**隐私：** 每个决策周期都会把页面 URL、可见文本和索引元素表发送到决策端点，
`TYPE_TEXT` 还会把字段上下文发给文本模型。在对包含私密或敏感内容的页面运行
智能体前，先获得用户同意；切勿在目标或证据中包含凭据、cookie 或令牌。

## 试用

```bash
git clone https://github.com/himetuki/neohorse-jev-ultrafast.git
cd neohorse-jev-ultrafast
uv sync
cp .env.example .env
# 填入 NEO_HORSE_API_KEY 和 TEXT_MODEL_API_KEY。
uv run jev
```

打开 **http://127.0.0.1:8766**，点击 **Start demo → Run automatically**。
检查器会显示编号元素、操作概率、目标概率和已执行的动作。
**Choose next** 会在执行前暂停。

Chrome 通过 [Browser Harness](https://github.com/browser-use/browser-harness)
连接（由 `uv sync` 安装）。需要连接时运行 `uv run browser-harness --doctor`；
Chrome 弹出远程调试授权时请允许。

`NEO_HORSE_API_KEY` 是 tokenrhythm.studio 平台密钥。示例配置中
`TEXT_MODEL_API_KEY` 使用 OpenRouter 密钥，当前演示模型为 `inception/mercury-2.5`
（关闭推理）。Gemini、GLM、DeepSeek 也可用于这个 OpenAI 兼容的文本辅助端点，
配置对应的模型、端点和推理开关即可。

## 使用库

```python
from jev_ultrafast import Agent

with Agent(
    "https://www.google.com/travel/flights?hl=en",
    "Find one-way flights from Zurich to London on September 20, 2026, "
    "for one adult in economy. Stop when matching flight options are visible.",
) as agent:
    for state in agent.run():
        print(state["elapsed_ms"], state["status"])
```

用 `uv run --env-file .env python your_script.py` 运行。同一策略可以跑不同任务：

```bash
uv run --env-file .env python examples/run.py \
  --url https://en.wikipedia.org/wiki/Main_Page \
  --goal 'Find and open the Wikipedia article about Gödel’s incompleteness theorems.'
```

`uv run --env-file .env python examples/flights.py --keep-open` 会完成航班搜索，
核对真实的航线/日期/结果并保存轨迹；不会选票或订票。

## 为什么它快

- **每个决策周期只发一次请求。** 操作头和目标头共享同一次观测状态。
- **默认循环不使用截图。** 决策模型消费结构化状态；检查器按需开启截图。
- **每次快照只调用一次浏览器。** 原子读取可见控件、名称、取值和文本，并保留对真实 DOM 节点的引用。
- **校验选中的目标。** 点击前检查文档、表单取值、目标和邻近上下文；仅凭动画不触发重新预测；执行前解析当前几何并拒绝被遮挡的控件。
- **等待有用状态。** 向组合框输入后等待可见建议（上限 200 ms）；其他交互最多等两个动画帧或 50 ms。这些读取发生在执行日志之后。
- **保持隐藏标签页渲染。** 焦点模拟避免后台标签页动画降频，无需切换 Chrome 可见标签页。
- **只发送可见文本。** 视口外的正文和页脚不进入模型上下文。
- **复用被打断的文本请求。** 只有当文本辅助的完整输入不变时，生成值才可在页面过期重试后复用。

每个执行的目标都从观测节点解析。执行器会重新检查页面新鲜度和点击遮挡。
模型输出永远不会变成选择器、坐标、shell 命令或可执行 JavaScript。
文本辅助的输出必须先解析为一个小 JSON 对象才会被输入。

## 小到可以读完

| 文件 | 职责 |
| --- | --- |
| [agent.py](jev_ultrafast/agent.py) | 完整循环与文本辅助交接 |
| [snapshot.js](jev_ultrafast/snapshot.js) | 原子 DOM 快照、索引控件、新鲜度守卫 |
| [browser.py](jev_ultrafast/browser.py) | 浏览器连接、当前几何、执行 |
| [model.py](jev_ultrafast/model.py) | 动态操作/目标头与文本生成 |
| [questions.py](jev_ultrafast/questions.py) | 模型指令 |
| [demo.py](jev_ultrafast/demo.py) | 本地检查器 |

## 证据与边界

本复刻**不提供任何自己的性能证据**。上游项目针对 TypeSafe 模型发布过实测
运行（Google Flights 录像、对照比较和任务耗时）；这些结果没有在
NeoHorse-Jev-4B 端点上复现，已从本仓库移除。请先用
`python scripts/measure_flights.py --output <folder>` 实测（一次会写入
自己证据的实时运行），在此之前，一切速度、成本或可靠性数字都视为未知。

`DONE` 选择仍然需要独立的结果验证。DOM 读取器覆盖常见 HTML 和 ARIA 控件，
并非完整的可访问名规范。Shadow DOM、框架、canvas、上传、弹窗标签页、
嵌套滚动和任意键盘控件仍在 MVP 之外。被接管的标签页共享同一个 Chrome 配置文件。

## 开发

```bash
uv run ruff check .
uv run pytest
node --check jev_ultrafast/static/app.js
node --check jev_ultrafast/snapshot.js
uv build
```

测试全程离线。`uv run python scripts/check_guards.py` 在本地浏览器中检查真实
控件，不调用模型。实时示例、`scripts/measure_flights.py` 和 `scripts/smoke.py`
会产生付费 API 调用。凭据和原始轨迹保持忽略状态。

---

[Browser Use](https://github.com/browser-use/browser-use) · [Browser Harness](https://github.com/browser-use/browser-harness) · NeoHorse-Jev-4B 决策端点：`https://tokenrhythm.studio/v1/decision` · 英文版：[README_EN.md](README_EN.md)
