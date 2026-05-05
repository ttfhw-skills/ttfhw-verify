# TTFHW 验证报告模板说明

报告模板位于：`build-verification/example_verification_report.json`

## 核心字段说明

### meta
报告元信息：
- `report_version`: "2.0"
- `skill_name`: "ttfhw-verification"
- `generated_at`: ISO格式时间戳
- `generator`: "Claude Code"
- `remote_server`: 远端服务器地址（用户提供）

### repo_info
仓库基本信息：
- `name`: 仓库名
- `url`: Git URL
- `branch`: 分支名
- `commit_hash`: 提交hash
- `clone_depth`: 克隆深度（通常为1）
- `clone_duration_seconds`: 克隆耗时
- `repo_size_mb`: 仓库大小

### documentation_checklist
文档检查清单（全布尔值）：
- `readme_found`: 是否有README
- `build_docs_found`: 是否有构建文档
- `dockerfile_found`: 是否有Dockerfile
- `readme_sections`: 各章节存在性
- `build_doc_sections`: 构建文档章节存在性

### image_selection
镜像选择信息（必须呈现）：
- `method`: 选择方法（repo_documentation/repo_dockerfile/standard_image）
- `selected_image`: 选用的镜像
- `repo_dockerfile_used`: 是否使用仓库Dockerfile
- `image_tool_versions`: 镜像内工具版本

### hardware_config
硬件配置：
- `server`: 服务器标识
- `ssh_target`: SSH地址
- `cpu_cores`: CPU核数
- `memory_total_gb`: 内存大小
- `npu_available`: NPU是否可用
- `npu_count`: NPU数量

### memory_detection
内存检测与并发计算（新增）：
- `memory_available_mb`: 可用内存（MB）
- `memory_available_gb`: 可用内存（GB）
- `memory_detection_formula`: 公式描述（如："available / 3.5GB"）
- `safe_concurrency_limit`: 内存限制的安全并发数
- `cpu_cores_available`: CPU核数
- `recommended_concurrency`: 推荐并发数（取CPU和内存限制的较小值）
- `final_concurrency_used`: 最终使用的并发数
- `oom_occurred`: 是否发生OOM
- `oom_retry_count`: OOM重试次数

### build_result
构建结果：
- `status`: success/failed
- `start_time`: 开始时间
- `end_time`: 结束时间
- `artifacts`: 构建产物列表
- `error`: 错误信息（失败时）

### ut_stats
UT统计：
- `ut_framework`: 测试框架
- `ut_total_count`: 总测试数
- `ut_passed_count`: 通过数
- `ut_failed_count`: 失败数
- `ut_skipped_count`: 跳过数
- `ut_duration_seconds`: 耗时

### quick_start_analysis
Quick Start分析：
- `has_quick_start_section`: 是否有quick start章节
- `example_found`: 是否找到示例
- `example_runnable`: 示例能否运行
- `example_run_result`: 运行结果

### ttfhw_timeline
时间分解（必须呈现）：
- `clone_duration_seconds`: 克隆时长
- `docs_reading_estimate_seconds`: 文档阅读估算
- `dependency_install_duration_seconds`: 依赖安装时长
- `build_duration_seconds`: 构建时长
- `ut_duration_seconds`: UT时长
- `total_ttfhw_seconds`: 总TTFHW时长
- `total_attempt_duration_seconds`: 从容器启动到结果的时长

### attempt_log
尝试记录（必须呈现）：
- `total_attempts`: 总尝试次数
- `timeout_reached`: 是否超时
- `attempts`: 按时间顺序的尝试列表

### overall
整体结果（纯客观）：
- `result`: success/partial_success/failed
- `buildable`: 是否可构建
- `testable`: 是否可测试
- `example_runnable`: 示例是否可运行
- `total_attempts`: 总尝试次数
- `timeout_reached`: 是否超时

## 测量标准

| 类型 | 标准 |
|:---|:---|
| 时间 | 秒（duration_seconds） |
| 计数 | 整数 |
| 布尔 | true/false |
| 不适用 | null |
| 百分比 | 浮点数（如85.2） |
| 超时 | 7200秒 |

## overall.result判定

| 取值 | 条件 |
|:---|:---|
| `success` | buildable=true + testable=true + example_runnable=true |
| `partial_success` | buildable=true + (testable=false 或 UT部分失败) |
| `failed` | buildable=false 或 timeout_reached=true |