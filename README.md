# Embodied Agent Runtime
> Project Profile: **Pure Simulation, Engineering Research Open-Source Project**
> Core Goal: Build a VLM + Function Calling based embodied agent runtime from scratch. Research how general large language models can reliably control external simulated environments with physical states, execution constraints and stochastic uncertainties.
> Key Distinction: **We do not train low-level robot models or directly output joint angles. Instead, we enable LLMs to invoke abstract robot skill interfaces to complete physical tasks.**

## 📖 Project Overview
Embodied Agent Runtime is a modular, extensible runtime framework for embodied AI agents.
This project is not focused on robot motion algorithms or LLM pre-training. Its core is **Agent Engineering** practiced in embodied scenarios:
Decouple VLM visual perception, LLM reasoning & planning, function calling, state management and failure recovery. Natural language instructions drive a simulated robotic arm to perform pick-and-place and other physical manipulation tasks.

We adopt a two-stage simulation roadmap:
1. Phase 1: MuJoCo + Gymnasium Robotics (Fetch robotic arm). Lightweight setup to build the agent system rapidly. Focus on agent loop, tool abstraction and error recovery.
2. Phase 2: Upgrade to Isaac Lab for complex scenes, multi-sensor support, with optional ROS 2 integration.

> Core Architecture Principle: `Agent ≠ Robot Controller`
> Agent handles **decision-making, planning and orchestration**; Robot Runtime handles **physical execution, motion planning and constraint validation**; Function Calling acts as a strongly contracted interface between the two.

## ✅ Project Scope: What We Build / What We Exclude
### Core Research Scope
- VLM visual scene understanding
- Observable ReAct agent loop
- Standardized Function Calling
- Tool registry & dispatching system
- Task planning and explicit state management
- Execution error detection & automatic recovery

### Mature Reused Components (No Custom Implementation)
- Robotic arm motion planning
- Low-level pick/place control logic

### Out-of-Scope for This Project
- End-to-end VLA robot model training
- Reinforcement learning training
- World model
- Multi-agent collaboration (deferred)

> Critical Design Constraint: **The VLM can only consume camera images and tool return results. It cannot directly access ground-truth object coordinates from the simulator, to preserve the validity of visual perception experiments.**
> Three isolated data channels: Agent-visible observations / Internal simulator ground-truth states / Evaluator metrics.

## 🏗️ 5-Tier System Architecture
```
User Natural Language Instruction
        ↓
┌─────────────────────────────────────┐
│ ① VLM Adapter                      │
│ Image ingestion, scene understanding, semantic object grounding │
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
│ Tool (LLM-callable schema) ←→ Skill (underlying capability) │
│ Tool registration, param validation, async task management │
└───────────────┬─────────────────────┘
                ↓
┌─────────────────────────────────────┐
│ ④ Robot Runtime                    │
│ Motion planning, gripper control, collision checking, execution state │
└───────────────┬─────────────────────┘
                ↓
┌─────────────────────────────────────┐
│ ⑤ Simulation Environment           │
│ MuJoCo physics engine, robot arm, objects, RGB camera │
└─────────────────────────────────────┘
        ↓ Observation Feedback
New visual observations flow back to VLM to close the loop
```

### Module Responsibilities
1. **VLM Adapter**: Convert RGB camera images into semantically described scene objects and grounded target positions. No direct access to simulator ground truth.
2. **Agent Runtime**: Project core. Implements ReAct loop, task decomposition, tool routing, task state tracking and failure recovery policies.
3. **Tool & Skill Layer**:
   - Tool: High-level semantic interface definition exposed to LLM (JSON Schema)
   - Skill: Encapsulated robot capability behind a tool. One skill may compose multiple low-level actions.
4. **Robot Runtime**: LLM-free layer. Translate semantic skills into Cartesian-space motion and gripper commands, enforcing low-level motion constraints.
5. **Simulation Env**: Handles physics simulation, collision, rendering and camera image generation only; no awareness of agent logic.

## 🛠️ Function Calling Interface Design
Unified tool invocation protocol: `ToolCall <--> ToolResult`, supporting async execution, status polling, timeout and cancellation.
Tool status enum: `ACCEPTED / RUNNING / SUCCEEDED / FAILED / CANCELLED`

### 3 Tool Groups
#### A. Perception Tools
| Tool | Input | Output |
|------|-------|--------|
| observe_scene | camera_id | RGB image of current scene |
| detect_object | object_name | object_id, visual confidence and spatial description |
| get_object_pose | object_id | visually estimated 3D pose |
| get_robot_state | None | End-effector and gripper state of robot |

#### B. Manipulation Tools
| Tool | Input | Purpose |
|------|-------|---------|
| move_to | position | Move robot end-effector to target pose |
| pick_object | object_id | Grasp target object |
| place_object | object_id, target_id | Place object onto target container/location |
| open_gripper | None | Open gripper |
| close_gripper | None | Close gripper |

#### C. Monitoring Tools
| Tool | Input | Purpose |
|------|-------|---------|
| get_execution_status | task_id | Query async task state |
| verify_grasp | object_id | Validate grasp success |
| verify_placement | object_id, target_id | Validate placement completion |
| cancel_execution | task_id | Abort ongoing task |

> Design Principle: **Expose high-level semantic tools only. Do not expose raw joint control APIs to the LLM.** Offload motion planning to Robot Runtime to save LLM inference budget.

## 🔁 Agent ReAct Execution Loop
```
User natural language instruction
    ↓
Build task context
    ↓
Observe environment (capture camera image)
    ↓
VLM + LLM reasoning
    ↓
Tool call required?
├─ YES → Validate tool args → Execute tool → Receive ToolResult → Update task state → Check termination → Return to environment observation
└─ NO → Output final task conclusion
```

