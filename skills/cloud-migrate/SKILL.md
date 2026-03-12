---
name: cloud-migrate
description: 分布式系统云迁移就绪度检查。检测 Dockerfile、JVM 参数、CI/CD 流水线、配置外部化、前端容器化等维度，可通过浏览器自动化验证 sym3 平台配置和 Sentry 注册状态，支持单项目和批量扫描。输入 /cloud-migrate 或 "检查项目云迁移就绪状态" 触发。
---

# 分布式系统云迁移就绪度检查

对当前项目（或批量扫描多个项目）按六层框架做迁移就绪度审计：
`Dockerfile → JVM 参数 → 配置外部化 → CI/CD 流水线 → sym3 平台配置 → Sentry 监控`

目标：识别阻塞上云的硬伤、不符合平台规范的结构性问题、以及可渐进优化的改进项。支持通过浏览器自动化访问 sym3 平台和 Sentry 平台做在线验证。

---

## Step 0: 加载/初始化平台配置

首次运行时需要收集平台地址，之后永久保存到项目根目录的 `.cloud-migrate.json` 中（应加入 `.gitignore`）。

```bash
P=$(pwd)
CONFIG="$P/.cloud-migrate.json"

if [ -f "$CONFIG" ]; then
  echo "=== 已保存的平台配置 ==="
  cat "$CONFIG"
  echo ""
  echo "(如需修改，请删除 $CONFIG 后重新运行)"
else
  echo "CONFIG_NOT_FOUND"
fi
```

如果输出 `CONFIG_NOT_FOUND`，**必须向用户询问**以下信息（逐项确认，不能跳过）：

1. **sym3 测试环境地址** — 例如 `https://sym3-test.example.com`
2. **sym3 线上环境地址** — 例如 `https://sym3.example.com`
3. **Sentry 平台地址** — 例如 `https://sentry.example.com`
4. **当前项目在 sym3 上的应用名**（通常等于镜像名或服务名）

用户回答后，执行以下脚本保存配置：

```bash
P=$(pwd)
CONFIG="$P/.cloud-migrate.json"

cat > "$CONFIG" << 'JSONEOF'
{
  "sym3_test_url": "<用户提供的测试环境地址>",
  "sym3_prod_url": "<用户提供的线上环境地址>",
  "sentry_url": "<用户提供的 Sentry 地址>",
  "app_name": "<用户提供的应用名>",
  "created_at": "<当前时间>",
  "updated_at": "<当前时间>"
}
JSONEOF

echo "✅ 平台配置已保存到 $CONFIG"

# 确保 .gitignore 中排除此文件
if [ -f "$P/.gitignore" ]; then
  grep -q '.cloud-migrate.json' "$P/.gitignore" || echo '.cloud-migrate.json' >> "$P/.gitignore"
else
  echo '.cloud-migrate.json' > "$P/.gitignore"
fi
```

配置加载后进入项目识别。

---

## Step 0.5: 识别项目类型与扫描模式

先确认是**单项目检查**还是**批量扫描**：
- 如果当前目录下有 `pom.xml` / `package.json` / `Dockerfile`，进入**单项目模式**。
- 如果用户提供了父目录路径或明确要求批量扫描，进入**批量模式**（见 Step 6）。

对于单项目模式，运行以下脚本识别项目类型：

