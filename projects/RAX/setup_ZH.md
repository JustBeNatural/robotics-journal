# RAX 仿真实验操作指南

> 环境：实验室 Ubuntu + RTX 4090；RAX `robodojo-behavior/container`；Isaac Sim / RoboDojo；Codex + OpenAI API。  
> **首次安装只做一次；每个新终端初始化路径；每个新实验创建独立配置。** 日常运行按第 1–6 步操作，环境与安装历史见附录。

**已有记录状态：** 环境检查已通过，`stack_blocks` 实验曾成功启动；正式评估是否完成、最终成功率及当前 API 请求是否成功仍待日志确认。本次仅整理文档，没有重新运行实验。

这里的命令在实验室工作站上实际运行 RAX 的源码目录（`$RAX_HOME`）中执行。本 journal 的 `codes/rax-robodojo-behavior-container/` 保存代码资料；不要将 journal 根目录当成已经安装仿真环境的运行目录。

## 开始之前：可选使用 tmux

长时间运行时，**先进入 tmux，再执行第 1–4 步**，避免切换 Shell 后丢失环境与配置变量：

```bash
tmux new -s rax-sol
```

ctrl+b再点d就能退出tmux

## 1. 初始化终端（每次新终端）

```bash
export RAX_DISK="$(findmnt -rn -S UUID=f583df79-1c0f-4507-85f1-d50f4625976f -o TARGET)"
test -n "$RAX_DISK" && mountpoint -q "$RAX_DISK" || { echo 'SSD 未挂载'; exit 1; }
export RAX_HOME="$RAX_DISK/rax-project/rax"
export ROBODOJO_RUNTIME_ROOT="$RAX_DISK/rax-runtime/runtime"
export PATH="$HOME/.local/bin:$RAX_DISK/rax-runtime/tools/bin:$PATH"
cd "$RAX_HOME" || exit 1
```

**用途：** 找到外接 SSD、RAX 源码和仿真环境，修复新终端路径变量丢失问题。

## 2. 确认运行条件

```bash
nvidia-smi                       # 查看 RTX 4090 与显存
docker ps                       # 检查 Docker 是否可访问
codex --version                 # 检查 Codex CLI
bash scripts/robodojo_doctor.sh # 可选：综合环境检查
```

如果 `docker ps` 报权限错误，且管理员已将当前用户加入 `docker` 组，先执行 `newgrp docker`，在新 Shell 中重新执行第 1 步，再检查 `docker ps`。不要修改 Docker socket 的权限。

## 3. 配置 OpenAI API（首次或更换模型时）

查看已有模型配置：

```bash
test -f "$HOME/.codex/rax-openai/openai-sol.config.toml" \
  && echo '模型 Profile 文件存在'
# 仅显示模型和接口相关字段，不输出整份本地配置
rg -n '^\s*(model|model_provider|base_url|wire_api)\s*=' \
  "$HOME/.codex/rax-openai/openai-sol.config.toml"
test -s "$HOME/.codex/rax-openai/secrets/openai_api_key" && echo 'API key file OK'
```

当前配置目标：`model = "gpt-6.1-sol"`、`base_url = "https://api.openai.com/v1"`、`wire_api = "responses"`。API Key 放在私有文件，不要显示、提交或分享。**通过 OpenAI API 运行会单独计费。** 模型是否可用以实际 API 响应为准。

## 4. 创建新实验（每个新 Session 一次）

```bash
CONFIG="$(runtime/envs/robodojo/bin/python scripts/configure_auto_research.py \
  --task stack_blocks \
  --agent-cli codex \
  --agent-provider custom \
  --agent-effort xhigh \
  --sim-gpu 0 \
  --episodes 2 \
  --exploration-envs 1 \
  --subagents 0 \
  --formal-episodes 2 \
  --formal-workers 1 \
  --agent-formal off \
  --external-perception off)" || { echo '配置生成失败，停止启动'; exit 1; }

test -f "$CONFIG" || { echo "配置创建失败：$CONFIG"; exit 1; }
export CONFIG
echo "CONFIG=$CONFIG"
```

