# RAX Simulation Setup & Reproduction Notes

> **Project:** RAX: Scaling Online Exploration for Robot Agents  
> **Repository:** https://github.com/robot-agent-exploration/rax  
> **Branch:** `robodojo-behavior/container`  
> **Host:** `LMBS1263PC05` (Ubuntu workstation)  
> **Record date:** 2026-11-09  
> **Scope:** RoboDojo simulation with Codex and OpenAI API; **no SSH configuration**.  
> **Status:** Environment checks passed; `stack_blocks` session has launched. **Formal completion and success rate have not been verified in this note.**

## 1. Experiment overview

This reproduction uses the RAX container branch and its RoboDojo backend. An LLM coding agent explores a robot manipulation task by calling MCP tools, accumulates observations/experience, writes an executable robot controller, rehearses it, and (if qualified and submitted) evaluates it on held-out layouts.

```
Task & session config -> Codex (OpenAI API)
                      -> RoboDojo / Isaac Sim / MCP exploration
                      -> agent memory + controller code
                      -> register -> rehearsal -> submit
                      -> formal evaluation -> scores / videos / logs
```

**This is not conventional policy-weight training.** The agent learns through tool interactions and writes control code. The simulation uses the GPU; model inference is requested from the OpenAI API.

## 2. Host machine and verified software

### 2.1 Hardware and operating system

| Component                          | Observed configuration                     | Evidence / note                                              |
| ---------------------------------- | ------------------------------------------ | ------------------------------------------------------------ |
| Hostname                           | `LMBS1263PC05`                             | Terminal prompt                                              |
| Operating system                   | Ubuntu 22.04.5 LTS (`jammy`)               | `lsb_release -a`                                             |
| Kernel                             | `6.8.0-138-generic`                        | `uname -r` at setup time                                     |
| CPU                                | Intel Core i9-13900K, 32 logical CPUs      | `lscpu`                                                      |
| RAM                                | 62 GiB                                     | `free -h`                                                    |
| GPU                                | NVIDIA GeForce RTX 4090, 24,564 MiB VRAM   | `nvidia-smi`                                                 |
| NVIDIA driver                      | `580.65.06`                                | `nvidia-smi` after driver update                             |
| Driver-reported CUDA compatibility | `13.0`                                     | The `CUDA Version` reported by `nvidia-smi` is **not** the PyTorch build version |
| GPU PyTorch runtime                | PyTorch `2.7.0+cu128`, CUDA runtime `12.8` | Successful CUDA tensor/matrix multiplication test            |
| GPU display usage                  | GPU also drives Ubuntu desktop             | May reduce available VRAM for simulation                     |

### 2.2 RoboDojo Python environment

| Component                        | Observed version / state                                  |
| -------------------------------- | --------------------------------------------------------- |
| Isaac Sim                        | `5.1.0.0`                                                 |
| Isaac Lab                        | `0.54.3`                                                  |
| Python in simulation environment | `3.11`                                                    |
| NumPy                            | `1.26.0`                                                  |
| cuRobo                           | Available; `nvcc` check passed                            |
| Git LFS                          | Installed                                                 |
| RoboDojo source commit           | `ee67a1468510da7624a089164402359f2afc72c8`                |
| RoboDojo assets commit           | `91f76c28d93dd20c5fa46ce6a5a1d96a4f384acd`                |
| RAX repository commit            | **Record using `git rev-parse HEAD`** (not captured here) |

A previous dependency check detected `isaaclab 0.54.3` requiring `starlette==0.49.1` while `0.45.3` was installed, as well as an `idna` mismatch for `httpx2`. The RAX doctor nevertheless passed; keep these warnings in the record if relevant to debugging, and avoid gratuitous package upgrades within the pinned simulator environment.

### 2.3 Container runtime and agent

| Component                   | Observed version / status                                    |
| --------------------------- | ------------------------------------------------------------ |
| Docker Engine               | `29.9.3`                                                     |
| containerd                  | `2.4.2` installed                                            |
| Docker image                | `robodojo-auto-research:dev` successfully built              |
| Docker access               | Current shell works when it has the `docker` group (`docker ps`) |
| Codex CLI                   | `0.167.0` observed in an agent execution log; host CLI version may differ |
| Model provider              | Custom OpenAI Responses API profile                          |
| Configured model            | `gpt-6.1-sol`                                                |
| Configured reasoning effort | `xhigh`                                                      |

