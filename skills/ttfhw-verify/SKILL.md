---
name: ttfhw-verify
description: Verify repository build and unit test execution. Use when validating compilation capabilities, running CI checks, or measuring TTFHW (Time To First Hello World) metrics. Outputs JSON reports with build stats, test results, and timeline data.
---

> **安全声明 / Security Disclaimer**
>
> **风险级别**：HIGH — 本 Skill 涉及远程 SSH 连接、容器命令执行、依赖安装等高风险操作。
>
> **安全措施**：
> 1. **用户主动授权**：所有连接参数（IP、工作目录）需用户明确提供，Skill 不自动推测
> 2. **容器隔离**：所有构建和测试在 Docker 容器内完成，不影响宿主机
> 3. **人工监督**：关键操作（如 SSH 连接、容器启动）需用户确认后才执行
> 4. **源码保护**：Skill 严禁修改被验证仓库的源代码
> 5. **推荐环境**：仅在用户可控的隔离服务器上运行，不建议在生产环境使用
>
> **无恶意行为**：本 Skill 不包含凭据窃取、数据外泄、后门植入等恶意代码。所有操作可审计、可追溯。

---

# TTFHW 仓库编译验证

验证仓库编译和UT运行情况，量化TTFHW能力。

**输出**：JSON报告（遵循`build-verification/example_verification_report.json`模板）

**核心原则**：只采集客观指标，不主观评分。所有编译在容器内进行。

---

## 快速使用

```
/ttfhw-verify                  # 交互式验证
/ttfhw-verify example-repo   # 直接验证指定仓库
/ttfhw-verify --remote IP      # 指定远端服务器
```

---

## 执行铁律

1. **本地阅读 + 远端编译**：本地`tmp_src/<repo>`读文档，远端`<REMOTE_WORKDIR>/<repo>`挂载容器编译
2. **同一容器完成构建+UT**：启动容器后完成所有操作
3. **主动尝试解决问题**：失败时尝试解决（安装依赖、调配置），**严禁修改源码**
4. **2小时时间限制**：最多尝试7200秒，超时输出当前报告
5. **记录所有尝试**：每个动作记录在`attempt_log`
6. **实时同步报告**：scp同步到本地`build-verification/<repo>/`
7. **容器命名**：`wsk-<repo>`前缀
8. **文档优先原则**：**必须优先查看仓库相关文档**获取编译指导，按以下顺序阅读：
   - README.md（项目介绍、快速开始）
   - docs/目录（构建文档、API文档）
   - .devcontainer/目录（VS Code开发容器配置，包含镜像、依赖、环境变量等关键信息）
   - build.md / CONTRIBUTING.md（构建指南）
   
   仅在文档缺失或遇到无法解决的问题时，才查阅源码（如CMakeLists.txt、Makefile、setup.py等）进行分析解决

**关键路径**：
```
本地 tmp_src/<repo> → 读文档、提取构建信息
远端 <REMOTE_WORKDIR>/<repo> → 挂载容器 /workspace
容器内: 安装依赖 → 构建 → UT → 示例
远端 → 本地: scp同步报告
```

---

## Phase 1: 交互式初始化

### 1.0 上下文检查（首先执行）

**在询问用户之前，先检查当前对话上下文是否已提供足够信息**：

检查以下关键参数是否已在对话中明确：
| 参数 | 判断条件 | 可跳过询问 |
|:---|:---|:---|
| **验证仓库** | 用户明确指定仓库名或URL | ✓ |
| **验证环境** | 用户说明"远端"/"本地"，或之前对话已确认 | ✓ |
| **SSH认证** | 用户说明"免密已完成"，或之前对话已确认 | ✓ |
| **远端IP** | 用户明确提供IP地址 | ✓ |
| **远端工作目录** | 用户提供或使用默认`~/workspace` | ✓ |

**判断逻辑**：
```
IF 所有5个参数均已提供:
    直接进入Phase 2执行验证
    record attempt: "跳过询问，使用已确认参数"
ELSE:
    执行下方询问流程（仅询问缺失的参数）
```

