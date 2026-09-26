# ⚙️ CI/CD 与测试工程化（Jenkins / GitLab CI / Docker / K8s）

> 测试开发的"工程底座"：自动化测试只有接入 CI/CD 才有价值。
> 本文覆盖：流水线设计、质量门禁、覆盖率统计、容器化测试环境、K8s 执行器、环境治理。

---

## 一、CI/CD 基础概念

| 概念 | 含义 | 产出 |
| --- | --- | --- |
| CI 持续集成 | 频繁合并代码并自动构建、测试 | 每次提交快速反馈 |
| CD 持续交付 | 构建产物随时可发布到生产 | 一键发布能力 |
| CD 持续部署 | 通过验证后自动发布生产 | 无人值守发布 |
| Pipeline 流水线 | 从提交到部署的自动化流程 | 可视化阶段与结果 |

**流水线的黄金原则：**

1. 快速反馈：提交后 10 分钟内出结果（快阶段在前）；
2. 失败即阻断：门禁不通过不允许合并；
3. 可重复：同样的输入得到同样的结果（环境/依赖固定）；
4. 可追溯：每次执行关联代码版本、报告、制品。

---

## 二、流水线中的测试分层

```
Commit / MR
   │
   ├─ 1. 静态检查：Lint + SonarQube + 安全扫描（SAST）        ← 秒级~分钟
   ├─ 2. 单元测试：JUnit/pytest + 覆盖率（JaCoCo/coverage）    ← 分钟级
   ├─ 3. 构建 & 打包：Jar/镜像
   ├─ 4. 部署测试环境
   ├─ 5. 接口测试 / 契约测试 / 冒烟                            ← 分钟级
   ├─ 6. UI 测试 / E2E（核心链路）                              ← 十分钟级，可夜间全量
   ├─ 7. 性能基准（首轮可选，非阻塞）
   └─ 8. 部署预发 → 验收 → 灰度 → 生产
```

**分阶段触发策略：**

| 触发 | 执行内容 |
| --- | --- |
| 每次 push | Lint + 单测 + 单接口冒烟 |
| Merge Request | 上述 + 接口回归 + 增量覆盖率门禁 |
| 合并主干 | 全量接口 + 核心 UI + 部署测试环境 |
| 每日夜间 | 全量回归 + 性能基准 + 兼容性 |
| 发版前 | 全链路回归 + 压测 + 混沌演练 |

---

## 三、Jenkins

### 3.1 架构

```
Master（调度、配置、UI）
   ├── Agent/Node 1（Linux 构建机）
   ├── Agent/Node 2（Windows）
   └── Kubernetes Agent（按需 Pod，用完即毁）
```

### 3.2 Pipeline as Code（声明式）

```groovy
pipeline {
    agent none
    options { timeout(time: 30, unit: 'MINUTES') }
    stages {
        stage('Checkout') {
            agent any
            steps { checkout scm }
        }
        stage('Build & Unit Test') {
            agent { label 'linux' }
            steps {
                sh 'mvn -B clean test'
                junit 'target/surefire-reports/*.xml'
                jacoco execPattern: 'target/jacoco.exec'
            }
        }
        stage('API Test') {
            agent { label 'linux' }
            steps {
                sh 'pytest -m api --alluredir=allure-results -n 4'
            }
            post { always { allure includeProperties: false, results: [[path: 'allure-results']] } }
        }
        stage('Quality Gate') {
            steps {
                script {
                    def qg = waitForQualityGate()   // SonarQube 门禁
                    if (qg.status != 'OK') error "质量门禁未通过: ${qg.status}"
                }
            }
        }
        stage('Deploy') {
            when { branch 'main' }
            steps { sh 'kubectl apply -f k8s/' }
        }
    }
    post {
        failure {
            sh './scripts/notify.sh "构建失败"'
        }
    }
}
```

### 3.3 Jenkins 高频面试题

| 问题 | 回答要点 |
| --- | --- |
| Jenkins 怎么实现分布式？ | Master/Agent 架构，Label 匹配节点；K8s 插件动态创建 Pod |
| 流水线怎么区分环境？ | 参数化构建 + 凭据管理（Credentials）+ 环境变量注入 |
| 如何保证构建机环境干净？ | Docker Agent（镜像即环境）、K8s 一次性 Pod |
| 凭据怎么管理？ | Jenkins Credentials + Vault，禁止硬编码 |
| 共享库是什么？ | Shared Library：把通用流水线逻辑抽成 Groovy 库复用 |
| 如何加速？ | 缓存依赖、并行 stage、仅跑增量用例、预热镜像 |

---

## 四、GitLab CI

### 4.1 Runner 与执行器

| 执行器 | 说明 | 适用 |
| --- | --- | --- |
| shell | 直接跑在 Runner 机器 | 简单、有污染风险 |
| docker | 每个 Job 一个容器 | 最常用、环境隔离 |
| docker+machine | 自动扩缩容 | 云环境 |
| kubernetes | Job 即 Pod | 云原生首选 |

### 4.2 .gitlab-ci.yml 示例