**Verification:** `bash scripts/robodojo_doctor.sh` returned `All checks passed.` This is a dependency/readiness check, **not** proof of a successful formal evaluation.

## 3. Storage layout

The RAX project uses the user's own Lenovo PS6 portable SSD. The device was reported as a 931.5 GiB external drive with several partitions; the **RAX partition** is an ext4 filesystem of about 469 GiB (`/dev/sdb5` on this host). Device letters (`sdb`) can change, so commands should find the partition by UUID.

```text
UUID: f583df79-1c0f-4507-85f1-d50f4625976f
Mount: /media/huayi/f583df79-1c0f-4507-85f1-d50f4625976f
```

Recorded disk state at setup: 469 GiB total, 157 GiB used, 289 GiB available. This is a **historical snapshot** and will change as sessions accumulate.

```text
<SSD mount>/
├── rax-project/
│   └── rax/                            # main RAX repository
└── rax-runtime/
    ├── runtime/                         # Python env, caches, sessions
    │   ├── envs/robodojo/
    │   ├── robodojo-cache/
    │   └── auto-research/<session-id>/
    ├── BEHAVIOR-1K/                     # source checkout (optional backend)
    ├── docker-data/                     # Docker image and container storage
    └── containerd-data/                 # containerd storage
```

Important symlinks under the repository:

- `runtime` -> `<SSD mount>/rax-runtime/runtime`
- `BEHAVIOR-1K` -> `<SSD mount>/rax-runtime/BEHAVIOR-1K`
- `RoboDojo/.cache` -> `<SSD mount>/rax-runtime/runtime/robodojo-cache`
- `RoboDojo/Assets` -> `<SSD mount>/rax-runtime/runtime/robodojo-cache/robodojo_assets_repo/Assets`

Docker paths:

- `/etc/docker/daemon.json`: `data-root = <SSD mount>/rax-runtime/docker-data`
- `/etc/containerd/config.toml`: `root = <SSD mount>/rax-runtime/containerd-data`
- `docker.service` and `containerd.service` have `ExecStartPre=/usr/bin/mountpoint -q <SSD mount>` to prevent starting with the drive absent.

**Do not unplug the drive while Docker, Isaac Sim, or RAX is running.** The host's unrelated disks are not part of this RAX setup.

## 4. Setup performed (one-time)

These commands document installation stages that **have already been completed**. Do not rerun destructively on a working environment. The original repository and documentation contain additional prerequisites, pinned dependencies, licenses, and installation details.

### 4.1 Initialize paths (also do this for every new shell)

```bash
export RAX_DISK="$(findmnt -rn \
  -S UUID=f583df79-1c0f-4507-85f1-d50f4625976f \
  -o TARGET)"

if [ -z "$RAX_DISK" ] || ! mountpoint -q "$RAX_DISK"; then
  echo 'ERROR: RAX SSD is not mounted'
  return 1 2>/dev/null || exit 1
fi

export RAX_HOME="$RAX_DISK/rax-project/rax"
export ROBODOJO_RUNTIME_ROOT="$RAX_DISK/rax-runtime/runtime"
export PATH="$HOME/.local/bin:$RAX_DISK/rax-runtime/tools/bin:$PATH"
cd "$RAX_HOME"
```

### 4.2 Prepare source and assets

```bash
# Run within RAX root on initial installation only
bash scripts/bootstrap_sources.sh
bash scripts/setup_robodojo_env.sh --accept-nvidia-eula
bash scripts/setup_assets.sh
```

| Command                 | Purpose                                                      |
| ----------------------- | ------------------------------------------------------------ |
| `bootstrap_sources.sh`  | Retrieve pinned RoboDojo source/submodules, patches, and BEHAVIOR source |
| `setup_robodojo_env.sh` | Build simulator Python environment (Isaac Sim/Lab/cuRobo and dependencies) |
| `setup_assets.sh`       | Retrieve RoboDojo assets and layouts                         |

### 4.3 Prepare the offline agent image and verify host