> **重要**：不要重复询问已确认的参数。如果用户在创建skill时已确认"远端环境、SSH免密已完成"，后续验证直接使用这些参数。

### 1.1 仓库确认（仅当未指定时询问）

若用户未指定仓库，读取`ttfhw.txt`并询问：
- 本次验证范围（单个仓库/指定类型/全部）

### 1.2 环境确认（仅当未指定时询问）

若用户未指定环境，询问选择：

| 选项 | 说明 |
|:---|:---|
| **远端环境** | 本地读文档，远端服务器挂载容器编译（推荐） |
| **本地环境** | 所有操作本地完成，需本地Docker环境 |

### 1.3 远端配置（仅当选择远端且未确认时询问）

若选择远端但配置未确认，询问：
- **远端服务器IP**（必须由用户提供）
- **SSH免密认证是否完成？**
- **远端工作目录**（用户提供，默认`~/workspace`）

---

## 参数记录

无论是否询问，都应在报告中记录使用的参数：
```json
{
  "verification_context": {
    "repo": "<仓库名>",
    "url": "<仓库URL>",
    "environment": "remote/local",
    "remote_ip": "<用户提供的远端IP>",
    "remote_workdir": "<用户提供的远端工作目录>",
    "ssh_auth": "passwordless_verified/pending",
    "params_source": "user_input/previous_conversation"
  }
}
```

---

## Phase 2: 克隆（本地+远端）

```bash
# 本地：读文档
git clone --depth=1 <url> tmp_src/<repo>
gh repo view <repo> --json stargazersCount,forksCount  # 社区数据

# 远端：编译验证
ssh root@<REMOTE_IP> "cd <REMOTE_WORKDIR> && time git clone --depth=1 <url> <repo>"
```

记录`clone_duration_seconds`。

---

## Phase 3: 文档检查（本地）

```bash
cd tmp_src/<repo>
ls README.md docs/build.md Dockerfile
grep -qi "install\|quick start\|api" README.md  # 章节检查
```

生成`documentation_checklist`字段。

---

## Phase 4: 镜像选择

### 镜像选择流程（按优先级）

**优先级1: 从项目文档提取**

```bash
cd tmp_src/<repo>
# 检查devcontainer配置（重要：包含镜像、依赖、环境变量）
cat .devcontainer/devcontainer.json 2>/dev/null || cat .devcontainer.json 2>/dev/null
# 检查README和文档中的镜像信息
grep -rn "docker\|image\|镜像\|Dockerfile" README.md docs/*.md
# 检查Dockerfile
find . -iname "*dockerfile*"
```

若发现`.devcontainer/`目录或`devcontainer.json`文件，重点查看：
- `image`字段：指定的Docker镜像
- `build.dockerfile`：自定义Dockerfile路径
- `features`：预装的开发工具
- `extensions`：推荐安装的工具
- `postCreateCommand`：容器启动后的初始化命令

若文档中提及镜像或存在Dockerfile，提取并使用。记录`method: repo_documentation`

**优先级2: 交互式询问用户**

若文档未提及镜像，询问用户：
- 是否有指定的Docker镜像？
- 若无，提示用户需自行准备编译环境（gcc/cmake/python等）

记录`method: user_provided` 或 `method: none_specified`

> **注意**：若用户未提供镜像且文档未提及，需在容器启动后手动安装编译依赖。

---

## Phase 5: 启动容器（远端）

### 普通容器（无NPU）

```bash
ssh root@<REMOTE_IP> "docker run -d --name wsk-<repo> \
    -v <REMOTE_WORKDIR>/<repo>:/workspace -w /workspace \
    <镜像> sleep infinity"
```

### 带NPU容器（绑定1卡）

```bash
ssh root@<REMOTE_IP> "docker run -d --name wsk-<repo> \
    --user root \
    --device /dev/davinci0 \
    --device /dev/davinci_manager \
    --device /dev/devmm_svm \
    --device /dev/hisi_hdc \
    -v /usr/local/sbin/npu-smi:/usr/local/sbin/npu-smi \
    -v /usr/local/Ascend/driver:/usr/local/Ascend/driver \
    -v /usr/local/Ascend/firmware:/usr/local/Ascend/firmware \
    -v <REMOTE_WORKDIR>/<repo>:/workspace -w /workspace \
    <镜像> sleep infinity"
```

