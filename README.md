# Embodied Agent Runtime
> 项目定位：**纯仿真、工程研究型开源项目**
> 核心目标：从零实现一套基于VLM+Function Calling的具身智能Agent运行时，研究通用大模型如何可靠控制带物理不确定性的外部仿真环境。
> 一句话区分：**不训练机器人底层模型，不直接输出关节角；让大模型调用抽象机器人技能接口完成物理任务**。

## 📖 项目简介
Embodied Agent Runtime 是一个模块化、可扩展的具身智能Agent运行时框架。
该项目的核心不是机器人运动算法或大模型预训练，而是**Agent工程（Agent Engineering）**在具身场景的落地实践：
将VLM视觉感知、大模型推理规划、工具调用、状态管理、异常恢复解耦，用自然语言指令驱动仿真机械臂完成抓取、放置等物理操作。

项目采用**两阶段仿真路线**：
1. 第一阶段：MuJoCo + Gymnasium Robotics（Fetch机械臂），轻量快速搭建Agent系统，聚焦Agent循环、工具抽象、错误恢复；
2. 第二阶段：升级至Isaac Lab，支持更复杂场景、多传感器，可对接ROS2。

> 核心架构原则：`Agent ≠ Robot Controller`
> Agent 负责**决策、规划、编排**；Robot Runtime 负责**物理执行、运动规划、约束校验**；Function Calling 是两者之间强契约式接口。

## ✅ 项目边界：做什么 / 不做什么
### 核心研究内容
- VLM 视觉场景理解
- 可观测的 Agent ReAct 循环
- 标准化 Function Calling 工具调用
- 工具注册与调度系统 Tool Registry
- 任务规划与显式状态管理
- 执行错误检测与自动恢复

### 复用成熟组件（不重新造轮子）
- 机械臂运动规划
- 抓取放置底层控制逻辑

### 不在本项目内实现
- VLA / 机器人端到端模型训练
- 强化学习训练
- World Model 世界模型
- 多智能体 Multi-Agent 协作（暂不纳入）

> 关键设计约束：**VLM只能读取相机图像与工具返回结果，不能直接读取仿真引擎Ground-truth物体坐标**，保证视觉感知实验有效性。
> 三个数据通道隔离：Agent可见观测数据 / 仿真引擎内部真实状态 / Evaluator评估指标。

## 🏗️ 系统五层架构
```
User Natural Language Instruction
        ↓
┌─────────────────────────────────────┐
│ ① VLM Adapter                      │
│ 图像输入、场景理解、物体语义定位      │
└───────────────┬─────────────────────┘
                ↓
┌─────────────────────────────────────┐
│ ② Agent Runtime (Core)             │
│ Agent Controller / Planner / Tool Router │
│ State Manager / Recovery Manager    │
└───────────────┬─────────────────────┘
                ↓
┌─────────────────────────────────────┐
│ ③ Tool & Skill Layer               │
│ Tool（模型调用接口） ←→ Skill（执行能力封装）│
│ 工具注册、参数校验、异步任务管理      │
└───────────────┬─────────────────────┘
                ↓
┌─────────────────────────────────────┐
│ ④ Robot Runtime                    │
│ 运动规划、夹爪控制、碰撞检测、执行状态 │
└───────────────┬─────────────────────┘
                ↓
┌─────────────────────────────────────┐
│ ⑤ Simulation Environment           │
│ MuJoCo物理引擎、机械臂、物体、RGB相机 │
└─────────────────────────────────────┘
        ↓ Observation Feedback
新图像观测回流至VLM，形成闭环
```

### 模块职责简述
1. **VLM Adapter**：把相机RGB图像转为模型可读的场景物体描述、目标定位信息；不读取仿真内部真值。
2. **Agent Runtime**：项目核心，实现ReAct循环、任务分解、工具路由、任务状态维护、失败恢复策略。
3. **Tool & Skill Layer**：
   - Tool：给大模型调用的**语义化接口定义**（JSON Schema）
   - Skill：Tool背后封装的机器人动作能力，一个Skill可组合多个底层动作