```bash
P=$(pwd)

echo "=== 项目类型识别 ==="
# 基础信息
echo "project_name: $(basename "$P")"
echo "has_pom: $(test -f "$P/pom.xml" && echo yes || echo no)"
echo "has_package_json: $(test -f "$P/package.json" && echo yes || echo no)"
echo "has_dockerfile: $(find "$P" -maxdepth 2 -name "Dockerfile" 2>/dev/null | head -5)"
echo "has_gitlab_ci: $(test -f "$P/.gitlab-ci.yml" && echo yes || echo no)"

# Java 项目类型判断
if [ -f "$P/pom.xml" ]; then
  echo "=== Maven 项目分析 ==="
  echo "packaging: $(grep -m1 '<packaging>' "$P/pom.xml" 2>/dev/null | sed 's/.*<packaging>\(.*\)<\/packaging>.*/\1/' || echo 'jar(default)')"
  echo "has_spring_boot: $(grep -l 'spring-boot' "$P/pom.xml" 2>/dev/null | wc -l)"
  echo "has_dubbo: $(grep -rl 'dubbo' "$P"/src/ 2>/dev/null | head -1 | wc -l)"
  echo "modules: $(grep '<module>' "$P/pom.xml" 2>/dev/null | sed 's/.*<module>\(.*\)<\/module>.*/  \1/')"
  echo "has_sym3_profile: $(grep -c '<id>sym3</id>' "$P/pom.xml" 2>/dev/null || echo 0)"
  echo "configs_dirs: $(ls -d "$P"/configs/*/ 2>/dev/null)"
fi

# 前端项目判断
if [ -f "$P/package.json" ]; then
  echo "=== 前端项目分析 ==="
  echo "has_nginx_conf: $(find "$P" -maxdepth 2 -name "nginx*.conf" 2>/dev/null | head -3)"
  echo "build_script: $(python3 -c "import json; d=json.load(open('$P/package.json')); print(d.get('scripts',{}).get('build','(none)'))" 2>/dev/null)"
  echo "has_dist_or_build: $(ls -d "$P"/{dist,build}/ 2>/dev/null | head -2)"
fi

echo "=== 子模块 Dockerfile 扫描 ==="
find "$P" -maxdepth 3 -name "Dockerfile" 2>/dev/null | while read -r df; do
  dir=$(dirname "$df")
  echo "  dockerfile: $df"
  echo "    base_image: $(grep -m1 '^FROM' "$df" 2>/dev/null)"
done
```

根据以上输出，确定项目属于以下哪种类型：

| 类型 | 识别信号 | 检查重点 |
|------|---------|---------|
| **Tomcat WAR** | packaging=war，基础镜像含 `gpx11-jdk8-tomcat` | Dockerfile 分层、CATALINA_OPTS、war 解压流程 |
| **Dubbo JAR** | 有 dubbo 依赖，packaging=jar，无 spring-boot | ENTRYPOINT 命令、JAVA_BASE_CONF 含 Sentry、lib/conf 目录 |
| **Spring Boot** | 有 spring-boot-maven-plugin，packaging=jar | Fat JAR 启动、JVM 参数外置、无内嵌配置 |
| **前端 (Nginx)** | 有 package.json、build 脚本、nginx.conf | 多阶段构建、Nginx 配置、静态资源路径 |
| **多模块混合** | pom.xml 含 \<module\>，多个 Dockerfile | 每个子模块独立检查，镜像名不能重复 |

**在后续所有 Step 中只检查与当前项目类型相关的规则。**

## Step 1: 收集配置快照 — Dockerfile 与镜像构建

```bash
P=$(pwd)

echo "=== 所有 Dockerfile 内容 ==="
find "$P" -maxdepth 3 -name "Dockerfile" 2>/dev/null | while read -r df; do
  echo "--- $df ---"
  cat "$df"
  echo ""
done

echo "=== .dockerignore ==="
cat "$P/.dockerignore" 2>/dev/null || echo "(none)"

echo "=== Sentry 相关 ==="
# pom.xml 中的 Sentry 版本
grep -r "sentry-javaagent" "$P/pom.xml" "$P"/*/pom.xml 2>/dev/null
# Dockerfile 中的 Sentry 配置
grep -r "sentry" "$P"/*/Dockerfile "$P/Dockerfile" 2>/dev/null

echo "=== JVM 参数检测 ==="
# Dockerfile 中所有 ENV 行
find "$P" -maxdepth 3 -name "Dockerfile" 2>/dev/null | while read -r df; do
  echo "--- $df ENV ---"
  grep -E '^ENV|JAVA_OPTS|CATALINA_OPTS|JAVA_BASE_CONF|JDK_JAVA_OPTIONS|Xmx|Xms|UseContainerSupport' "$df" 2>/dev/null
done
```

## Step 2: 收集配置快照 — CI/CD 与配置文件

