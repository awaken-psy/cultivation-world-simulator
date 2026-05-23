# cultivation-world-simulator 学习指南

> 整理日期：2026-05-23
> 学习阶段：Roadmap 阶段 4 — 深入中文项目
> 目标：理解 Memory + Planning + 规则引擎如何结合，以及如何在此基础上扩展

---

## 一、项目定位

这是一个 **AI 驱动的修仙世界模拟器**，核心思路是：

- **规则引擎** 约束世界物理（境界、灵根、功法、宗门、战斗胜率……），防止 LLM 幻觉失控
- **LLM** 负责角色的思考、对话、决策，产生涌现式剧情
- **玩家扮演"天道"**，只观察或微干预，不直接操控角色

与斯坦福 generative_agents 的核心区别：加入了严谨的修仙世界规则体系，LLM 的想象力被限制在合理框架内。

---

## 二、技术栈速览

| 层 | 技术 |
|---|---|
| 后端 | Python 3.10+，FastAPI，WebSocket，asyncio |
| 前端 | Vue 3 + TypeScript + Vite + PixiJS |
| AI | 任意 OpenAI 兼容接口（DeepSeek / MiniMax / Ollama 等） |
| 配置 | OmegaConf（`static/config.yml`） |
| 测试 | pytest（后端），Vitest（前端） |

---

## 三、目录结构导读

```
cultivation-world-simulator/
├── src/
│   ├── server/          # FastAPI 服务层（HTTP + WebSocket）
│   │   ├── main.py      # 启动入口，拉起前后端
│   │   ├── bootstrap.py # 应用初始化
│   │   ├── loop_runtime.py      # 模拟主循环
│   │   ├── command_handlers.py  # 外部命令处理（天道干预）
│   │   └── public_query_builders.py  # 只读查询构建
│   ├── sim/             # 模拟器核心
│   │   ├── simulator.py # 主模拟器，step() 驱动世界推进
│   │   └── simulator_engine/phases/  # 每个 step 的各阶段
│   ├── classes/         # 领域模型
│   │   ├── core/        # Avatar（修士）、World、Sect（宗门）等核心类
│   │   ├── action/      # 单人动作（修炼、移动、炼丹……）
│   │   └── mutual_action/  # 多人动作（战斗、对话、双修……）
│   ├── systems/         # 独立子系统
│   │   ├── battle.py    # 战斗胜率计算
│   │   ├── cultivation.py  # 修炼与突破
│   │   ├── fortune.py   # 奇遇系统
│   │   └── tribulation.py  # 天劫 & 心魔
│   └── utils/
│       └── llm/         # LLM 客户端封装（统一入口 client.py）
├── web/                 # Vue 前端
├── static/
│   ├── config.yml       # 只读版本配置（世界参数、概率等）
│   └── locales/         # i18n 多语言文件
├── docs/specs/          # 各子系统设计文档（含金量极高）
├── tests/               # pytest 测试
└── AGENTS.md            # AI 代理工作说明（规则索引）
```

---

## 四、核心概念映射（对比 generative_agents 论文）

| 论文概念 | 本项目实现位置 | 说明 |
|---|---|---|
| **Memory Stream（记忆流）** | `src/classes/core/avatar.py` 中的短期/长期记忆字段 | 角色经历的事件列表，按重要性存储 |
| **Reflection（反思）** | `src/utils/llm/` + Avatar 决策流程 | LLM 周期性总结记忆，生成高阶判断 |
| **Planning（计划）** | Avatar 的长短期目标系统 | 支持玩家主动设定，LLM 自主规划行动序列 |
| **Perception（感知）** | `src/sim/simulator_engine/phases/` | 每个 step 收集周围环境信息注入 Avatar |
| **Action（行动）** | `src/classes/action/` + `mutual_action/` | 注册式动作框架，规则校验 + LLM 决策 |

---

## 五、启动流程（代码追踪路径）

```
main.py
  └─ bootstrap.py          # 加载 config.yml、初始化 FastAPI app
       └─ loop_runtime.py  # 启动模拟主循环（asyncio）
            └─ simulator.py → step()
                 ├─ phases/perception.py   # 感知阶段
                 ├─ phases/decision.py     # 决策阶段（调用 LLM）
                 ├─ phases/action.py       # 动作执行
                 └─ phases/finalize.py     # 结算（死亡、突破、事件入库）
```

**本地启动**：
```bash
cws   # zsh alias：cd 项目目录 + .venv/bin/python src/server/main.py --dev
# 前端：http://localhost:5173
# 后端：http://127.0.0.1:8002
```

---

## 六、LLM 集成方式

所有 LLM 调用统一走 `src/utils/llm/client.py`，按场景选 `LLMMode`：