4. **Robot Runtime**：无LLM依赖层，将语义技能转换为笛卡尔空间运动、夹爪控制，处理底层运动约束。
5. **Simulation Env**：仅负责物理仿真、碰撞、渲染相机图像，不感知Agent逻辑。

## 🛠️ 工具接口设计（Function Calling）
统一工具调用协议：`ToolCall <--> ToolResult`，支持异步执行、状态轮询、超时/取消。
工具状态枚举：`ACCEPTED / RUNNING / SUCCEEDED / FAILED / CANCELLED`

### 三类工具集
#### A. Perception Tools 感知工具
| 工具 | 输入 | 输出 |
|------|------|------|
| observe_scene | camera_id | 当前场景RGB图像 |
| detect_object | object_name | object_id、视觉置信度与位置描述 |
| get_object_pose | object_id | 视觉估计3D坐标 |
| get_robot_state | - | 机械臂末端、夹爪状态 |

#### B. Manipulation Tools 操作工具
| 工具 | 输入 | 作用 |
|------|------|------|
| move_to | position | 移动机械臂末端执行器 |
| pick_object | object_id | 抓取目标物体 |
| place_object | object_id, target_id | 将物体放置到目标容器/位置 |
| open_gripper | - | 打开夹爪 |
| close_gripper | - | 闭合夹爪 |

#### C. Monitoring Tools 执行监控工具
| 工具 | 输入 | 作用 |
|------|------|------|
| get_execution_status | task_id | 查询异步任务执行状态 |
| verify_grasp | object_id | 校验抓取是否成功 |
| verify_placement | object_id, target_id | 校验放置是否到位 |
| cancel_execution | task_id | 中断当前任务 |

> 设计原则：**优先暴露高层语义工具，不向大模型暴露底层关节控制API**。将运动规划交给Robot Runtime，不占用大模型推理资源。

## 🔁 Agent ReAct 执行循环
```
用户自然语言指令
    ↓
构建任务上下文
    ↓
观测环境（获取相机图像）
    ↓
VLM+LLM推理思考
    ↓
是否调用工具？
├─ 是 → 校验工具参数 → 执行工具 → 获取ToolResult → 更新任务状态 → 检查终止条件 → 回到环境观测
└─ 否 → 输出最终任务结论
```

> 关键规则：
> - **任务成功判定不由大模型文字自述决定，由独立Evaluator读取仿真物理状态做客观校验**；
> - 显式维护task_state，记录子任务、当前对象、历史失败/成功步骤；
> - 分层失败处理：瞬时执行失败优先由Skill层重试；环境发生显著变化时触发重新观测；连续失败触发全局重规划；安全约束触发则立即停机。

## 📁 项目目录结构
```
embodied-agent-runtime/
├── configs/                    # 配置文件
│   ├── agent.yaml
│   ├── simulator.yaml
│   └── tools.yaml
├── simulator/                  # 仿真环境封装
│   ├── environment.py
│   ├── robot.py
│   ├── camera.py
│   ├── objects.py
│   └── state.py
├── perception/                 # VLM感知层
│   ├── vlm_adapter.py
│   ├── scene_observer.py
│   └── grounding.py
├── agent/                      # Agent Runtime核心
│   ├── controller.py
│   ├── loop.py
│   ├── planner.py
│   ├── state_manager.py
│   └── recovery.py
├── tools/                      # 工具定义、注册、分发
│   ├── registry.py
│   ├── schemas.py
│   ├── dispatcher.py
│   └── robot_tools.py
├── skills/                     # 技能实现层
│   ├── base.py
│   ├── pick.py
│   ├── place.py
│   └── move.py
├── runtime/                    # Robot Runtime底层执行
│   ├── motion_planner.py
│   ├── execution_manager.py
│   └── safety.py
├── evaluation/                 # 评测、指标、benchmark
│   ├── task_evaluator.py
│   ├── metrics.py
│   └── scenarios.py
├── tests/                      # 单元/集成测试
├── examples/                   # 示例脚本
├── main.py                     # 项目入口
└── README.md
```
> 模块解耦目标：后续切换仿真后端（MuJoCo → Isaac Lab）时，Agent上层代码基本无需改动。

