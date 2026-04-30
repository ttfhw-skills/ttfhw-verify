# TTFHW Verify Skill

验证仓库编译和UT运行情况，量化评估TTFHW（Time To First Hello World）能力。

## 功能

- 克隆仓库并检查文档完整性
- 选择合适的Docker镜像进行编译
- 自动尝试解决构建依赖问题
- 运行单元测试
- 生成JSON格式的验证报告

## 安装

将 `ttfhw-verify` 目录放入Claude Code的skills目录：

```bash
# 方法1：直接复制
cp -r ttfhw-verify ~/.claude/skills/

# 方法2：使用.skill文件安装
claude skills install ttfhw-verify.skill
```

## 使用方法

启动验证：

```
/ttfhw-verify                  # 交互式验证，逐步询问参数
/ttfhw-verify example-repo   # 直接验证指定仓库
```

### 交互式流程

1. **选择验证环境**：远端服务器 或 本地环境
2. **配置远端连接**（若选远端）：
   - 提供远端服务器IP地址
   - 确认SSH免密认证状态
   - 确认远端工作目录
3. **选择Docker镜像**：
   - 自动从项目文档提取镜像信息
   - 若无，询问用户是否提供指定镜像
4. **指定验证仓库**：单个仓库 / 指定类型 / 全部仓库
5. **开始验证**：自动执行克隆、构建、测试流程

## 前提条件

### 远端环境验证

- 远端服务器需有Docker环境
- 本地需配置SSH免密认证到远端服务器
- 远端需有足够存储空间（每个仓库约50-500MB）

### 本地环境验证

- 本地需有Docker环境
- 若验证NPU相关仓库，需有NPU硬件

## 输出格式

验证完成后生成JSON报告，保存到 `build-verification/<repo>/verification_report.json`。

报告包含以下核心字段：

| 字段 | 说明 |
|:---|:---|
| `repo_info` | 仓库信息、克隆时长 |
| `documentation_checklist` | 文档检查结果 |
| `image_selection` | 镜像选择方法和版本 |
| `build_result` | 构建状态、产物、错误信息 |
| `ut_stats` | 测试通过/失败数量 |
| `attempt_log` | 所有尝试记录 |
| `overall.result` | success / partial_success / failed |

## 验证规则

- **时间限制**：最多尝试2小时（7200秒）
- **问题解决**：自动尝试安装缺失依赖，但不修改源码
- **记录追踪**：每个操作步骤记录在attempt_log中

## 常见问题

### 构建失败常见原因

| 错误 | 解决方案 |
|:---|:---|
| 缺少头文件 | 自动安装 xxx-devel 包 |
| Python模块缺失 | 自动 pip install |
| Git submodule缺失 | 自动 git submodule init |

### 网络优化建议

```bash
# Pip镜像加速
pip config set global.index-url https://mirrors.aliyun.com/pypi/simple

# Cargo镜像加速
mkdir -p ~/.cargo
cat > ~/.cargo/config.toml << 'EOF'
[source.crates-io]
replace-with = "ustc"
[source.ustc]
registry = "https://mirrors.ustc.edu.cn/crates.io-index"
EOF
```

## 文件结构

```
ttfhw-verify/
├── SKILL.md          # Skill主文档
├── README.md         # 使用说明
├── references/
│   └── report_template.md  # 报告模板说明
└── evals/
    └── evals.json    # 测试用例
```

## 许可证

MIT License