```bash
P=$(pwd)

echo "=== .gitlab-ci.yml ==="
cat "$P/.gitlab-ci.yml" 2>/dev/null || echo "(none)"

echo "=== settings.xml ==="
if [ -f "$P/settings.xml" ]; then
  cat "$P/settings.xml"
else
  echo "(none — 可能需要创建)"
fi

echo "=== Maven Profile 检查 ==="
grep -A5 '<id>sym3</id>' "$P/pom.xml" "$P"/*/pom.xml 2>/dev/null || echo "(无 sym3 profile)"
grep -A5 '<id>dg-ol</id>' "$P/pom.xml" "$P"/*/pom.xml 2>/dev/null

echo "=== 配置文件结构 ==="
echo "-- configs/ 目录 --"
find "$P/configs" -type f 2>/dev/null | head -30
echo "-- application*.properties/yml --"
find "$P" -name "application*.properties" -o -name "application*.yml" 2>/dev/null | head -20
echo "-- dubbo*.properties --"
find "$P" -name "dubbo*.properties" 2>/dev/null | head -10

echo "=== 硬编码配置嗅探 ==="
# 搜索疑似硬编码的地址
grep -rn --include="*.properties" --include="*.yml" --include="*.xml" \
  -E '(jdbc:|mysql://|redis://|zookeeper://|[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+:[0-9]+)' \
  "$P/configs" "$P/src/main/resources" 2>/dev/null | head -30

echo "=== 前端配置（如适用）==="
if [ -f "$P/package.json" ]; then
  echo "-- nginx.conf --"
  find "$P" -maxdepth 2 -name "nginx*.conf" 2>/dev/null | while read -r f; do
    echo "--- $f ---"
    cat "$f"
  done
  echo "-- Dockerfile --"
  cat "$P/Dockerfile" 2>/dev/null
fi
```

---

## Step 3: 启动并行诊断子代理

启动**四个聚焦子代理**并行检查，每个负责一个维度。根据 Step 0 识别的项目类型，只传入相关检查项。

### Agent A — Dockerfile 与镜像构建审计

Prompt:
```
你是容器化迁移审计专家。审计以下项目的 Dockerfile 和镜像构建配置。

项目类型：[TOMCAT_WAR / DUBBO_JAR / SPRINGBOOT / FRONTEND / 多模块混合]

[粘贴 Step 1 的 Dockerfile 内容]

按项目类型检查以下规则：

【Tomcat WAR 项目】
- [ ] 基础镜像必须是 `ncr.nie.netease.com/webase/gpx11-jdk8-tomcat:8.5.99-4`
- [ ] ENV 分层：JAVA_BASE_CONF → CATALINA_OPTS → JAVA_OPTS 三层分工
- [ ] JAVA_BASE_CONF 必须包含：-Dlog.dir、-Dfile.encoding=UTF-8、-Dsun.jnu.encoding=UTF-8、-Duser.timezone=Asia/Shanghai
- [ ] CATALINA_OPTS = JAVA_BASE_CONF + UseContainerSupport + Sentry agent
- [ ] JAVA_OPTS / JDK_JAVA_OPTIONS 在 Dockerfile 中不赋值（留给 K8s 注入）
- [ ] CMD 为 ["/home/gpx/tomcat/bin/catalina.sh", "run"]
- [ ] EXPOSE 包含 8080
- [ ] 如有 JaCoCo 需求，必须多阶段构建（test + prod）
- [ ] APP_ROOT 固定为 /home/gpx/app/webroot

【Dubbo JAR 项目】
- [ ] 基础镜像必须是 `ncr.nie.netease.com/webase/gpx11-jdk:1.8.0-1`
- [ ] JAVA_BASE_CONF 中必须直接包含 Sentry agent 参数（无 CATALINA_OPTS）
- [ ] ENTRYPOINT 格式：sh -c "java ${JAVA_BASE_CONF} ${JDK_JAVA_OPTIONS} ${JAVA_OPTS} -cp \"lib/*:conf/\" com.alibaba.dubbo.container.Main"
- [ ] COPY 路径包含 target/lib/、target/conf/、target/*.jar
- [ ] APP_HOME 固定为 /home/gpx/app

【Spring Boot 项目】
- [ ] 基础镜像使用 JDK 镜像（非 Tomcat 镜像）
- [ ] -XX:+UseContainerSupport 必须存在
- [ ] 堆内存参数（-Xmx/-Xms）不能硬编码在 Dockerfile 中
- [ ] JAVA_OPTS 留空，由 K8s 注入
- [ ] 应使用 ENTRYPOINT ["java", ...] 或 sh -c 方式启动

【前端 Nginx 项目】
- [ ] 应使用多阶段构建：Stage 1 用 node 镜像 build，Stage 2 用 nginx 镜像
- [ ] nginx.conf 存在且配置合理（gzip、缓存头、SPA fallback）
- [ ] 不应在最终镜像中包含 node_modules 或源码
- [ ] EXPOSE 端口与 nginx.conf 中 listen 一致

【通用检查】
- [ ] .dockerignore 存在且排除 .git/、node_modules/、target/
- [ ] Sentry agent 版本：sentry-javaagent-collector 为 1.2.76-cmdb-register-SNAPSHOT
- [ ] Sentry agent JAR 文件名为 sentry-javaagent-premain-2.0.0.jar
- [ ] 无调试用 RUN ls -la 残留
- [ ] 无 root 用户敏感操作

输出格式：bullet points，按 [基础镜像] [ENV/JVM 分层] [Sentry] [构建流程] [安全/卫生] 分组
```