### 验证NPU设备（NPU依赖仓库必须执行）

```bash
ssh root@<REMOTE_IP> "docker exec wsk-<repo> bash -c '
    source /usr/local/Ascend/cann/set_env.sh 2>/dev/null || true
    source /usr/local/Ascend/nnal/atb/set_env.sh 2>/dev/null || true
    npu-smi info
'"
```

### 验证torch_npu可用

```bash
ssh root@<REMOTE_IP> "docker exec wsk-<repo> bash -c '
    source /usr/local/Ascend/cann/set_env.sh
    source /usr/local/Ascend/nnal/atb/set_env.sh
    python -c \"import torch; import torch_npu; print(torch_npu.npu.is_available())\" 2>&1
'"
```

---

## Phase 5.5: 资源检测与并发计算

**重要**：在构建前检测容器内存，自动计算安全并发数，避免 OOM 导致编译失败。

### 内存检测逻辑

**基于实际验证数据（libmcpp 案例）**：
- 实测：13GB 可用内存，4并发成功 → 每进程约 3.5GB 需求
- 公式：`SAFE_JOBS = 可用内存 / 3.5GB`
- 验证：13GB / 3.5 = 3.7 → 4并发 ✓ 匹配实际成功值

```bash
ssh root@<REMOTE_IP> "docker exec wsk-<repo> bash -c '
    # 检测可用内存（注意：用 available，不是 total）
    MEMORY_AVAILABLE_MB=\$(free -m | awk \"/Mem:/ {print \$7}\")
    MEMORY_AVAILABLE_GB=\$((MEMORY_AVAILABLE_MB / 1024))
    CPU_CORES=\$(nproc)

    # 计算安全并发数（每进程预留 3.5GB，实测约3GB峰值+0.5GB波动）
    SAFE_JOBS=\$((MEMORY_AVAILABLE_GB * 1000 / 3500))

    # 确保最小值为 1
    SAFE_JOBS=\$((SAFE_JOBS > 0 ? SAFE_JOBS : 1))

    # 取 CPU 核数和内存限制的较小值
    RECOMMENDED_JOBS=\$((CPU_CORES < SAFE_JOBS ? CPU_CORES : SAFE_JOBS))

    # 最终并发数：最小为 2（避免单线程过慢），最大不超过推荐值
    FINAL_JOBS=\$((RECOMMENDED_JOBS > 2 ? RECOMMENDED_JOBS : 2))

    echo \"检测结果：CPU=\${CPU_CORES}核，可用内存=\${MEMORY_AVAILABLE_GB}GB，推荐并发=\${FINAL_JOBS}\"

    # 保存到环境变量供后续构建使用
    echo \"FINAL_JOBS=\${FINAL_JOBS}\" > /tmp/concurrency.conf
'"
```

### 记录到 attempt_log

```json
{
  "sequence": N,
  "phase": "resource_check",
  "action": "Detect available memory and calculate safe concurrency",
  "command": "free -m && nproc",
  "result": "success",
  "output": "CPU=24, AvailableMemory=13GB, RecommendedJobs=4",
  "duration_seconds": 1
}
```

### 本地容器环境

若使用本地 Docker 环境（无 SSH）：

```bash
docker exec wsk-<repo> bash -c '
    MEMORY_AVAILABLE_MB=$(free -m | awk "/Mem:/ {print $7}")
    # ... 同上逻辑
'
```

### 公式推导依据

**实测数据来源**：
- 仓库：libmcpp（C++ 项目，Meson + Ninja 构建）
- 环境：24核CPU，15GB总内存，13GB可用内存
- 失败：默认24并发 → OOM → 139分钟失败
- 成功：手动降为4并发 → 338秒成功

**推算过程**：
```
13GB可用 / 4进程 = 3.25GB/进程（理论峰值）
实际峰值约 2.5-3GB（因未完全占满）
系统预留约 1-2GB（dbus、日志等）
结论：每进程安全预留 3.5GB
```

---