```bash
runtime/envs/robodojo/bin/python scripts/build_agent_image.py
bash scripts/robodojo_doctor.sh
```

| Command                | Purpose                                                      |
| ---------------------- | ------------------------------------------------------------ |
| `build_agent_image.py` | Build the RAX agent image `robodojo-auto-research:dev`       |
| `robodojo_doctor.sh`   | Validate pinned files, simulator environment, Docker, Codex, assets, and GPU |
| `docker ps`            | Validate access to Docker daemon without `sudo`              |
| `nvidia-smi`           | Inspect GPU/driver and current VRAM consumption              |

If `docker ps` fails due to group permissions, start a shell with updated membership using `newgrp docker` **only if already authorized to be in that group**. Docker group access carries root-equivalent privileges on the host.

## 5. Model provider: OpenAI API with Codex

The current experimental profile is stored **on the host**, outside the Git repository:

```text
~/.codex/rax-openai/openai-sol.config.toml
~/.codex/rax-openai/secrets/openai_api_key
```

Representative profile (no credentials shown):

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

`--agent-provider custom` in the session selects this host-side profile. `--provider-home` and `--profile` in the launcher choose the provider configuration. API usage is billed separately from ChatGPT subscription usage. Keep the key private and check model availability/permission on the API account.

To check for the files **without displaying the secret**:

```bash
test -s "$HOME/.codex/rax-openai/secrets/openai_api_key" && echo 'API key file present'
test -f "$HOME/.codex/rax-openai/openai-sol.config.toml" && echo 'Provider profile present'
```

## 6. Current simulation configuration

The experiment currently being attempted is a **minimal RoboDojo `stack_blocks` run**. The options below reflect the commands used to prepare the Sol session; always consult the session's `operator-input.json` for the authoritative saved values.

| Parameter                         | Value                  | Meaning                                                      |
| --------------------------------- | ---------------------- | ------------------------------------------------------------ |
| Benchmark                         | `robodojo`             | Tabletop dual-arm manipulation benchmark                     |
| Robot                             | Dual ARX X5 arms       | RoboDojo simulated robot                                     |
| Task                              | `stack_blocks`         | Block-stacking task                                          |
| Simulator                         | Isaac Sim 5.1          | Physics simulation                                           |
| Agent CLI                         | `codex`                | Coding agent                                                 |
| Provider                          | `custom` -> OpenAI API | Model transport                                              |
| Model                             | `gpt-6.1-sol`          | In host provider profile, not necessarily in session JSON    |
| Reasoning effort                  | `xhigh`                | Codex reasoning setting                                      |
| Simulation GPU                    | `0`                    | RTX 4090                                                     |
| Exploration episodes              | `2`                    | Small budget; rehearsal also consumes an episode             |
| Parallel exploration environments | `1`                    | Run one exploration simulator                                |
| Subagents                         | `0`                    | No additional agents                                         |
| Formal episodes                   | `2`                    | Two held-out test episodes                                   |
| Formal workers                    | `1`                    | One formal worker                                            |
| Agent formal                      | `off`                  | Evaluate frozen generated code rather than live agent copies |
| External perception               | `off`                  | Do not use additional Gemini perception                      |

The observation profile, demonstration context, task rules, seed settings, and exact task success criteria were **not independently extracted from the saved Sol session in the available logs**; check the JSON, `TASK.md`, and RoboDojo task code rather than claiming exact values from memory.

### 6.1 Create a fresh session

```bash
cd "$RAX_HOME"

CONFIG="$(runtime/envs/robodojo/bin/python \
  scripts/configure_auto_research.py \
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
  --external-perception off)" || exit 1

test -f "$CONFIG" || { echo "Invalid CONFIG: $CONFIG"; exit 1; }
echo "CONFIG=$CONFIG"
```

**Purpose:** generate a new isolated session and its `operator-input.json`. Configuration does not yet mean the agent has started, and it should not be repeated for a resume.

### 6.2 Launch (billed API requests may begin)

```bash
bash scripts/start_auto_research_agent.sh \
  --config "$CONFIG" \
  --provider-home "$HOME/.codex/rax-openai" \
  --profile openai-sol \
  --non-interactive
```

**Purpose:** launch the session's Agent container, simulator/MCP service, exploration, controller creation, and eligible evaluation. Use `tmux` for long runs if desired.

