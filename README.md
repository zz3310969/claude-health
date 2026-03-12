# claude-health

A collection of Claude Code skills for configuration health auditing and cloud migration readiness checks.

## Skills

### 🩺 health — Claude Code 配置健康审计

Systematically reviews your project's Claude Code setup using the six-layer framework:
`CLAUDE.md → rules → skills → hooks → subagents → verifiers`

- Detects project tier (Simple / Standard / Complex) and calibrates checks accordingly
- Runs three parallel diagnostic agents: Context Layer, Control + Verification Layer, Behavior Pattern
- Outputs a prioritized report: 🔴 Critical / 🟡 Structural / 🟢 Incremental

### ☁️ cloud-migrate — 分布式系统云迁移就绪度检查

对项目按六层框架做云迁移就绪度审计：
`Dockerfile → JVM 参数 → 配置外部化 → CI/CD 流水线 → sym3 平台配置 → Sentry 监控`

- 自动识别项目类型（Tomcat WAR / Dubbo JAR / Spring Boot / 前端 Nginx / 多模块混合）
- 4 个并行诊断子代理检查代码侧配置
- 通过浏览器自动化验证 sym3 平台配置和 Sentry 注册状态
- 平台地址永久保存，无需重复输入
- 支持单项目检查和 100+ 项目批量扫描
- 输出 6 维度就绪度评分报告

## Install

```bash
# 安装所有 skills
npx skills add zz3310969/claude-health

# 只安装某一个
npx skills add zz3310969/claude-health -s health
npx skills add zz3310969/claude-health -s cloud-migrate
```

## Usage

In any Claude Code session:

```
/health
/cloud-migrate
```

Or describe what you want:

> "Run a health check on my Claude Code config"

> "检查项目云迁移就绪状态"

> "批量扫描 /path/to/projects/ 的云迁移状态"

## What gets checked

### health

| Layer | Checks |
|-------|--------|
| CLAUDE.md | Signal-to-noise ratio, missing Verification/Compact Instructions sections, prose bloat |
| rules/ | Language-specific rules placement, coverage gaps |
| skills/ | Description token count, trigger clarity, auto-invoke appropriateness |
| hooks | Pattern field presence, file-type coverage, stale entries |
| allowedTools | Dangerous or stale one-time commands |
| Behavior | Rules violated in practice, repeated corrections, missing patterns |

### cloud-migrate

| 维度 | 检查项 |
|------|--------|
| Dockerfile | 基础镜像、ENV 分层、Sentry Agent、多阶段构建、.dockerignore |
| JVM 参数 | UseContainerSupport、参数分工（Dockerfile vs K8s）、硬编码检测、反模式 |
| 配置外部化 | 硬编码地址嗅探、configs/sym3/ 目录、Dubbo/Spring 配置规范 |
| CI/CD 流水线 | Maven Profile(-Psym3)、镜像 Tag 规则、分支触发策略、缓存配置 |
| sym3 平台 | 环境变量、ConfigMap、容器资源配置、测试 vs 线上一致性（在线验证）|
| Sentry 监控 | 服务注册状态、Agent 心跳检查（在线验证）|

## License

MIT