### Agent B — JVM 参数与运行时配置审计

Prompt:
```
你是 JVM 容器化调优专家。审计以下项目的 JVM 参数配置。

项目类型：[TOMCAT_WAR / DUBBO_JAR / SPRINGBOOT]
（前端项目跳过此代理）

[粘贴 Step 1 的 JVM 参数相关内容]

检查规则：

【参数分工】
- Dockerfile 中只应定义不随环境变化的基础参数（编码、时区、日志路径、Sentry）
- 堆内存（-Xmx/-Xms）、Spring Profile（-Dspring.profiles.active）等运行时参数必须通过 K8s 的 JAVA_OPTS / JDK_JAVA_OPTIONS 注入
- 严禁在 Dockerfile 中硬编码 -Xmx、-Xms 值

【容器化必备】
- [ ] -XX:+UseContainerSupport 必须存在（使 JVM 感知容器 cgroup 限制）
- [ ] 不应存在 -XX:+UseCGroupMemoryLimitForHeap（已废弃，被 UseContainerSupport 替代）
- [ ] 不应存在 -XX:MaxRAMFraction（粗粒度，推荐用 -XX:MaxRAMPercentage）
- [ ] 如果使用 MaxRAMPercentage，值应在 60-75% 之间

【Tomcat 特定】
- [ ] CATALINA_OPTS 引用了 ${JAVA_BASE_CONF}（不是复制内容）
- [ ] CATALINA_OPTS 中包含 Sentry -javaagent 参数
- [ ] Dockerfile 中没有给 JAVA_OPTS 赋值

【Dubbo 特定】
- [ ] JAVA_BASE_CONF 中直接包含 Sentry -javaagent 参数
- [ ] ENTRYPOINT 中引用了 ${JAVA_BASE_CONF} ${JDK_JAVA_OPTIONS} ${JAVA_OPTS}

【Spring Boot 特定】
- [ ] 启动命令中预留了环境变量注入点（如 ${JAVA_OPTS}）
- [ ] 没有在 Dockerfile 中固定 Spring Profile

【反模式检测】
- ENV JAVA_OPTS="-Xmx512m ..." 这种硬编码
- ENTRYPOINT 中直接写死内存参数
- 同时设置了 -Xmx 和 MaxRAMPercentage（冲突）
- 使用了 -XX:+UnlockExperimentalVMOptions 但没有明确理由

输出格式：bullet points，按 [参数分工] [容器化兼容] [Sentry 集成] [反模式] 分组
```

### Agent C — 配置外部化审计

Prompt:
```
你是微服务配置管理专家。审计以下项目的配置外部化状态。

项目类型：[TOMCAT_WAR / DUBBO_JAR / SPRINGBOOT / FRONTEND]

[粘贴 Step 2 的配置文件结构和硬编码嗅探结果]

检查规则：

【硬编码检测 — 必须外部化的配置】
以下类型的值不应出现在打入镜像的配置文件中，必须通过 K8s 环境变量或 ConfigMap 注入：
- [ ] 数据库连接串（jdbc:、mysql://、host:port）
- [ ] ZooKeeper 地址（zookeeper://、zk 集群地址）
- [ ] Redis 地址
- [ ] 消息队列地址（Kafka、RocketMQ broker）
- [ ] 具体 IP 地址 + 端口
- [ ] 环境相关的域名（如 test.xxx.com、prod.xxx.com）
- [ ] 密码、Token、Secret

【Dubbo 配置规范】
- [ ] dubbo.registry 必须使用平台变量：${ZK_${SYM3_CLUSTER}_ADDRESS}，禁止硬编码 ZK 地址
- [ ] dubbo.port 应使用固定值（如 20001）或平台变量
- [ ] configs/sym3/ 目录下应有 dubbo.properties

【Spring 配置规范】
- [ ] application.properties/yml 中不应包含环境相关值
- [ ] 推荐使用 Spring Profile 区分环境，运行时通过 -Dspring.profiles.active 注入
- [ ] 数据源配置应使用占位符（${DB_URL}）而非直接写连接串

【configs/ 目录规范】
- [ ] 应存在 configs/sym3/ 目录（平台规范环境目录）
- [ ] sym3/ 与 dg-ol/ 配置内容应一致（sym3 是从 dg-ol 复制后调整）
- [ ] 不同环境的差异应尽量只在连接地址、端口等外部依赖上

【前端配置规范】
- [ ] API 地址不应硬编码在前端代码中
- [ ] nginx.conf 中的 proxy_pass 后端地址应使用环境变量或可替换的模板
- [ ] 如有运行时配置，应通过 entrypoint 脚本注入（envsubst 等）

输出格式：bullet points，按 [硬编码告警（需立即修复）] [配置结构问题] [外部化建议] 分组
```