```yaml
stages: [test, build, deploy]

variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"

cache:
  key: "$CI_COMMIT_REF_SLUG"
  paths: [.cache/pip]

unit-test:
  stage: test
  image: python:3.12
  script:
    - pip install -r requirements.txt
    - pytest tests/unit --cov=app --cov-report=xml --junitxml=report.xml
  coverage: '/TOTAL.*\s+(\d+%)$/'
  artifacts:
    when: always
    reports:
      junit: report.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

api-test:
  stage: test
  image: python:3.12
  services:
    - name: mysql:8.0
      alias: mysql
  variables:
    MYSQL_ROOT_PASSWORD: test
  script:
    - pytest tests/api -m api -n 4
  needs: [unit-test]

deploy-staging:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/app app=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  environment:
    name: staging
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

### 4.3 Jenkins vs GitLab CI

| 维度 | Jenkins | GitLab CI |
| --- | --- | --- |
| 部署 | 自建、需维护 | 与 GitLab 一体 |
| 配置 | Jenkinsfile（Groovy） | .gitlab-ci.yml（YAML） |
| 插件 | 1800+，生态最大 | 内置为主，扩展靠 Runner |
| 学习曲线 | 陡 | 缓 |
| 云原生 | 靠插件 | 原生 K8s Runner |
| 适合 | 复杂定制、多系统集成 | GitLab 用户、追求开箱即用 |

---

## 五、质量门禁（Quality Gate）

### 5.1 门禁清单

| 门禁项 | 工具 | 参考阈值 |
| --- | --- | --- |
| 静态代码扫描 | SonarQube | 0 blocker、严重问题不新增 |
| 单元测试通过率 | JUnit/pytest | 100% |
| 单元测试覆盖率 | JaCoCo/coverage | 增量 ≥ 80%，存量不下降 |
| 接口测试通过率 | 自研框架 | 挂起+失败 = 0（P0 用例） |
| 安全扫描 | SCA/SAST/DAST | 无高危 |
| 许可证合规 | 依赖扫描 | 无禁用协议 |
| 构建时长 | CI | 不超过阈值（触发优化） |

### 5.2 增量覆盖率（面试重点）

**为什么要增量？** 存量代码覆盖率低时，要求全量覆盖率会永远无法达标，增量门禁更公平。

```bash
# JaCoCo 增量报告（示例思路）
# 1. 采集本次执行覆盖率 jacoco.exec
# 2. 与基线覆盖率合并：jacoco:merge
# 3. 与 Git Diff 对比，只统计变更行的覆盖率
# 4. 生成 report，CI 解析阈值
```

**门禁策略设计要点：**

- 刚接入时先"只报告不阻断"，避免引发对抗；
- 阈值分级：核心服务严格，边缘服务宽松；
- 提供一键豁免（需 TL 审批），豁免有记录、有期限；
- 大盘展示趋势，让团队看到改进收益。

### 5.3 SonarQube 核心指标

| 指标 | 含义 |
| --- | --- |
| Reliability / Bugs | 可靠性问题 |
| Security / Vulnerabilities | 安全问题 |
| Maintainability / Code Smells | 可维护性 |
| Coverage | 覆盖率 |
| Duplications | 重复率 |
| Complexity | 复杂度 |
| Quality Gate | 门禁状态（Passed/Failed） |

---

## 六、Docker 与测试环境

### 6.1 为什么要容器化测试环境

- 环境一致：开发/测试/CI 使用同一镜像；
- 快速启动：秒级拉起数据库、Redis、Mock 服务；
- 隔离性：每个流水线独立环境，互不干扰；
- 可复现：镜像 + 数据脚本 = 确定性的环境。

### 6.2 Docker 核心命令（测试高频）

```bash
docker build -t autotest:1.0 .          # 构建镜像
docker run -d -p 8080:8080 --name app app:1.0
docker exec -it app bash                # 进入容器排查
docker logs -f --tail 100 app           # 看日志
docker ps -a                            # 查看所有容器
docker stats                            # 资源占用
docker cp app:/app/logs ./logs          # 拷贝产物
docker compose up -d                    # 启动多服务环境
docker system prune -f                  # 清理
```

### 6.3 docker-compose 测试环境示例

```yaml
version: "3.9"
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: test
      MYSQL_DATABASE: app_test
    ports: ["3306:3306"]
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      retries: 10
  redis:
    image: redis:7
    ports: ["6379:6379"]
  mock-server:
    image: wiremock/wiremock:3
    ports: ["8089:8080"]
    volumes: ["./mocks:/home/wiremock"]
  app:
    build: .
    depends_on:
      mysql: { condition: service_healthy }
```

### 6.4 测试镜像 Dockerfile 示例

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
ENV TZ=Asia/Shanghai
CMD ["pytest", "-m", "smoke", "--alluredir=/app/allure-results"]
```

> 面试点：镜像分层、构建缓存、非 root 用户、时区、健康检查。

---

## 七、Kubernetes 与测试

### 7.1 必知概念（测试视角）