### 6.3 Resume an interrupted session

```bash
# Reuse the SAME operator-input.json, not a new configuration
bash scripts/start_auto_research_agent.sh \
  --config "$CONFIG" \
  --provider-home "$HOME/.codex/rax-openai" \
  --profile openai-sol \
  --non-interactive \
  --resume
```

**Purpose:** continue an already launched session. This is needed after interruption; running the launch command again without `--resume` produces `FileExistsError: This session was already launched`.

For a new terminal, `$CONFIG` may be empty. Explicitly set it to the correct `operator-input.json` path instead of relying on a shell variable surviving.

### 6.4 Stop

```bash
bash scripts/stop_auto_research_agent.sh --config "$CONFIG"
```

**Purpose:** stop the specified session. Do not stop the whole Docker service or delete lock files manually.

## 7. Inspect the experiment and results

### 7.1 Locate sessions

```bash
find "$RAX_HOME/runtime/auto-research" \
  -mindepth 2 -maxdepth 2 -name operator-input.json -print
```

Historic ChatGPT-backed session observed in the setup conversation:

```text
runtime/auto-research/20261109T041728Z_e5f9bc2f8f/
```

**Do not assume this is the new OpenAI API + Sol session.** The Sol session's ID was not provided in the available logs.

Read the **actual** session configuration:

```bash
python3 -m json.tool "$CONFIG"
SESSION="$(dirname "$CONFIG")"
```

### 7.2 Review the task, generated controller, and progress

```bash
find "$SESSION/agent-workspace" \
  \( -name TASK.md -o -name AGENTS.md -o -name PROGRESS.md \) -print
find "$SESSION/agent-workspace" -name controller.py -print
```

- `TASK.md`: task instructions delivered to the agent.
- `AGENTS.md` / `PROGRESS.md`: agent policy and work-in-progress notes.
- `code/**/controller.py`: program generated by the agent, if one exists.

### 7.3 Check lifecycle and formal evaluation

```bash
for file in agent-lifecycle.json formal.json formal-scores.json; do
  if [ -f "$SESSION/$file" ]; then
    echo "=== $file ==="
    python3 -m json.tool "$SESSION/$file"
  else
    echo "Not yet present: $file"
  fi
done
```

**Interpretation:** an Agent starting an episode, or completing `register`, does not alone establish formal success. `formal-scores.json` appears only when scoring has been written; check status, per-episode outcomes, and visual evidence.

### 7.4 Inspect logs and videos

```bash
[ -f "$SESSION/agent.log" ] && tail -n 100 "$SESSION/agent.log"
[ -f "$SESSION/frontend.log" ] && tail -n 100 "$SESSION/frontend.log"
find "$SESSION/native/results" -type f -name '*.mp4' 2>/dev/null
```

**Purpose:** troubleshoot model or simulator errors; inspect recorded manipulation attempts and formal episodes.

### 7.5 Dashboard (on the same Ubuntu computer)

```bash
cd "$RAX_HOME"
bash scripts/run_operator_dashboard.sh
```

Open the complete local URL printed by the script in the Ubuntu browser; the URL may contain an access token. Keep the token private. The dashboard can show sessions, logs, camera frames, videos, and scores when available.

## 8. Important troubleshooting notes from this reproduction

| Symptom observed                                             | Cause / response                                             |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `No such file or directory: scripts/start_auto_research_agent.sh` | Shell was in `~`, not the RAX project root. Run `cd "$RAX_HOME"`. |
| `CONFIG=` empty, then `IsADirectoryError: '.'`               | New shell lost `$CONFIG`. Point it to the real `operator-input.json`. |
| Docker socket `permission denied`                            | Current shell lacked docker group membership. `newgrp docker` worked once membership was granted. |
| `--non-interactiveCONFIG=` unrecognized                      | Two separate commands were accidentally pasted together. Execute configuration and launch in separate blocks. |
| `FileExistsError: This session was already launched`         | Use `--resume` with the same config, after confirming no active launcher still owns it. |
| Codex `You've hit your usage limit`                          | Earlier ChatGPT subscription-backed run exhausted account quota; it was **not** an Isaac Sim failure. New OpenAI API run is billed separately. |
| `KeyboardInterrupt` inside `subprocess.wait`                 | The launch was manually interrupted. Resume the existing session instead of creating a new one. |
| `docker ps` returns only a header                            | Docker access works; no containers are currently running.    |

