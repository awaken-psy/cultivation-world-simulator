# 学习计划表

> 项目：cultivation-world-simulator
> 目标：理解 Memory + Planning + 规则引擎结合方式，具备扩展能力

---

## 第一周：读懂架构

| # | 任务 | 文件 | 完成 |
|---|---|---|---|
| 1 | 读启动入口，搞清前后端拉起流程 | `src/server/main.py` | [ ] |
| 2 | 读模拟器主循环，理解 `step()` 编排 | `src/sim/simulator.py` | [ ] |
| 3 | 读修士数据模型（记忆、目标、属性） | `src/classes/core/avatar.py` | [ ] |
| 4 | 读事件/小故事生成机制设计文档 | `docs/specs/story-event-system.md` | [ ] |
| 5 | 读全项目规则索引 | `AGENTS.md` | [ ] |

---

## 第二周：追踪一次完整决策

| # | 任务 | 文件 | 完成 |
|---|---|---|---|
| 6 | 找到决策阶段，读感知→决策→执行流程 | `src/sim/simulator_engine/phases/` | [ ] |
| 7 | 读 LLM prompt 构建方式 | `src/utils/llm/client.py` | [ ] |
| 8 | 读一个简单动作（如移动） | `src/classes/action/` | [ ] |
| 9 | 读一个复杂动作（如战斗），对比规则与 LLM 分工 | `src/classes/mutual_action/` | [ ] |
| 10 | 跑一次完整测试，确认环境正常 | `pytest -n 8` | [ ] |

---

## 第三周：对比与扩展

| # | 任务 | 说明 | 完成 |
|---|---|---|---|
| 11 | 找到 Memory 检索打分机制，与 generative_agents 原版对比 | 重点看重要性/时效/相关性三维 | [ ] |
| 12 | 找到 Reflection（反思）触发时机和实现 | 周期性 LLM 总结记忆 | [ ] |
| 13 | 读配置系统架构 | `docs/specs/config-architecture.md` | [ ] |
| 14 | 尝试添加一个最简单的新动作 | 参考 `AGENTS.md` 3.1 节动作开发规则 | [ ] |

---

## 第四周：外接 Agent 实验

| # | 任务 | 说明 | 完成 |
|---|---|---|---|
| 15 | 读外接控制 API 设计文档 | `docs/specs/external-control-api.md` | [ ] |
| 16 | 用脚本调用 `/api/v1/query/world/state` 拉取世界快照 | Python requests | [ ] |
| 17 | 实现"观察 → 决策 → 干预"闭环脚本 | 基于 `/api/v1/command/*` | [ ] |
| 18 | 整理与 generative_agents 原版的完整差异对比笔记 | 输出到 `learn/notes/` | [ ] |
