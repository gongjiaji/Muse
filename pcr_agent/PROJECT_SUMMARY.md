# PCR Agent 项目讨论总结

> 给 Claude Code 看的项目背景和改造要求。最后更新：2026-11-08

## 1. 项目目标

审计 ITSM 里的 Change Request（CR），检查 CR 申明的 implementation plan
和工程师实际在设备上执行的命令日志是否匹配。

- CR 说做 A，实际执行的是做 B，就是 unauthorized change，要抓出来。
- 输入：一个 CR 号。输出：是否 unauthorized、依据是什么、问题出在哪、为什么。

## 2. 现状（重要：不要重建平台）

- 已经有一个完整项目在跑，只是用 one shot prompt：一次把 plan + logs + SOP 全塞给模型，一次返回结果。
- **false positive 很高**，这是要解决的主要问题。
- 平台、API、数据库、Data Collector、前台都已经有了，**只改 Agent 层，不要动它们**。
- 保留现有的 one shot 版本当兜底，新版本先并行跑，对比后再切换。

## 3. 数据和规则

### 数据来源

```
ITSM / Graylog / SOP -> Data Collector 服务 -> 数据库
数据库 -> 平台后端 API -> 前台（显示 CR 信息、log 信息）
数据库 -> Agent 工具 -> LangGraph 循环 -> 判断结论
```

- 前台要显示 CR 和 log 信息，所以数据必须存数据库，不能现拉不存。
- 前台和 Agent 读同一份存量数据，避免显示和判断不一致。
- Collector 触发：CR 创建时 / 平台点刷新按钮时重新拉取；SOP 可按计划（例如每天）同步。

### Plan

- CR 里的 free-text 字段，说明要做什么，有时包含实际命令，**不是结构化数据**。
- 先由模型把它拆成结构化清单：设备、动作、如果有写明的命令也列出来。

### Logs（Graylog API，结构化 JSON）

- 每条记录：谁、在哪台设备、用什么 ID、什么时间、执行了什么命令。
- 一个 CR 跑 100 条命令，就有约 100 条记录。

### SOP

- 例外 / allowlist：如果做了 plan 里没有的东西，**SOP 能 cover 就没事**，输出要引用 SOP 编号，例如 SOP123，并引用原文。

### Unauthorized 的定义

- **write（改了任何东西），且 plan 里没申明、SOP 也没 cover = unauthorized**。
- **只读命令如 `show` 永远没事**，因为没改任何东西。
- 漏掉一个该报的问题 = false negative（漏报），比 false positive 更严重。
- 没问题却报了 = false positive（误报），现在的主要问题。

## 4. 模型和调用方式

- 公司提供模型：通过公司 LiteLLM API 调用，背后是 **GLM 5.3 Flash**。
- 走 Chat Completions 接口，传 `tools` 就能做 function calling；模型自己决定调不调、调哪个、传什么参数。
- **不要一次传所有工具**，只传当前步骤需要的工具。当前工具只有 3 个，全传也可以。
- 模型返回 `finish_reason: "tool_calls"` 时，Agent 代码执行真实函数，再把结果以 `role: "tool"` 的消息喂回模型，继续循环，直到返回 `finish_reason: "stop"` 的纯文本结论。
- 工具定义用 JSON Schema，描述要写清楚，模型靠描述决定调哪个。

## 5. 架构：数据在 DB，循环用 LangGraph，工具用 SQLAlchemy ORM

- **数据在数据库里，Agent 循环用 LangGraph，工具是 SQLAlchemy ORM 函数去 DB 取数据**。
- 工具只读数据库，LLM 只给参数（如 cr_number、关键词），**绝不让 LLM 写自由 SQL**。
- 工具返回 dict / JSON，不要返回原始 ORM 对象。
- 三个工具：取 plan、取 logs、查 SOP。

## 6. LangGraph 图

节点：取 plan -> 拆 plan -> 过滤 logs -> 查 SOP -> 逐条判断 -> 复核 -> 输出