### Agent D — CI/CD 流水线审计

Prompt:
```
你是 GitLab CI/CD 和容器化部署专家。审计以下项目的 CI 流水线配置。

项目类型：[TOMCAT_WAR / DUBBO_JAR / SPRINGBOOT / FRONTEND / 多模块混合]

[粘贴 Step 2 的 .gitlab-ci.yml 和 settings.xml 内容]

检查规则：

【基础规范】
- [ ] 使用 Maven 镜像：ncr.nie.netease.com/webase/gpx11-jdk8-maven:3.8.8-2
- [ ] 镜像仓库鉴权：before_script 中 docker login ncr.nie.netease.com
- [ ] settings.xml 存在于项目根目录，且 mvn 命令使用 -s settings.xml

【Maven 构建】
- [ ] 必须使用 -Psym3（禁止 -Pdg-ol 或其他环境 profile）
- [ ] pom.xml 中必须定义 sym3 profile
- [ ] MAVEN_OPTS 中 maven.repo.local 路径与 cache.paths 一致（/home/gpx/.m2/repository）
- [ ] Runner tag 为 gpxrunner_gt

【镜像构建流程 — Tomcat WAR】
- [ ] package job 中有 mkdir -p webroot && find target/ -name "*.war" -exec unzip {} -d webroot
- [ ] 多阶段构建时 test stage 用 --target test，prod 用 --target prod

【镜像构建流程 — Dubbo JAR】
- [ ] artifacts 包含 target/*.jar、target/lib/、target/conf/
- [ ] package job 无 unzip 步骤（直接 docker build）

【镜像构建流程 — Spring Boot】
- [ ] artifacts 包含 target/*.jar
- [ ] docker build 命令正确

【镜像构建流程 — 前端】
- [ ] build job 使用 node 镜像执行 npm/yarn build
- [ ] package job 执行 docker build（Dockerfile 应含多阶段构建或直接 COPY dist/）

【镜像 Tag 规则】
- [ ] 普通提交：{分支名}-{短SHA}
- [ ] Git Tag：{tag}
- [ ] test 镜像追加 -test 后缀

【分支触发策略】
- [ ] build：所有分支 + MR
- [ ] package（生产镜像）：仅 master/tags/staging/release-*/hotfix-*
- [ ] package_test：所有分支（如适用）

【多模块项目】
- [ ] 每个子模块镜像名通过 variables 独立声明（禁止自动从仓库名取）
- [ ] 每个子模块有独立的 build + package job
- [ ] needs 依赖正确

【缓存配置】
- [ ] 全局 cache.paths 和 build job 内 cache.paths 和 MAVEN_OPTS 三处路径一致
- [ ] 使用绝对路径 /home/gpx/.m2/repository

输出格式：bullet points，按 [基础规范] [Maven 构建] [镜像构建] [Tag/分支策略] [缓存] 分组
```

---

## Step 3.5: sym3 平台配置在线验证

**此步骤通过浏览器自动化访问 sym3 平台和 Sentry 平台，验证线上实际配置是否正确。**

前提条件：
- Step 0 中已保存平台地址到 `.cloud-migrate.json`
- 用户的浏览器已登录 sym3 平台（Cookie 认证），如果未登录，提示用户先在浏览器中登录

从配置文件中读取地址：
```bash
P=$(pwd)
CONFIG="$P/.cloud-migrate.json"
SYM3_TEST=$(python3 -c "import json; print(json.load(open('$CONFIG'))['sym3_test_url'])")
SYM3_PROD=$(python3 -c "import json; print(json.load(open('$CONFIG'))['sym3_prod_url'])")
SENTRY_URL=$(python3 -c "import json; print(json.load(open('$CONFIG'))['sentry_url'])")
APP_NAME=$(python3 -c "import json; print(json.load(open('$CONFIG'))['app_name'])")
echo "sym3_test: $SYM3_TEST"
echo "sym3_prod: $SYM3_PROD"
echo "sentry: $SENTRY_URL"
echo "app_name: $APP_NAME"
```