**用途：** 设置 `stack_blocks` 任务、探索回合、并行环境及正式评估，生成新 Session 的 `operator-input.json`；本步骤尚未调用模型。

| 参数 | 含义 |
|---|---|
| `--task stack_blocks` | 任务：积木堆叠 |
| `--episodes 2` | 探索预算（包含可能的程序排练） |
| `--exploration-envs 1` | 同时运行 1 个仿真探索环境 |
| `--subagents 0` | 关闭额外 Agent |
| `--formal-episodes 2` | 2 个正式测试回合 |
| `--agent-formal off` | 正式评估执行生成的程序，而非 Agent 实时控制 |
| `--agent-provider custom` | 具体 API/模型由主机 provider profile 决定（见第 3 步及附录 D） |

## 5. 启动实验

在完成第 1–4 步的同一个 Shell 中运行。若使用 tmux，应按文首说明先进入 tmux，再执行这些步骤；不要在生成配置后切换到一个未设置 `CONFIG` 的 Shell。

启动前确认配置有效：

```bash
test -n "$CONFIG" && test -f "$CONFIG" \
  || { echo 'CONFIG 未设置或文件不存在，请先创建或找回实验配置'; exit 1; }
```

运行：

```bash
bash scripts/start_auto_research_agent.sh \
  --config "$CONFIG" \
  --provider-home "$HOME/.codex/rax-openai" \
  --profile openai-sol \
  --non-interactive
```

**用途：** 启动 Codex Agent、RoboDojo 仿真、在线探索、控制程序生成及后续评估。检查日志中是否使用目标模型且 API 请求成功。**本步骤开始产生 API 费用。**

`tmux` 分离：按 **Ctrl+B**，松开后按 **D**；重新进入：`tmux attach -t rax-sol`。

### 中断后继续（不要重新创建同一个实验）

先确认第 1 步的环境已初始化，且 `CONFIG` 指向原 Session 的配置文件、原 Launcher 已退出。查看状态与找回配置的方法见第 6 步。

```bash
bash scripts/start_auto_research_agent.sh \
  --config "$CONFIG" \
  --provider-home "$HOME/.codex/rax-openai" \
  --profile openai-sol \
  --non-interactive \
  --resume
```

**用途：** 继续已启动但已中断的 Session。**如果原实例仍在运行，不要重复启动。**

### 正常停止指定 Session

```bash
bash scripts/stop_auto_research_agent.sh --config "$CONFIG"
```

等待 Launcher 清理完成后再恢复。不要通过关闭整个 Docker 服务来停止单个实验。

## 6. 查找 Session、监控日志、查看结果

```bash
ls -td runtime/auto-research/*/  # 列出全部 Session
test -n "$CONFIG" && test -f "$CONFIG" \
  || { echo '请先设置正确的 CONFIG'; exit 1; }
SESSION="$(dirname "$CONFIG")"

if [ -f "$SESSION/agent.log" ]; then
  tail -n 80 "$SESSION/agent.log"
else
  echo 'agent.log 尚未生成'
fi

for file in agent-lifecycle.json formal.json formal-scores.json; do
  if [ -f "$SESSION/$file" ]; then
    echo "查看 $file"
    python3 -m json.tool "$SESSION/$file"
  else
    echo "尚未生成：$file"
  fi
done
```

`formal-scores.json` 尚未生成时，结合 `formal.json`、生命周期和 Agent 日志判断是仍在运行、探索未完成还是发生错误。启动成功不等于正式评估成功。

查找仿真视频：

```bash
if [ -d "$SESSION/native/results" ]; then
  find "$SESSION/native/results" -type f -name '*.mp4'
else
  echo '视频目录尚未生成'
fi
```

在 Ubuntu 本机打开可视化控制台：

```bash
bash scripts/run_operator_dashboard.sh
```