> Critical Rules:
> - **Task success is NOT judged by LLM text self-report. An independent Evaluator checks simulator physical states for objective validation.**
> - Maintain explicit `task_state` to track subtasks, active objects, historical successes and failures.
> - Hierarchical failure handling: Transient execution failures get limited retries at Skill layer; significant environment changes trigger re-observation; consecutive failures trigger global replanning; safety violations trigger immediate halt.

## 📁 Project Directory Layout
```
embodied-agent-runtime/
├── configs/                    # Configuration files
│   ├── agent.yaml
│   ├── simulator.yaml
│   └── tools.yaml
├── simulator/                  # Simulation environment wrapper
│   ├── environment.py
│   ├── robot.py
│   ├── camera.py
│   ├── objects.py
│   └── state.py
├── perception/                 # VLM perception stack
│   ├── vlm_adapter.py
│   ├── scene_observer.py
│   └── grounding.py
├── agent/                      # Core Agent Runtime
│   ├── controller.py
│   ├── loop.py
│   ├── planner.py
│   ├── state_manager.py
│   └── recovery.py
├── tools/                      # Tool definition, registry & dispatcher
│   ├── registry.py
│   ├── schemas.py
│   ├── dispatcher.py
│   └── robot_tools.py
├── skills/                     # Robot skill implementations
│   ├── base.py
│   ├── pick.py
│   ├── place.py
│   └── move.py
├── runtime/                    # Low-level Robot Runtime execution
│   ├── motion_planner.py
│   ├── execution_manager.py
│   └── safety.py
├── evaluation/                 # Evaluation, metrics & benchmark
│   ├── task_evaluator.py
│   ├── metrics.py
│   └── scenarios.py
├── tests/                      # Unit & integration tests
├── examples/                   # Example scripts
├── main.py                     # Project entrypoint
└── README.md
```
> Decoupling Goal: When switching simulation backend from MuJoCo to Isaac Lab, upper-level Agent code requires minimal modification.

## 🚩 Milestone Roadmap
### M1｜Simulation Environment Setup (Infrastructure)
Goal: No LLM integration; control robot arm purely from Python
Acceptance: Script can drive Fetch arm through a full pick-and-place cycle, read RGB images and robot states.

### M2｜Robot Skill Runtime (Skill System)
Goal: Wrap low-level controls into stable semantic skills
Acceptance: Calling `pick_object()` / `place_object()` directly completes tasks, with execution state tracking and basic failure feedback.

### M3｜Function Calling Integration (Model Integration)
Goal: VLM autonomously selects and invokes robot tools
Acceptance: Given natural language instruction, model can invoke multiple perception/manipulation tools iteratively.

### M4｜Agent Loop Implementation (Core Research)
Goal: Long-horizon multi-subtask execution with explicit state management
Acceptance: Successfully runs compound tasks with ≥3 sequential skills while preserving task context and state.

### M5｜Failure & Exception Recovery (Reliability)
Goal: Handle diverse execution failure cases
Acceptance: Detect failed grasp, object drift, lost detection, motion timeout; select retry / re-observe / replan / abort accordingly.

### M6｜System Evaluation & Ablation Studies (Research Output)
Goal: Build reproducible benchmark for agent runtime
Acceptance: Collect full metrics including success rate, recovery rate and token consumption; support multiple ablation experiments.

## 📊 Evaluation Metrics & Ablation Experiments
### Core Metrics
- Task Success Rate
- Tool Selection Accuracy
- Argument Parameter Accuracy
- Recovery Success Rate
- Average Tool Calls per Task
- Task Completion Time
- Token Consumption per Task

### Ablation Test Suite (Core Research)
- A: Single-shot Function Calling only
- B: Basic ReAct loop
- C: ReAct + explicit task state management
- D: ReAct + state + failure recovery
- E: D + environment observation feedback loop

> Controlled comparisons quantify the impact of each module on task performance and token cost, producing actionable engineering insights.

## 💻 Development Environment & Dependencies
Recommended Python `3.11 / 3.12`. Use `uv` for dependency locking, Git for version control and pytest for unit tests.
Base dependencies:
```bash
pip install mujoco gymnasium gymnasium-robotics numpy opencv-python pydantic pytest python-dotenv
```
> Start with VLM API for perception. Run MuJoCo on Windows at early stage; migrate to Ubuntu 24.04 only if ROS 2 + MoveIt 2 is required later.

## 🔍 Core Research Questions (Technical Report / Thesis Track)
1. **Tool Abstraction**: What abstraction granularity is optimal for robot skill tools?
2. **Agent Loop**: When to continue execution and when to trigger full replanning?
3. **Context Management**: How to bound visual and tool history to avoid context window inflation?
4. **State Consistency**: How to resolve mismatches between LLM’s mental task state and the true physical simulator state?
5. **Error Recovery**: Which failures should be retried inside Skills, and which require Agent-level replanning?

## 🚀 Week 1 Development Checklist
> Prioritize M1; do NOT implement agent logic prematurely
```
[ ] Create GitHub repository embodied-agent-runtime
[ ] Install Python 3.12 + uv
[ ] Install MuJoCo
[ ] Install Gymnasium Robotics
[ ] Launch FetchPickAndPlace simulation environment
[ ] Capture RGB camera image from simulator
[ ] Read robot joint and gripper states
[ ] Write Python script to fully automate one pick & place cycle
```

## 📄 License
> To be filled, MIT license recommended

## 🤝 Contribute
Iterative development workflow. Preserve runnable snapshots, unit tests and Architecture Decision Records (ADR) for every milestone.
Run pytest before submitting code; add corresponding test cases for new skills/tools.