### 3.5.1 sym3 平台配置验证

使用 browser MCP 工具按以下流程操作（先检测测试环境，再检测线上环境）。

对**每个环境**（test / prod）执行：

1. **导航到应用页面**：`browser_navigate` 到 `{SYM3_URL}/app/{APP_NAME}` 或平台首页搜索应用名
2. **获取页面快照**：`browser_snapshot` 确认已登录（如果看到登录页，停止并提示用户先登录）
3. **找到应用配置入口**：通过 snapshot 找到"环境变量"、"ConfigMap"、"容器配置"等 tab 或菜单项，`browser_click` 进入
4. **抓取环境变量配置**：`browser_snapshot` 读取页面上显示的所有环境变量键值对
5. **抓取 ConfigMap 配置**：找到 ConfigMap 配置页面，snapshot 读取挂载的 ConfigMap 列表和内容
6. **抓取容器资源配置**：找到容器规格页面，snapshot 读取 CPU/Memory request/limit

**注意事项：**
- sym3 平台的页面结构可能不同项目有差异，需通过 snapshot 动态识别页面元素，不要硬编码 ref
- 每次 click 后等待 2 秒再 snapshot，确保页面加载完成
- 如果页面有多个 tab（如"基本信息"、"环境变量"、"存储配置"），依次点击每个 tab 抓取
- 将抓取到的所有配置信息记录下来，传入 Agent E 做对比分析

### 3.5.2 Sentry 平台注册验证

1. **导航到 Sentry 平台**：`browser_navigate` 到 `{SENTRY_URL}`
2. **搜索应用**：在 Sentry 平台上搜索 `{APP_NAME}`，确认服务是否已注册
3. **检查 Agent 状态**：如果找到服务，进入详情页查看 Agent 是否在线、最近是否有心跳上报
4. **记录结果**：
   - 已注册 + Agent 在线 → ✅
   - 已注册 + Agent 离线 → ⚠️（可能是 Sentry Agent 配置有问题）
   - 未注册 → ❌（需要在 Sentry 平台注册并确认镜像中 Sentry 配置正确）

### Agent E — sym3 平台配置对比审计

**仅在 Step 3.5 成功抓取到平台配置后启动此代理。** 不并行，在 Agent A-D 完成后单独运行。

Prompt:
```
你是 K8s 平台配置审计专家。对比项目代码侧的配置与 sym3 平台上的实际运行时配置。

项目类型：[TOMCAT_WAR / DUBBO_JAR / SPRINGBOOT / FRONTEND]
应用名：[APP_NAME]

=== 代码侧配置（来自 Dockerfile 和 configs/）===
[粘贴 Step 1-2 收集的 Dockerfile ENV 和配置文件内容]

=== sym3 测试环境配置（来自平台抓取）===
[粘贴从 sym3 测试环境抓取的环境变量、ConfigMap、容器配置]

=== sym3 线上环境配置（来自平台抓取）===
[粘贴从 sym3 线上环境抓取的环境变量、ConfigMap、容器配置]

=== Sentry 注册状态 ===
[粘贴 Sentry 检查结果]

检查规则：

【必填环境变量 — 所有 Java 项目】
平台侧 JAVA_OPTS / JDK_JAVA_OPTIONS 必须包含：
- [ ] -Xmx 或 -XX:MaxRAMPercentage（堆内存限制）
- [ ] -XX:+UseContainerSupport（如 Dockerfile 中已设置则可选）
- [ ] -Dspring.profiles.active=sym3（Spring 项目）

【必填环境变量 — Dubbo 项目】
- [ ] SYM3_CLUSTER 已设置（用于 dubbo.registry 的 ZK 地址解析）

【环境变量一致性】
- [ ] 测试环境和线上环境的结构性变量应一致（如都有 JAVA_OPTS），只是值不同
- [ ] 不应存在测试环境有但线上环境没有的变量（可能是遗漏）

【ConfigMap 检查】
- [ ] 如果 configs/sym3/ 中有配置文件，平台上应有对应的 ConfigMap 挂载
- [ ] ConfigMap 中的内容不应包含另一个环境的地址（如测试环境的 ConfigMap 含有线上 DB 地址）
- [ ] 挂载路径应与应用期望的配置读取路径一致

【资源配置合理性】
- [ ] Memory limit 应 >= -Xmx 值 + 200MB（非堆开销）
- [ ] Memory request 与 limit 建议保持一致（避免 OOM Kill）
- [ ] CPU request 存在（不能为 0 或空）

【Sentry 集成】
- [ ] 服务在 Sentry 平台已注册
- [ ] Agent 状态正常（有最近心跳）
- [ ] 如果 Agent 离线，对比 Dockerfile 中 Sentry 配置是否正确（SENTRY_HOME 路径、agent JAR 名称）

【代码 ↔ 平台配置矛盾检测】
- [ ] Dockerfile 中 ENV 定义的值与平台环境变量是否有冲突（平台会覆盖，但需确认是有意为之）
- [ ] 代码侧配置使用的占位符（如 ${DB_URL}）在平台侧是否有对应的环境变量

输出格式：
分两个环境分别输出，每个环境按 [环境变量] [ConfigMap] [资源配置] [Sentry] [代码↔平台矛盾] 分组。
最后输出一个测试环境 vs 线上环境的差异汇总。
```