**用途：** 在 Firefox/Chrome 中打开脚本输出的完整本地网址，查看 Session、相机画面、轨迹视频、日志与评估结果。访问 token 不要公开。

> 若新终端的 `CONFIG` 已丢失，请先用 `ls -td runtime/auto-research/*/` 找到**正确的** Session，然后设置：  
> `export CONFIG="$RAX_HOME/runtime/auto-research/<真实session-id>/operator-input.json"`。

## 7. RAX 核心 Methodology 与源码入口

**任务流程：** Task → Agent 阅读观测 → 工具控制机器人 → 反馈与经验记录 → 编写控制代码 → `register` → `rehearse` → `submit` → Formal Evaluation。

| 文件/目录 | 用途 |
|---|---|
| `$SESSION/operator-input.json` | 保存本次实验参数 |
| `$SESSION/agent-workspace/TASK.md` | Agent 实际收到的任务要求 |
| [Agent 规则](codes/rax-robodojo-behavior-container/auto_research_agent/AGENTS.md) | Agent 的探索与程序生成规则 |
| [技能目录](codes/rax-robodojo-behavior-container/agent_skills/) | 机器人操作、感知、经验记忆等技能 |
| [配置脚本](codes/rax-robodojo-behavior-container/scripts/configure_auto_research.py) | 新实验参数与 Session 创建 |
| [Controller](codes/rax-robodojo-behavior-container/services/controller/) | 实验调度、预算和评估控制 |
| `$SESSION/agent-workspace/code/` | Agent 生成的控制程序（如存在） |
| `$SESSION/mcp.jsonl` | 实际工具调用记录（如存在） |
| `$SESSION/formal-scores.json` | 正式评估结果（评估完成后） |

查看任务和技能：

```bash
cat "$SESSION/agent-workspace/TASK.md"
find agent_skills -name SKILL.md | sort
```

**科研建议：** 保持任务及正式评估条件一致，分别改变模型、探索预算或 Agent prompts/Skills；每组实验新建 Session，不要直接修改已启动 Session 的配置。

## 8. 常见问题速查

以下保留操作指南中的排查建议，实际原因应结合对应 Session 日志确认。

| 报错 | 原因与处理 |
|---|---|
| `No such file or directory: scripts/...` | 不在 RAX 根目录；先执行第 1 步 |
| `CONFIG=` / `IsADirectoryError: '.'` | 配置变量为空；重新创建配置或指定正确的 `operator-input.json` |
| `permission denied ... docker.sock` | 当前 Shell 缺少 Docker 组权限；`newgrp docker` 后重设变量 |
| `This session was already launched` | 已启动过；停止原实例后使用 `--resume` |
| `You've hit your usage limit` | 模型额度耗尽；检查 API/订阅计费状态，避免自动无效重试 |
| `Another launcher is running this session` | 原 Launcher 仍存活；停止并等待清理完成后再恢复。 |
| 没有 `formal-scores.json` | 可能尚未完成正式评估；检查 `formal.json` 和 Agent 日志 |


---

## 附录 A：已记录的实验环境

### A.1 硬件与操作系统

| 项目 | 本机配置 | 信息来源或说明 |
|---|---|---|
| 主机名 | `LMBS1263PC05` | 终端提示符 |
| 操作系统 | Ubuntu 22.04.5 LTS（`jammy`） | `lsb_release -a` |
| Linux 内核 | `6.8.0-138-generic` | 配置时的 `uname -r` |
| CPU | Intel Core i9-13900K，32 逻辑处理器 | `lscpu` |
| 内存 | 62 GiB | `free -h` |
| GPU | NVIDIA GeForce RTX 4090，24,564 MiB 显存 | `nvidia-smi` |
| NVIDIA 驱动 | `580.65.06` | 驱动升级后的 `nvidia-smi` |
| 驱动报告的 CUDA 兼容版本 | `13.0` | `nvidia-smi` 的 CUDA Version，**不等于 PyTorch 内置 CUDA 版本** |
| PyTorch GPU 运行时 | PyTorch `2.7.0+cu128`，CUDA Runtime `12.8` | CUDA Tensor/矩阵乘法测试成功 |
| GPU 桌面显示 | RTX 4090 同时承担 Ubuntu 桌面显示 | 会占用部分可供仿真使用的显存 |

