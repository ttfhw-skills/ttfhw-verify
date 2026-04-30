# ttfhw-verify

TTFHW (Time To First Hello World) 仓库编译验证 Skill。

验证仓库能否成功编译和运行单元测试，输出 JSON 格式的客观指标报告。

## 安装

使用 skills CLI 安装：

```bash
npx skills add ttfhw-skills/ttfhw-verify
```

安装后可在 Claude Code 中使用 `/ttfhw-verify` 命令。

## 功能

- 克隆仓库并检查文档完整性
- 自动选择合适的 Docker 镜像
- 循环尝试解决构建依赖问题（不修改源码）
- 运行单元测试
- 生成 JSON 验证报告

## 使用

```bash
/ttfhw-verify                  # 交互式验证
/ttfhw-verify example-repo     # 直接验证指定仓库
/ttfhw-verify --remote IP      # 指定远端服务器
```

## 输出

验证完成后生成 `build-verification/<repo>/verification_report.json`，包含：

- 构建状态和产物
- UT 通过/失败数量
- 各阶段时长统计
- 所有尝试记录日志

## ⚠️ 安全警告

本 Skill 涉及高风险操作，请务必遵循以下安全要求：

| 要求 | 说明 |
|:---|:---|
| **隔离环境** | 仅在专用测试服务器上运行，禁止在生产环境使用 |
| **SSH 安全** | 确保 SSH 连接目标是用户可控的服务器 |
| **容器隔离** | 所有操作在 Docker 容器内完成，不影响宿主机 |
| **参数审查** | 用户应审查所有 AI 生成的命令后再执行 |
| **人工监督** | 关键操作需用户确认，不建议完全自动化 |

**推荐配置**：
- 使用专用 VM 或容器作为远端服务器
- 配置 SSH 免密登录但限制 sudo 权限
- 定期清理 Docker 容器和镜像

## 许可证

MIT License