| 概念 | 说明 | 测试用途 |
| --- | --- | --- |
| Pod | 最小调度单元 | 测试执行器/被测服务 |
| Deployment | 无状态工作负载 | 部署被测服务 |
| Service | 稳定访问入口 | 测试通过 Service 访问服务 |
| Ingress | 七层路由 | 暴露测试域名 |
| ConfigMap/Secret | 配置/密钥 | 注入测试配置 |
| Job/CronJob | 一次性/定时任务 | 跑自动化用例、夜间回归 |
| Namespace | 资源隔离 | 每个测试团队/需求独立环境 |
| Liveness/Readiness | 探针 | 判断服务是否可用（测试关注就绪延迟） |

### 7.2 用 K8s Job 跑自动化测试

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: api-test-${CI_COMMIT_SHA}
spec:
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: autotest
          image: registry.example.com/autotest:latest
          args: ["pytest", "-m", "api", "-n", "8"]
          env:
            - name: BASE_URL
              value: "http://app.test.svc.cluster.local"
          volumeMounts:
            - name: reports
              mountPath: /app/allure-results
      volumes:
        - name: reports
          emptyDir: {}
```

### 7.3 动态测试环境（环境治理）

**问题：** 多人共用一套测试环境，互相覆盖数据、服务频繁重启。

**方案：**

1. **按需环境**：每个 MR 流水线用 Helm/Operator 拉起独立 Namespace，跑完销毁；
2. **环境分层**：本地 → 开发(dev) → 集成(test) → 预发(staging) → 生产；
3. **数据隔离**：影子库、按用户隔离的数据分区、Mock 外部依赖；
4. **配置中心**：Nacos/Apollo 按环境隔离配置；
5. **服务依赖治理**：测试环境依赖的第三方统一走 Mock。

### 7.4 常见面试题

| 问题 | 要点 |
| --- | --- |
| 测试环境部署怎么做的？ | 镜像 + Helm/kubectl + 配置中心 + 健康检查 |
| 怎么排查服务起不来？ | 看 Pod 状态（Pending/CrashLoopBackOff）→ describe 事件 → 日志 → 探针/镜像/资源限制 |
| 怎么保证环境一致性？ | 镜像化、IaC（Helm/Terraform）、配置版本化 |
| 测试数据被污染怎么办？ | 数据隔离、幂等造数、用例自清理、定期重置 |

---

## 八、发布策略与测试配合

| 策略 | 说明 | 测试配合 |
| --- | --- | --- |
| 蓝绿发布 | 双环境切换 | 切流前在绿环境完成回归 |
| 灰度/金丝雀 | 按比例放量 | 灰度监控指标 + 自动化冒烟 |
| 滚动发布 | 逐批替换 | 观察实例健康、接口成功率 |
| 回滚 | 版本回退 | 一键回滚预案 + 数据兼容验证 |

**发布检查清单：**

- [ ] 冒烟用例通过
- [ ] 数据库变更脚本可回滚
- [ ] 配置开关（Feature Flag）就绪
- [ ] 监控告警已配置新指标
- [ ] 回滚方案经过演练

---

## 九、面试真题演练

### Q1：自动化测试怎么接入 CI/CD？

```
命令行可执行（pytest/mvn test）→ 退出码 0/1
→ CI 中配置阶段与触发条件 → 报告归档（JUnit/Allure）
→ 质量门禁判断 → 失败通知到人
```

补充：镜像化执行环境、并行加速、失败 Trace 上传、夜间全量。

### Q2：怎么让流水线又快又稳？

- 分阶段：快用例先跑，慢用例后置/夜间；
- 缓存：依赖包、构建产物、镜像层；
- 并行：用例分片、多执行机；
- 精准：只跑变更影响的用例（精准测试）；
- 稳定：隔离 Flaky 用例，门禁只信任稳定集。

### Q3：覆盖率怎么统计？

- Java：JaCoCo（on-the-fly / offline 插桩），Maven/Gradle 插件；
- Python：coverage.py / pytest-cov；
- 前端：istanbul / nyc；
- 服务端接口测试覆盖率：agent 方式采集（jacoco agent 挂载）；
- 增量覆盖率 = 变更行覆盖 / 变更行总数。

### Q4：CI 里 UI 测试挂了但本地能过，怎么排查？

1. 环境差异：浏览器/驱动版本、分辨率、时区、依赖服务 Mock；
2. 资源差异：内存、CPU、无头模式渲染差异；
3. 并发冲突：共享数据、端口、登录态；
4. 超时差异：CI 更慢，等待不足；
5. 用 Trace/截图/视频定位（Playwright Trace 是首选）。

### Q5：什么是不可变基础设施？和测试有什么关系？

镜像/环境部署后不修改，变更靠重新构建。测试因此获得确定性：同镜像同结果，问题可复现；环境漂移问题消失。

---

## 十、总结

> **工程化能力 = 自动化能力 × CI/CD 集成度 × 环境治理水平。**
> 面试时能画出一条"提交 → 门禁 → 部署 → 回归 → 报告 → 通知"的完整流水线，并说清每步的卡点与优化，就超过了大多数候选人。