### A.2 RoboDojo Python 仿真环境

| 组件 | 已记录版本或状态 |
|---|---|
| Isaac Sim | `5.1.0.0` |
| Isaac Lab | `0.54.3` |
| 仿真 Python | `3.11` |
| NumPy | `1.26.0` |
| cuRobo | 已安装，`nvcc` 检查通过 |
| Git LFS | 已安装 |
| RoboDojo 源码 Commit | `ee67a1468510da7624a089164402359f2afc72c8` |
| RoboDojo Assets Commit | `91f76c28d93dd20c5fa46ce6a5a1d96a4f384acd` |
| RAX 仓库 Commit | **待记录：运行 `git rev-parse HEAD`** |

此前的依赖检查曾发现：`isaaclab 0.54.3` 要求 `starlette==0.49.1`，但环境中安装的是 `0.45.3`；`httpx2` 也存在 `idna` 依赖版本不匹配。尽管如此，RAX doctor 检查已经通过。调试时应保留这些警告记录，**不要为了清除警告而随意升级已固定版本的仿真依赖**。

### A.3 Docker 与 Agent 环境

| 组件 | 已记录版本或状态 |
|---|---|
| Docker Engine | `29.9.3` |
| containerd | 已安装 `2.4.2` |
| Agent Docker 镜像 | `robodojo-auto-research:dev`，构建成功 |
| Docker 权限 | 当前 Shell 启用 `docker` 用户组后，`docker ps` 成功 |
| Codex CLI | Agent 运行日志中观察到 `0.167.0`；主机上的 CLI 版本可能不同 |
| 模型提供商 | 自定义 OpenAI Responses API Profile |
| 配置模型 | `gpt-6.1-sol` |
| 推理强度 | `xhigh` |

验证结果：

```bash
bash scripts/robodojo_doctor.sh
```

返回 `All checks passed`。这意味着主要依赖与运行条件已就绪，**不能据此推断机器人任务或正式评估一定成功**。

## 附录 B：SSD 与运行目录

本次 RAX 项目放在自己的 **Lenovo PS6 外接 SSD** 上，避免占用实验室其他磁盘。外接硬盘整体容量约 931.5 GiB，包含多个分区；其中存放 RAX 的分区是约 469 GiB 的 ext4 文件系统，在本机曾显示为 `/dev/sdb5`。Linux 设备名可能变化，因此推荐始终通过 UUID 定位。

```text
UUID: f583df79-1c0f-4507-85f1-d50f4625976f
挂载路径: /media/huayi/f583df79-1c0f-4507-85f1-d50f4625976f
```

配置时记录的分区空间：总计 469 GiB，已使用 157 GiB，剩余 289 GiB。**这是历史快照，不是当前实时剩余容量。**

```text
<SSD 挂载目录>/
├── rax-project/
│   └── rax/                            # RAX 主仓库
└── rax-runtime/
    ├── runtime/                        # Python 环境、缓存、实验 Session
    │   ├── envs/robodojo/
    │   ├── robodojo-cache/
    │   └── auto-research/<session-id>/
    ├── BEHAVIOR-1K/                     # 可选 Benchmark 的源码
    ├── docker-data/                     # Docker 镜像和容器数据
    └── containerd-data/                 # containerd 数据
```

RAX 仓库中的重要符号链接：

- `runtime` → `<SSD 挂载目录>/rax-runtime/runtime`
- `BEHAVIOR-1K` → `<SSD 挂载目录>/rax-runtime/BEHAVIOR-1K`
- `RoboDojo/.cache` → `<SSD 挂载目录>/rax-runtime/runtime/robodojo-cache`
- `RoboDojo/Assets` → `<SSD 挂载目录>/rax-runtime/runtime/robodojo-cache/robodojo_assets_repo/Assets`