- 第 1 层用代码批量处理：过滤时间窗口、去重、划掉只读命令、原文对上的直接通过，大部分日志在这里解决。
- 第 2 层模型只判断剩下的未匹配 write 命令，分批处理（例如每次约 10 条），带着 plan 清单 + SOP 结果。
- 第 3 层按设备 + 时间 + 会话分组，一个配置会话不要算成多个 unauthorized。
- 查到 SOP 的走输出分支，没查到的走判断分支（conditional edge）。
- 扩展时只加节点 + 一两条边，例如判断后加人工复核节点，输出后加通知节点，其他节点不动。

## 7. 规则分层（不要全塞进一个 prompt）

1. **代码**：只读过滤、时间窗口、去重、原文精确匹配。
2. **系统提示词**：核心判断标准，短而稳定：写命令必须有出处（plan 或 SOP）；每判一条 unauthorized 必须引用具体 log 记录和查不到 SOP 的依据，否则降级为 needs-review。
3. **配置文件 / skill 文件**：可变规则，例如各厂商 write 命令动词清单、SOP 索引、团队约定。
4. **当次任务数据**：这个 CR 的 plan、logs、相关 SOP 摘要。

One shot 失败的原因就是这四层全混在一个 prompt 里，上下文一大模型就丢规则。

## 8. 输出格式（固定 JSON，不要只返回一段话）

```json
{
  "verdict": "unauthorized",
  "unauthorized_commands": [
    {"command": "...", "log_id": "...", "device": "...", "reason": "..."}
  ],
  "sop_covered": [
    {"command": "...", "sop_id": "SOP123"}
  ],
  "needs_review": []
}
```

- verdict 取值：authorized / unauthorized / needs-review。
- 复核节点：每条 unauthorized 必须能引用真实 log 记录和一次失败的 SOP 查询，否则降级为 needs-review。这是降低 false positive 最直接的一步。

## 9. 评估方法

用人工标注过的历史 CR 当测试集，填混淆矩阵：

- TP：真有问题，报出来了；FP：没问题，误报了；FN：真有问题，漏掉了；TN：没问题，放行了。
- **Recall = TP / (TP + FN)，第一优先级**：真问题抓到多少。
- **Precision = TP / (TP + FP)**：报出来的有多少是真的，低就是现在的 false positive 问题。
- **F1 = 2 x Precision x Recall / (Precision + Recall)**：单个对比数字。
- **Accuracy = (TP + TN) / 总数，不能当主指标**：100 个 CR 里只有 5 个真问题，什么都不报也能得 95% 准确率，Recall 是 0。
- 每次改规则 / prompt / 流程后都要重跑，先确认 Recall 没掉，再庆祝 Precision 上升。

## 10. 约束条件

- 同一份代码要在本地和 Linux server 上都能跑，环境相关的值（LiteLLM 地址、密钥、Graylog / ITSM 地址、DB 地址）全部放环境变量，**不要硬编码**。
- 安装只能用 pip / npm，**不能自己去网站下载东西**；公司电脑不能上外网，公司 npm 可以下载部分 JS 包。
- 优先 pip 安装的包。Playwright 浏览器二进制、Docker 镜像这类要联网下载的不要用。
- 已安装：LangGraph、LiteLLM，以及 SQLAlchemy + DB 驱动、FastAPI + Uvicorn、Pydantic、python-dotenv、pytest。
- DB 用 Postgres 还是 MySQL 待确认，驱动对应是 psycopg2-binary 或 pymysql。

## 11. MCP 和 A2A（概念，不用在 v1 里引入）

- **MCP**：Agent 调工具的协议，工具是确定性的单步操作，例如查 SOP、取 logs。
- **A2A**：Agent 调 Agent 的协议，一个 Agent 把任务交给另一个专家 Agent。
- 工具是单一技能，Agent 是把技能、知识、规则包在一起的专家，能独立完成一整类任务。
- v1 不需要 A2A，先把单 Agent + 直接工具做稳。

## 12. 给 CC 的改造步骤

1. 先读现有的 one shot 代码，总结输入、输出格式和 prompt 拼法，给出改造方案，等确认后再动手。
2. 按第 6 节搭 LangGraph 图，一次只改一个节点，改完跑通再改下一个。
3. 把数据从 prompt 里移出来，放到三个 SQLAlchemy ORM 工具里；把规则按第 7 节分层。
4. 输出改成第 8 节的固定 JSON，加上复核节点。
5. 用历史 CR 跑评估，按第 9 节算 Recall / Precision / F1，对比 one shot，确认变好再切换。