---

## Step 4: 汇总并呈现报告

将 Agent A-D（代码侧）和 Agent E（平台侧，如有）的输出合并为统一报告。

### 🔴 Critical — 阻塞上云（必须立即修复）

会导致容器启动失败、运行时异常、或安全风险的问题：
- 基础镜像错误
- JVM 参数硬编码导致 OOM
- 配置硬编码导致无法跨环境部署
- 缺少 UseContainerSupport
- Sentry Agent 缺失或版本错误
- CI 中使用了错误的 Maven Profile
- **平台侧 JAVA_OPTS 未配置堆内存**
- **平台侧 Memory limit < Xmx + 200MB（OOM Kill 风险）**
- **Sentry 未注册或 Agent 离线**
- **ConfigMap 内容包含错误环境的地址**

### 🟡 Structural — 不符合平台规范（尽快修复）

不影响运行但违反团队/平台规范的问题：
- Dockerfile ENV 分层不正确
- configs/sym3/ 目录缺失
- CI 流水线模式不匹配项目类型
- 镜像 Tag 规则不合规
- settings.xml 缺失或不规范
- .dockerignore 缺失
- **测试环境与线上环境变量结构不一致**
- **代码侧占位符在平台侧无对应变量**

### 🟢 Incremental — 可渐进优化

改善可维护性、安全性、构建效率的建议：
- 添加多阶段构建减小镜像体积
- 优化缓存配置
- 添加 healthcheck
- 配置文件整理
- CI 调试命令残留清理
- **平台资源配置精细调优**

---

每个问题须标注来源代理和对应文件路径，格式：
```
- [Agent X][文件路径] 问题描述 → 建议修复方式
```

报告末尾附加 **迁移完成度评分**：

```
┌──────────────────────────────────────────────────┐
│  云迁移就绪度评分                                   │
│                                                    │
│  Dockerfile 规范     ██████████░░  80%             │
│  JVM 容器化适配      ████████░░░░  65%             │
│  配置外部化          ██████░░░░░░  50%             │
│  CI/CD 流水线        ████████████  100%            │
│  sym3 平台配置       ████████░░░░  70%  (在线验证) │
│  Sentry 监控集成     ██████████░░  85%  (在线验证) │
│  ────────────────────────────────                  │
│  综合就绪度                         73%            │
└──────────────────────────────────────────────────┘
```

如果 Step 3.5 未执行（用户未登录平台或跳过），在评分中标注：
```
│  sym3 平台配置       ░░░░░░░░░░░░  --   (未验证) │
│  Sentry 监控集成     ░░░░░░░░░░░░  --   (未验证) │
```

**Stop condition:** 报告呈现后，询问：
> "需要我逐项修复吗？可以按维度处理：Dockerfile / JVM 参数 / 配置外部化 / CI 流水线 / sym3 平台配置。也可以生成一份修复 checklist 用于团队协作。"

不主动修改，需用户明确确认。

---

## Step 5: 平台地址管理

用户可以随时更新已保存的平台配置：
- 输入 "更新平台地址" 或 "修改 sym3 地址" 触发
- 读取现有 `.cloud-migrate.json`，展示当前值，询问用户要修改哪项
- 更新后写回文件，更新 `updated_at` 字段