Docker 数据位置：

- `/etc/docker/daemon.json`：`data-root = <SSD 挂载目录>/rax-runtime/docker-data`
- `/etc/containerd/config.toml`：`root = <SSD 挂载目录>/rax-runtime/containerd-data`
- `docker.service` 与 `containerd.service` 使用 `ExecStartPre=/usr/bin/mountpoint -q <SSD 挂载目录>`，以避免外接盘未挂载时将数据错误写入主机本地目录。

**Docker、Isaac Sim 或 RAX 运行时不要拔掉外接 SSD。**

## 附录 C：首次安装记录

以下沿用已有安装记录，当前环境正常时不必重复执行。不要用重装依赖代替排查具体日志。

### C.1 获取源码并安装仿真依赖、Assets

```bash
# 首次部署时，在 RAX 根目录执行
bash scripts/bootstrap_sources.sh
bash scripts/setup_robodojo_env.sh --accept-nvidia-eula
bash scripts/setup_assets.sh
```

| 命令 | 用途 |
|---|---|
| `bootstrap_sources.sh` | 获取固定版本的 RoboDojo 源码、子模块、补丁及相关基准源码 |
| `setup_robodojo_env.sh` | 配置包含 Isaac Sim、Isaac Lab、cuRobo 等的 Python 仿真环境 |
| `setup_assets.sh` | 下载 RoboDojo 所需机器人/物体 Assets 与任务场景布局 |

### C.2 构建 Agent Docker 镜像并检查环境

```bash
runtime/envs/robodojo/bin/python scripts/build_agent_image.py
bash scripts/robodojo_doctor.sh

docker ps
nvidia-smi
```

| 命令 | 用途 |
|---|---|
| `build_agent_image.py` | 构建 Agent 镜像 `robodojo-auto-research:dev` |
| `robodojo_doctor.sh` | 检查相关源码、资产、仿真依赖、Docker、Codex 和 GPU |
| `docker ps` | 不使用 `sudo` 检查 Docker Daemon 访问权限 |
| `nvidia-smi` | 检查 GPU、驱动、显存占用 |

若 `docker ps` 因用户组权限失败，而当前用户**已经被授权加入 `docker` 组**，可以执行：

```bash
newgrp docker
docker ps
```

注意：Docker 组通常具备接近主机 root 的权限，不能把它当作普通低权限用户组。

## 附录 D：已有模型配置示例

目前使用的配置保存在**主机用户目录**，不属于 RAX Git 仓库：

```text
~/.codex/rax-openai/openai-sol.config.toml
~/.codex/rax-openai/secrets/openai_api_key
```

已使用的模型配置示例（不包含密钥值）：

```toml
model_provider = "openai"

[model_providers.openai]
name = "OpenAI API"
base_url = "https://api.openai.com/v1"
wire_api = "responses"

[robodojo]
model = "gpt-6.1-sol"
key_file = "secrets/openai_api_key"
native_models = true
```

Session 配置中的 `--agent-provider custom` 配合启动参数 `--provider-home`、`--profile` 来选择这份主机侧 API Profile。OpenAI API 按 API 账户独立计费，与 ChatGPT 订阅额度不同。模型名是否可调用还取决于 API 账户权限及实际可用模型。

**不显示 API 密钥内容的文件检查：**

```bash
test -s "$HOME/.codex/rax-openai/secrets/openai_api_key" \
  && echo 'API Key 文件存在且非空'

test -f "$HOME/.codex/rax-openai/openai-sol.config.toml" \
  && echo '模型 Profile 文件存在'
```

不要将 API Key、包含密钥的日志或本地秘密文件提交到 Git。

## 附录 E：小预算实验参数说明

当前尝试运行的是一个**小预算的 RoboDojo 积木堆叠实验**。下表基于创建 Sol Session 时使用过的命令；最终应以**该 Session 的 `operator-input.json`** 为准。