## 🚩 分阶段开发里程碑（Milestone）
### M1｜搭建仿真环境（基础设施）
目标：不接入LLM，Python原生控制机械臂
验收：脚本可驱动Fetch机械臂完成一次抓取+放置，读取RGB图像与机器人状态

### M2｜实现Robot Skill Runtime（技能系统）
目标：封装底层控制为稳定语义技能
验收：直接调用`pick_object()`/`place_object()`即可完成任务，内置执行状态与基础失败反馈

### M3｜实现Function Calling（模型接入）
目标：VLM能够自主选择并调用机器人工具
验收：输入自然语言指令，模型自主调用多轮感知/操作工具

### M4｜实现Agent Loop（核心研究）
目标：长程多子任务执行、显式状态管理
验收：连续执行≥3个技能组合任务，稳定维护任务上下文与状态

### M5｜异常恢复（可靠性）
目标：处理各类执行失败场景
验收：识别抓取失败、物体移位、识别丢失、运动超时；按策略选择重试/重观测/重规划/终止

### M6｜系统评测与消融实验（研究成果）
目标：构建可复现Benchmark，量化评估Agent架构
验收：完整采集任务成功率、恢复率、token消耗等指标；支持多组消融对照实验

## 📊 评估指标 & 消融实验
### 核心评测指标
- Task Success Rate 任务最终成功率
- Tool Selection Accuracy 工具选择正确率
- Argument Accuracy 工具参数正确性
- Recovery Success Rate 失败恢复成功率
- Average Tool Calls 单任务平均工具调用次数
- Task Completion Time 任务耗时
- Token Consumption 单任务模型Token开销

### 消融对照实验（研究核心）
- A：仅单次Function Calling
- B：基础ReAct循环
- C：ReAct + 显式任务状态管理
- D：ReAct + 状态 + 错误恢复
- E：D + 环境观测反馈闭环

> 通过对照，量化验证每一个模块对系统性能、token开销的影响，产出可分析的工程结论。

## 💻 开发环境 & 依赖
推荐 Python `3.11 / 3.12`，使用 `uv` 管理依赖锁定版本，Git版本控制，pytest单元测试。
基础依赖：
```bash
pip install mujoco gymnasium gymnasium-robotics numpy opencv-python pydantic pytest python-dotenv
```
> 初期直接使用VLM API接入；Windows优先跑MuJoCo；如需ROS2+MoveIt2，后续迁移至Ubuntu 24.04。

## 🔍 核心研究问题（论文/技术报告主线）
1. **Tool Abstraction**：机器人技能工具抽象粒度应该到哪一层？
2. **Agent Loop**：什么时候继续执行，什么时候触发重新规划？
3. **Context Management**：如何控制感知与工具历史，避免上下文窗口爆炸？
4. **State Consistency**：大模型认知状态与仿真真实物理状态不一致如何处理？
5. **Error Recovery**：哪些失败交给Skill层重试，哪些必须交给Agent重规划？

## 🚀 第一周开发任务清单
> 优先完成M1，不提前写Agent逻辑
```
[ ] 创建GitHub仓库 embodied-agent-runtime
[ ] 安装Python3.12 + uv
[ ] 安装MuJoCo
[ ] 安装Gymnasium Robotics
[ ] 启动 FetchPickAndPlace 仿真环境
[ ] 读取仿真RGB相机图像
[ ] 读取机械臂关节、夹爪状态
[ ] 通过Python脚本，全自动完成一次Pick & Place
```

## 📄 License
> 待填写，推荐 MIT 开源协议

## 🤝 Contribute
项目采用迭代式开发，每个Milestone保留可运行版本、单元测试与架构决策记录（ADR）。
提交代码前运行pytest；新增技能/工具需要补充对应的测试用例。