- `LLMMode.MAX`：复杂推理（角色年度思考、宗门决策）
- `LLMMode.FLASH`：快速响应（日常动作决策、对话）

调用方不处理重试，底层统一处理；异常捕获 `LLMError / ParseError` 后降级（跳过本次 LLM 决策，走规则兜底）。

**配置模型**：首次启动后在前端设置页配置 API Key 和模型预设，保存到 `~/.local/share/CultivationWorldSimulator-dev/<id>/secrets.json`。

---

## 七、动作系统（重点）

动作是理解这个项目架构的关键。每个动作都是一个独立 Python 类：

```python
# 动作必须实现的字段和方法
class SomeAction(BaseAction):
    ACTION_NAME_ID = "action.some"   # i18n key
    EMOJI = "⚔️"
    PARAMS = [...]                    # 参数声明
    PARAM_OPTION_SOURCES = {...}      # 参数来源（从世界状态动态获取）

    def can_possibly_start(self, avatar, world) -> bool: ...
    async def execute(self, avatar, world) -> ActionResult: ...
```

注册方式：在 `src/classes/action/__init__.py` 导入即自动注册。

**学习建议**：先读一个简单动作（如移动），再读一个复杂动作（如战斗），对比理解规则校验和 LLM 决策的分工。

---

## 八、外接 API（天道干预接口）

可以用外部脚本或 Agent 通过 REST API 观察和干预世界：

```bash
# 查询世界状态
GET  /api/v1/query/world/state
GET  /api/v1/query/events
GET  /api/v1/query/detail?type=avatar&id=<id>

# 干预命令
POST /api/v1/command/game/start
POST /api/v1/command/avatar/*
POST /api/v1/command/world/*
```

详细文档：`docs/specs/external-control-api.md`

---

## 九、推荐学习顺序

### 第一周：读懂架构
- [ ] 读 `src/server/main.py` — 搞清启动流程（约 100 行）
- [ ] 读 `src/sim/simulator.py` — 理解 `step()` 的编排逻辑
- [ ] 读 `src/classes/core/avatar.py` — 理解修士的数据模型（记忆、目标、属性）
- [ ] 读 `docs/specs/story-event-system.md` — 理解事件/小故事生成机制

### 第二周：动手追踪一次完整决策
- [ ] 在 `src/sim/simulator_engine/phases/` 中找到决策阶段
- [ ] 追踪一个 Avatar 从"感知环境"到"选择动作"到"执行结算"的完整流程
- [ ] 在 `src/utils/llm/client.py` 中看 LLM 调用的 prompt 是怎么构建的

### 第三周：对比与扩展
- [ ] 对比 generative_agents 原版：Memory 的检索打分机制在哪里？
- [ ] 找到 Reflection（反思）的触发时机和实现
- [ ] 尝试添加一个最简单的新动作（参考 `AGENTS.md` 第 3.1 节的动作开发规则）

### 第四周：外接 Agent 实验
- [ ] 用 Python 脚本调用 `/api/v1/query/world/state` 拉取世界快照
- [ ] 实现一个简单的"观察 → 决策 → 干预"闭环脚本
- [ ] 参考 `docs/specs/external-control-api.md` 了解稳定接口约定

---

## 十、关键文档索引

| 文档 | 内容 |
|---|---|
| `AGENTS.md` | 全项目规则索引，AI 代理工作说明（必读） |
| `docs/specs/external-control-api.md` | 外接控制 API 设计 |
| `docs/specs/story-event-system.md` | 小故事/事件生成系统 |
| `docs/specs/avatar-roleplay-mode.md` | 角色扮演模式设计 |
| `docs/specs/config-architecture.md` | 配置系统架构 |
| `docs/specs/cultivation-alias-system.md` | 修为阶层别名系统 |
| `static/config.yml` | 世界参数配置（概率、境界定义等） |

---

## 十一、与 generative_agents 原版的核心差异

| 维度 | generative_agents（斯坦福） | cultivation-world-simulator |
|---|---|---|
| 世界规则 | 极简（Smallville 小镇日常） | 复杂修仙规则体系（境界/灵根/功法/宗门） |
| 记忆检索 | 向量数据库 + 三维打分 | 事件列表 + LLM 摘要 |
| 动作系统 | 自然语言描述 | 注册式强类型动作 + 规则校验 |
| 组织层 | 无 | 宗门意志 AI（组织级决策） |
| 干预方式 | 无 | 完整的天道干预 API |
| 部署方式 | 本地 Python | Docker / 桌面 Electron / 源码 |

---

## 十二、常用命令速查

```bash
# 启动
cws

# 后端测试
cd ~/projects/AI/cultivation-world-simulator
pytest
pytest -n 8          # 并行快速回归

# 前端测试
cd web && npm run test
cd web && npm run type-check

# 同步上游
git fetch upstream && git merge upstream/main
```