| 实验参数 | 当前值 | 作用 |
|---|---|---|
| Benchmark | `robodojo` | 桌面双臂机器人操作基准 |
| Robot | 双 ARX X5 机械臂 | RoboDojo 中的仿真机器人 |
| Task | `stack_blocks` | 积木堆叠任务 |
| Simulator | Isaac Sim 5.1 | 物理与场景仿真 |
| Agent CLI | `codex` | Coding Agent 执行框架 |
| Provider | `custom` → OpenAI API | 模型请求通道 |
| Model | `gpt-6.1-sol` | 写在主机 API Profile 中，不一定写入 Session JSON |
| Reasoning effort | `xhigh` | Agent 推理强度 |
| Simulation GPU | `0` | RTX 4090 |
| Exploration episodes | `2` | 总探索预算；rehearsal 同样消耗 episode |
| Parallel exploration environments | `1` | 同时运行 1 个仿真探索环境 |
| Subagents | `0` | 不启用额外子 Agent |
| Formal episodes | `2` | 2 个独立测试 episode |
| Formal workers | `1` | 1 个正式评估 Worker |
| Agent formal | `off` | 正式评估执行生成的控制程序，而非实时 Agent |
| External perception | `off` | 不使用额外的 Gemini 感知服务 |

**暂未从 Sol Session 的配置快照独立核实**：观测 Profile、演示上下文、任务规则、随机种子、具体评分阈值和成功判定。应分别查看 `operator-input.json`、`TASK.md` 与 RoboDojo 任务源码，不将默认值当作已确认值。

## 附录 F：原有复现进度（待补充证据）

- [x] 外接 SSD 已挂载，RAX 仓库可访问
- [x] RTX 4090 可识别，PyTorch CUDA 矩阵计算测试通过
- [x] Isaac Sim 5.1 和 RoboDojo Assets 已安装
- [x] Docker/containerd 数据目录位于个人外接 SSD
- [x] Agent Docker 镜像构建成功
- [x] `robodojo_doctor.sh` 返回 `All checks passed`
- [x] Codex Agent 至少成功启动过一个实验 Session
- [ ] 记录当前 RAX 仓库 Commit 和 Sol Session ID
- [ ] 归档当前实验的 `operator-input.json` 与 `TASK.md`
- [ ] 从日志确认 Sol/OpenAI API 请求真正成功
- [ ] 确认控制程序完成注册、排练与正式评估
- [ ] 保存正式测试分数、成功率、视频及 API 消耗

## 附录 G：方法修改的代码入口

| 希望修改的内容 | 源码位置 |
|---|---|
| Agent 的探索策略、代码生成流程 | `auto_research_agent/AGENTS.md`、`auto_research_agent/START_PROMPT.md` |
| 机器人技能和工具使用指导 | `agent_skills/` |
| 工作空间组装和 Task Prompt 生成 | `services/workspace/` |
| Task、预算、随机布局、观测和正式评估参数 | `scripts/configure_auto_research.py`、`services/controller/config.py` |
| RoboDojo 机器人仿真与任务实现 | `RoboDojo/`、`services/robodojo/` |
| MCP 操作及正式评估生命周期 | `services/controller/` |
| 模型 Provider、Relay 和 Codex CLI 调用 | `harness/codex_cli/` |
| 单元与集成测试 | `local_tests/` |

若目标是进行可信的 Methodology 对比，建议尽量固定任务、布局集合、随机种子、探索预算和正式评估协议，每次只修改一项方法，并记录相应 Git Commit。


## 参考与记录依据

- 本次日常流程依据提供的《RAX 仿真实验操作流程（精简版）》整理；本机环境、安装历史及勾选状态沿用修改前的 `setup_ZH.md`，未作为本次重新验证的结果。
- [本地安装说明](codes/rax-robodojo-behavior-container/docs/SETUP.md)
- [本地使用说明](codes/rax-robodojo-behavior-container/docs/USAGE.md)
- [本地 Dashboard 说明](codes/rax-robodojo-behavior-container/services/dashboard/README.md)
- [项目首页](README.md)
