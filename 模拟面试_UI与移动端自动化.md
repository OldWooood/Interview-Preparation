# 🎤 模拟面试：UI 与移动端自动化

> 覆盖：Selenium/Playwright 选型与原理、元素定位与等待、框架设计、Appium/鸿蒙、移动专项与弱网。
> 对应复习文档：UI与移动端自动化测试、安卓ADB自动化。

---

## 第一轮：Web UI 自动化

### Q1 ⭐⭐ Selenium 和 Playwright 怎么选？为什么？

**参考答话术：**

> 我按三个维度选：
> ① 项目形态：现代 SPA（React/Vue）优先 Playwright——自动等待减少 Flaky，原生网络拦截便于 Mock，Trace Viewer 排障快，并行简单；旧系统/需要 IE 兼容/多语言存量团队继续 Selenium。
> ② 团队能力：前端团队用 Cypress 做组件测试最顺；Java 体系 Selenium/Grid 沉淀多。
> ③ 迁移成本：存量 Selenium 不强行重写，新用例用 Playwright 双轨并行，核心模块优先迁移。
> 我们团队实际从 Selenium 迁到 Playwright 后，同一套 300 条回归从 2.5 小时降到 50 分钟，Flaky 率从 13% 降到 1% 左右。

🔁 追问：Playwright 为什么快？→ 基于 CDP/原生协议直连浏览器，无 WebDriver 三层转发；进程外 WebSocket 通信延迟低。

🔁 追问：Playwright 的自动等待等什么？→ attached → visible → stable → receives events → enabled，任一步超时才失败。

---

### Q2 ⭐ 元素定位不到，可能有哪些原因？

**答：**

1. 页面/元素未加载完（等待不足）；
2. 元素在 iframe 内或 Shadow DOM 内；
3. 打开了新窗口/新标签；
4. 定位器脆弱：动态 ID、随机 class、文本变化；
5. 元素被遮挡、不可见、disabled；
6. 分辨率/设备差异导致布局变化；
7. 弹窗/引导页拦截。

**解决优先级：** data-testid > 语义定位（role/label/text）> CSS > XPath。

🔁 追问：XPath 和 CSS 怎么选？→ CSS 快、简洁，适合属性和层级；XPath 支持文本和轴定位（找父级/兄弟），复杂场景用；但都尽量让位于语义定位。

---

### Q3 ⭐ 等待机制有哪几种？为什么 sleep 不行？

**答：**

| 方式 | 说明 |
| --- | --- |
| 强制 sleep | 固定等待，慢且不稳 |
| 隐式等待 | 全局等元素出现，和显式混用有坑 |
| 显式等待 | 等条件满足（可点击/可见/文本出现） |
| 自动等待 | Playwright/Cypress 内置，操作前自动检查可操作性 |

`sleep` 是"赌时间"，条件等待是"等事实"；前者慢且偶发失败，后者稳定又快。

---

### Q4 PO 模式是什么？怎么落地？

**答：** Page Object 把页面元素和操作封装成类，用例只调用业务方法。落地要点：元素定位集中在类属性；操作返回下一个页面对象支持链式；断言放用例层；大页面拆组件对象（弹窗、表格），避免"超级 PO"。

```python
class LoginPage:
    USERNAME = "#username"
    SUBMIT = "#login-btn"

    def login(self, user, pwd):
        self.page.fill(self.USERNAME, user)
        self.page.fill("#password", pwd)
        self.page.click(self.SUBMIT)
        return DashboardPage(self.page)
```

🔁 追问：定位器放 PO 还是 YAML？→ 都可；YAML 便于自愈和统一维护，PO 类属性便于 IDE 跳转，团队统一即可。

---

### Q5 Playwright 的 Trace Viewer 有什么用？举例。

**答：** 记录执行全过程：每步截图、DOM 快照、网络请求、console 日志。CI 失败时下载 trace.zip，`playwright show-trace` 回放，能定位"为什么这一步失败"——比如接口 500 导致按钮没出现。我遇到过线上偶发问题，本地无法复现，靠 Trace 看到某接口返回 409 导致流程中断，直接定位。

---

### Q6 UI 用例不稳定，怎么治理？

**答：** 分类：等待（改条件等待）、定位（data-testid）、数据（隔离+自清理）、环境（容器固定浏览器）、动画（禁用动画/等 networkidle）、弹窗（全局拦截器）。工程上：失败截图+Trace、重试上限、稳定性看板、隔离区。

---

### Q7 前端经常改版，定位器天天修怎么办？

**答：**

> 治本是与前端约定 `data-testid`，关键交互元素必须有；治标是用语义定位 + 多定位器兜底 + 自愈机制；
> 流程上把 UI 测试纳入需求评审，改版前评估影响用例；用例设计上减少对 UI 细节的依赖，能用接口验证的用接口。

---

### Q8 怎么测文件上传/下载？

**答：**

```python
# 上传
page.get_by_label("上传").set_input_files("invoice.pdf")

# 下载
with page.expect_download() as info:
    page.get_by_text("导出").click()
info.value.save_as("report.xlsx")
```

覆盖：格式/大小/多文件/断点续传/恶意文件/权限/下载文件内容校验。

---

## 第二轮：移动端

### Q9 ⭐ Appium 的工作原理？

**答：**

> Appium 是 C/S 架构：测试脚本通过 W3C WebDriver 协议发命令到 Appium Server；Server 按平台转译——Android 用 UiAutomator2/Espresso，iOS 用 XCUITest（WebDriverAgent）；设备执行后返回结果。
> 2.x 之后核心与 Driver 插件化，按平台安装 `appium driver install uiautomator2`，升级更灵活。