## 9. Research modification points

| What to change                                            | Where to look                                                |
| --------------------------------------------------------- | ------------------------------------------------------------ |
| Agent's exploration and program-writing behavior          | `auto_research_agent/AGENTS.md`, `auto_research_agent/START_PROMPT.md` |
| Robotics knowledge and tool-use instructions              | `agent_skills/`                                              |
| Workspace generation and task prompt composition          | `services/workspace/`                                        |
| Task, budget, layouts, observation / formal configuration | `scripts/configure_auto_research.py`, `services/controller/config.py` |
| RoboDojo environment and task implementation              | `RoboDojo/`, `services/robodojo/`                            |
| MCP actions and formal lifecycle                          | `services/controller/`                                       |
| Model provider / relay / CLI behavior                     | `harness/codex_cli/`                                         |
| Unit and integration tests                                | `local_tests/`                                               |

For scientifically meaningful method comparisons, preserve the same task, layout collections, random seeds, exploration budgets, and formal evaluation protocol; modify one methodological element at a time and record the Git commit of the change.

## 10. Reproducibility checklist

- [x] External SSD mounted and repository available
- [x] RTX 4090 recognized and PyTorch CUDA matrix multiplication passed
- [x] Isaac Sim 5.1 and RoboDojo assets installed
- [x] Docker and containerd data directed to personal SSD
- [x] Agent Docker image successfully built
- [x] `robodojo_doctor.sh` reported `All checks passed`
- [x] Codex Agent launch reached execution in a session
- [ ] Record exact RAX repository commit and active Sol session ID
- [ ] Archive the active `operator-input.json` and task `TASK.md`
- [ ] Confirm Sol/OpenAI API calls succeeded in the active session
- [ ] Confirm code was registered, rehearsal passed, and formal evaluation finished
- [ ] Record formal scores, success rate, videos, and API consumption

## 11. Quick daily commands

```bash
# 1. Environment (new shell)
export RAX_DISK="$(findmnt -rn -S UUID=f583df79-1c0f-4507-85f1-d50f4625976f -o TARGET)"
export RAX_HOME="$RAX_DISK/rax-project/rax"
export ROBODOJO_RUNTIME_ROOT="$RAX_DISK/rax-runtime/runtime"
export PATH="$HOME/.local/bin:$RAX_DISK/rax-runtime/tools/bin:$PATH"
cd "$RAX_HOME"

# 2. Verify access
docker ps
nvidia-smi

# 3. Find existing session configurations
find runtime/auto-research -mindepth 2 -maxdepth 2 -name operator-input.json -print

# 4. Set the correct CONFIG explicitly for an existing session
# export CONFIG="$RAX_HOME/runtime/auto-research/<correct-session-id>/operator-input.json"

# 5. For a previously launched, stopped session ONLY:
# bash scripts/start_auto_research_agent.sh --config "$CONFIG" \
#   --provider-home "$HOME/.codex/rax-openai" --profile openai-sol \
#   --non-interactive --resume

# 6. View dashboard
bash scripts/run_operator_dashboard.sh
```

## References

- [RAX project and paper](https://robot-agent-exploration.com/)
- [RAX repository](https://github.com/robot-agent-exploration/rax)
- [Container branch setup](https://github.com/robot-agent-exploration/rax/blob/robodojo-behavior/container/docs/SETUP.md)
- [Container branch usage](https://github.com/robot-agent-exploration/rax/blob/robodojo-behavior/container/docs/USAGE.md)
- [RAX architecture](https://github.com/robot-agent-exploration/rax/blob/robodojo-behavior/container/docs/ARCHITECTURE.md)
- [RAX dashboard](https://github.com/robot-agent-exploration/rax/blob/robodojo-behavior/container/services/dashboard/README.md)

---

**Data provenance:** Machine specifications, installed package versions, disk paths, successful checks, run commands, and errors are recorded from the terminal outputs collected during this reproduction. Values not verified in those outputs are marked explicitly as unconfirmed rather than assumed.