## Phase 6: 构建（循环尝试，最多2小时）

**使用检测到的并发数进行构建**：

```bash
START=$(date +%s)

ssh root@<REMOTE_IP> "docker exec wsk-<repo> bash -c '
    # 读取推荐的并发数
    source /tmp/concurrency.conf 2>/dev/null || FINAL_JOBS=2

    pip install -r requirements.txt | tee /tmp/pip.log
    apt install -y <deps> | tee /tmp/apt.log

    # 构建时使用安全并发数
    case \"<build_system>\" in
        meson|ninja)
            ninja -j \$FINAL_JOBS | tee /tmp/build.log
            ;;
        cmake)
            cmake --build . -- -j \$FINAL_JOBS | tee /tmp/build.log
            ;;
        make)
            make -j \$FINAL_JOBS | tee /tmp/build.log
            ;;
        *)
            <build_command> | tee /tmp/build.log
            ;;
    esac
'"
```

### 失败时循环尝试（不修改源码）

| 错误类型 | 典型输出 | 解决方案 |
|:---|:---|:---|
| **OOM/内存不足** | `Killed signal terminated program cc1plus` | 降低并发数：`FINAL_JOBS=$((FINAL_JOBS/2))` |
| **缺少头文件** | `fatal error: xxx.h: No such file` | `yum install xxx-devel` |
| **找不到库** | `Could not find xxx` | `export xxx_DIR=/path` 或 `yum install xxx-devel` |
| **链接错误** | `undefined reference to xxx` | `yum install libxxx-devel` |
| **Python模块缺失** | `ModuleNotFoundError: No module named 'xxx'` | `pip install xxx` |
| **Git submodule缺失** | `CMakeLists.txt not found in 3rdparty` | `git submodule update --init` |
| **Cargo版本问题** | `feature 'edition2024' is not stabilized` | `rustup install nightly` |

**OOM 特殊处理**（基于实测经验）：

若构建过程出现 OOM 错误（编译进程被 kill），按以下步骤处理：

1. **立即检测当前并发数**：
   ```bash
   docker exec wsk-<repo> bash -c 'cat /tmp/concurrency.conf'
   ```

2. **降低并发数并重试**：
   ```bash
   # 降低至当前值的一半
   docker exec wsk-<repo> bash -c '
       source /tmp/concurrency.conf
       NEW_JOBS=$((FINAL_JOBS / 2))
       NEW_JOBS=$((NEW_JOBS > 1 ? NEW_JOBS : 1))
       echo "降低并发：${FINAL_JOBS} -> ${NEW_JOBS}"
       echo "FINAL_JOBS=${NEW_JOBS}" > /tmp/concurrency.conf
   '

   # 清理失败的构建产物并重试
   ssh root@<REMOTE_IP> "docker exec wsk-<repo> bash -c '
       cd /workspace/builddir && ninja -t clean
       source /tmp/concurrency.conf
       ninja -j \$FINAL_JOBS
   '"
   ```

3. **记录尝试**：
   ```json
   {
     "phase": "build_retry",
     "action": "Reduce concurrency due to OOM",
     "result": "retrying",
     "previous_jobs": 24,
     "new_jobs": 12
   }
   ```

### 超时检查

```bash
ELAPSED=$(($(date +%s) - START))
if [ $ELAPSED > 7200 ]; then
    # 基于当前状态输出报告
fi
```

---

## Phase 7: UT（循环尝试）

```bash
ssh root@<REMOTE_IP> "docker exec wsk-<repo> bash -c '<ut_command> | tee /tmp/ut.log'"
```

失败时尝试：
- 缺依赖 → `pip install pytest-mock`
- 超时 → `--timeout=300`
- 需硬件 → `-k "not npu"`

---

## Phase 8: Quick Start示例

```bash
# 本地找示例命令
grep -A10 "quick start" README.md | grep -E "python|cargo|npm"

# 容器跑示例
ssh root@<REMOTE_IP> "docker exec wsk-<repo> bash -c '<命令>'"
```

---

## Phase 9: 依赖采集

成功时从日志提取，失败时从错误+文档提取。

---

## Phase 10: 生成报告