批量模式中，如果多个项目使用同一套平台地址，可以从一个项目的 `.cloud-migrate.json` 复制到其他项目：
```bash
# 示例：批量设置平台地址
SOURCE="/path/to/already-configured-project/.cloud-migrate.json"
for dir in /path/to/projects/*/; do
  if [ ! -f "$dir/.cloud-migrate.json" ]; then
    cp "$SOURCE" "$dir/.cloud-migrate.json"
    # 更新 app_name 为各项目自己的名字
    APP=$(basename "$dir")
    python3 -c "
import json
with open('$dir/.cloud-migrate.json','r+') as f:
    d=json.load(f); d['app_name']='$APP'; f.seek(0); json.dump(d,f,indent=2,ensure_ascii=False); f.truncate()
"
    echo "✅ $APP"
  fi
done
```

---

## Step 6: 批量扫描模式

当用户提供父目录（如 `/path/to/all-projects/`）时，进入批量模式：

```bash
PARENT=$1  # 用户指定的父目录
echo "=== 批量扫描：$(ls -d "$PARENT"/*/ | wc -l) 个项目 ==="

for dir in "$PARENT"/*/; do
  name=$(basename "$dir")
  type="unknown"

  # 快速类型识别
  if [ -f "$dir/package.json" ] && [ ! -f "$dir/pom.xml" ]; then
    type="frontend"
  elif grep -q '<packaging>war</packaging>' "$dir/pom.xml" 2>/dev/null; then
    type="tomcat-war"
  elif grep -rl 'dubbo' "$dir/src/" >/dev/null 2>&1; then
    type="dubbo-jar"
  elif grep -q 'spring-boot' "$dir/pom.xml" 2>/dev/null; then
    type="springboot"
  elif [ -f "$dir/pom.xml" ]; then
    type="java-unknown"
  fi

  # 快速就绪度评估
  has_dockerfile=$(test -f "$dir/Dockerfile" && echo "✅" || echo "❌")
  has_ci=$(test -f "$dir/.gitlab-ci.yml" && echo "✅" || echo "❌")
  has_sym3=$(grep -q '<id>sym3</id>' "$dir/pom.xml" 2>/dev/null && echo "✅" || echo "❌")
  has_dockerignore=$(test -f "$dir/.dockerignore" && echo "✅" || echo "❌")
  has_platform_config=$(test -f "$dir/.cloud-migrate.json" && echo "✅" || echo "❌")

  # 硬编码快速检测
  hardcode_count=$(grep -rn --include="*.properties" --include="*.yml" \
    -E '(jdbc:|mysql://|zookeeper://|[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+:[0-9]+)' \
    "$dir/configs" "$dir/src/main/resources" 2>/dev/null | wc -l)

  echo "$name | $type | Dockerfile:$has_dockerfile | CI:$has_ci | sym3:$has_sym3 | platform:$has_platform_config | 硬编码:${hardcode_count}处"
done
```

批量模式输出**汇总矩阵**：

```
┌────────────────────┬──────────────┬────────────┬──────┬───────┬──────────┬──────────┐
│ 项目名              │ 类型          │ Dockerfile │ CI   │ sym3  │ 平台配置  │ 硬编码数  │
├────────────────────┼──────────────┼────────────┼──────┼───────┼──────────┼──────────┤
│ urs-mail           │ tomcat-war   │ ✅         │ ✅   │ ✅    │ ✅       │ 0        │
│ urs_sms_ng         │ tomcat-war   │ ✅         │ ✅   │ ✅    │ ✅       │ 3        │
│ user-center        │ springboot   │ ❌         │ ❌   │ ❌    │ ❌       │ 12       │
│ admin-portal       │ frontend     │ ❌         │ ✅   │ -     │ ❌       │ 2        │
│ ...                │              │            │      │       │          │          │
├────────────────────┴──────────────┴────────────┴──────┴───────┴──────────┴──────────┤
│ 总计：107 个项目 | 已就绪：23 | 部分就绪：51 | 未开始：33                              │
└────────────────────────────────────────────────────────────────────────────────────┘
```

随后按优先级排序给出建议：
1. **硬编码数最多**的项目优先处理配置外部化
2. **缺少 Dockerfile** 的项目优先创建
3. **缺少 CI** 的项目优先补充流水线
4. 已有 Dockerfile + CI 的项目做规范性审查

询问用户：
> "检测到 N 个项目。要对哪些项目做详细检查？可以按类型（如所有 tomcat-war）或按状态（如所有硬编码>5 的）筛选。"