🔁 追问：为什么 Appium 启动慢？→ 会话初始化要启动 UiAutomator server、安装辅助应用；真机还有签名和连接开销。优化：复用会话、减少 reset、用 uiautomator2。

---

### Q10 Appium 和 uiautomator2 怎么选？

| 维度 | Appium | uiautomator2 |
| --- | --- | --- |
| 平台 | Android + iOS | 仅 Android |
| 速度 | 较慢 | 快（常驻服务） |
| 生态 | 多语言、企业级 | Python、轻量 |
| 选择 | 跨平台统一 | Android 专项/效率优先 |

### Q11 移动端怎么做兼容性测试？

**答：** 机型矩阵：主流品牌 × 高中低端 × 系统版本 × 分辨率；用云真机平台（WeTest/Testin）批量跑 Monkey 和核心用例；重点看布局错乱、崩溃、性能差异、权限差异。策略：按线上用户机型分布加权，不追求全覆盖。

### Q12 ⭐ 弱网测试怎么做？关注什么？

**答：**

> 工具：Charles Throttle、Android ATC、iOS Network Link Conditioner、Playwright CDP（模拟延迟/带宽/丢包）。
> 维度：延迟、带宽、丢包、抖动、断网、WiFi/4G 切换、弱网+后台切换。
> 关注：超时与重试策略、兜底 UI（骨架屏/错误提示/重试按钮）、数据一致性（断网重连续传、订单状态）、弱网下的崩溃/ANR、重复提交。

### Q13 移动专项指标有哪些？怎么测？

| 专项 | 指标 | 工具 |
| --- | --- | --- |
| 启动 | 冷/热启动耗时 | adb am start -W |
| 流畅度 | 掉帧、卡顿 | gfxinfo、Perfetto |
| 内存 | PSS、泄漏 | dumpsys meminfo、MAT |
| CPU | 占用率、热点 | top、Perfetto |
| 流量/电量 | 前后台流量、功耗 | Battery Historian |
| 稳定性 | Crash/ANR 率 | Monkey、Fastbot |

### Q14 Monkey 和 Fastbot 的区别？

**答：** Monkey 随机事件流，简单但覆盖率低、易卡死；Fastbot 模型驱动，基于页面结构引导探索，覆盖面和崩溃检出更高，支持截图和日志收集。测试策略：Fastbot 做稳定性遍历，Monkey 做简单压测。

### Q15 鸿蒙自动化怎么做？

**答：** 官方 DevEco Testing Hypium（ArkTS）编写 UI 用例；设备用 hdc（对标 adb）；社区有 hmdriver2（Python、无侵入，API 对齐 uiautomator2）。差异：hap 安装、hilog 日志、Appium 支持有限。上架前华为要求六大专项（兼容/稳定/性能/功耗/安全/UX）。

### Q16 App 和 Web 测试有什么区别？

**答：** 安装升级（覆盖安装/卸载重装/断点升级）、权限（首次授权/拒绝/系统设置改权限）、中断（来电/短信/锁屏/切后台）、网络（弱网/代理/切换）、推送（前台/后台/被杀）、设备差异、电量流量、混合应用 WebView 切换。功能维度类似，专项更多。

---

## 第三轮：框架与工程

### Q17 ⭐ 移动端自动化框架怎么设计？

**参考话术：**

> 分层同 Web：用例层（业务语义）+ 页面层（PO）+ 组件层（等待封装/截图/断言/Oracle 数据库校验）+ 驱动层（Appium/uiautomator2 适配）。
> 特色设计：设备管理（设备池、任务队列、锁）、环境重置（清数据/重装）、失败证据（截图+录屏+logcat/hilog）、混合应用上下文切换、稳定性治理。
> 执行：本地并发或云真机，接入 CI 夜间跑核心用例。

### Q18 UI 自动化投入产出怎么保证？

**答：** 分层投入：接口为主，UI 只覆盖核心链路 10%–20%；用"收益 = 手工耗时 × 频次 - 建设与维护成本"评估；优先高价值（主流程、跨端交互、接口测不到的场景）；改版频繁的页面用轻量用例或半自动。

### Q19 怎么在 CI 里跑 UI 测试？

**答：** 浏览器/驱动固化在 Docker 镜像；按用例标签分层（冒烟每次、全量夜间）；失败上传截图/Trace/视频；执行机与用例分片并行；网络依赖用 Mock 降低波动；Flaky 隔离区不阻塞门禁。

### Q20 给你一个 App 的登录功能，怎么设计 UI 自动化用例？

**答：** 正向登录 1 条、参数化异常（空/错误密码/长度边界）、多登录方式、记住我、退出登录、多端互踢、权限拒绝后表现；断言：进入首页、Token 写入、失败提示文案；数据：独立测试账号，登录态隔离，用例之间不共享会话。

---

## 评分自检表

| 维度 | 及格 | 优秀 |
| --- | --- | --- |
| 选型 | 会用一个工具 | 三维度对比 + 迁移策略 |
| 定位 | 会写 xpath | 定位优先级 + 稳定性设计 |
| 等待 | 会用 sleep | 条件等待 + 自动等待原理 |
| 移动端 | 会跑 Appium | 原理 + 专项 + 设备管理 |
| 工程 | 本地跑通 | CI + 分层 + 稳定性指标 |