生成JSON报告，保存到远端`/workspace/verification_report.json`，然后scp同步到本地。

```bash
ssh root@<REMOTE_IP> "docker exec wsk-<repo> bash -c 'cat > /workspace/verification_report.json << EOF
{JSON内容}
EOF'"
scp root@<REMOTE_IP>:<REMOTE_WORKDIR>/<repo>/verification_report.json build-verification/<repo>/.
ssh root@<REMOTE_IP> "docker rm -f wsk-<repo>"
```

---

## 报告格式规范

输出JSON报告必须遵循`build-verification/example_verification_report.json`模板格式。

### 核心字段（必须呈现）

| 字段 | 说明 |
|:---|:---|
| `meta` | 报告元信息（version、generated_at等） |
| `repo_info` | 仓库名、URL、克隆时长 |
| `documentation_checklist` | 文档章节检查（布尔值） |
| `image_selection` | method/selected_image/tool_versions |
| `dependency_stats` | 依赖信息 |
| `build_result` | status/artifacts/error |
| `ut_stats` | 框架/通过数/失败数 |
| `quick_start_analysis` | 示例能否运行 |
| `ttfhw_timeline` | 各阶段时长 + total_attempt_duration_seconds |
| `attempt_log` | 所有尝试记录（sequence/phase/action/command/result/duration_seconds） |
| `overall` | result/buildable/testable/timeout_reached |

### overall.result取值

| 取值 | 条件 |
|:---|:---|
| `success` | buildable + testable + example_runnable |
| `partial_success` | buildable + (testable失败 或 UT部分失败) |
| `failed` | buildable失败 或 timeout |

---

## 尝试记录格式

```json
{
  "attempt_log": {
    "total_attempts": N,
    "timeout_reached": true/false,
    "total_attempt_duration_seconds": 从启动容器到最终结果的时长,
    "attempts": [
      {
        "sequence": 1,
        "phase": "image_selection/dependency/build/ut/dependency_fix/config_fix/timeout",
        "action": "动作描述",
        "command": "执行的命令",
        "result": "success/failed/partial_success/timeout",
        "output": "成功时的输出片段",
        "error_message": "失败时的错误信息",
        "duration_seconds": 该尝试的耗时
      }
    ]
  }
}
```

---

## 关键教训

### Git Submodule项目

克隆时必须初始化submodule：
```bash
git clone --recurse-submodules <url>
# 或克隆后手动初始化
git submodule update --init --recursive
```

### Cargo/Rust网络下载慢

配置USTC镜像：
```bash
echo '[source.crates-io]
replace-with = "ustc"

[source.ustc]
registry = "https://mirrors.ustc.edu.cn/crates.io-index"' > ~/.cargo/config.toml
```

### Pip网络下载慢

使用阿里云镜像：
```bash
pip install -i https://mirrors.aliyun.com/pypi/simple <package>
```

### UBS系列依赖链

```
ubs-comm (libhcom.so) → ubs-comm-devel → ubs-engine / ubs-mem
```

### CANN系列需要NPU环境

必须：
1. 正确挂载NPU设备
2. 挂载Ascend驱动和固件
3. 导入CANN环境变量
4. 验证npu-smi info正常

---

## 网络优化配置

### Pip镜像配置
```bash
pip config set global.index-url https://mirrors.aliyun.com/pypi/simple
```

### Cargo镜像配置
```bash
mkdir -p ~/.cargo
cat > ~/.cargo/config.toml << 'EOF'
[source.crates-io]
replace-with = "ustc"
[source.ustc]
registry = "https://mirrors.ustc.edu.cn/crates.io-index"
EOF
```

### Git代理配置（如需要）
```bash
git config --global http.proxy http://proxy-server:port
```

---

## 验证仓库列表

仓库列表位于`ttfhw.txt`，格式：
```
序号	URL
1	https://example.com/org/repo.git
...
```

用户可根据实际情况配置自己的仓库列表。

---

## 参考文件

- `build-verification/example_verification_report.json` - 报告模板（必须遵循）
- `ttfhw.txt` - 仓库列表
- `CLAUDE.md` - 完整验证指南（本skill基于此文档）