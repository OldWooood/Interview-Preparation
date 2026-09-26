# 🖥️ UI 与移动端自动化测试（Selenium / Playwright / Appium / 鸿蒙）

> 覆盖 Web UI 自动化选型与原理、移动端自动化（Android/iOS/鸿蒙）、移动专项测试。
> 面试重点：**框架原理、元素定位、等待机制、选型对比、稳定性治理**。

---

## 一、Web UI 自动化三大框架对比（2025–2026 版）

### 1.1 核心对比表

| 维度 | Selenium 4 | Playwright | Cypress |
| --- | --- | --- | --- |
| 发布方 | 开源社区 | 微软 | Cypress.io |
| 底层协议 | W3C WebDriver（HTTP） | CDP / 原生协议，WebSocket | 运行在浏览器内部 |
| 浏览器支持 | 最广（含旧版） | Chromium/Firefox/WebKit | 仅 Chromium 系 |
| 语言 | Java/Python/JS/C#/Ruby… | JS/TS/Python/Java/.NET | JS/TS |
| 等待机制 | 手动显式等待 | **自动等待**（可操作性检查） | 内置自动重试 |
| 速度 | 较慢（HTTP 转发） | 快（直连协议） | 较快 |
| 并行 | Selenium Grid | 内置 workers | 内置（收费 Dashboard） |
| 网络拦截 | 需代理/三方库 | `page.route()` 原生 | `cy.intercept()` 原生 |
| 多标签/多窗口 | 支持 | 上下文隔离，原生支持 | 有限 |
| 移动端模拟 | 无（依赖 Appium） | 设备模拟内置 | 视口模拟 |
| 调试能力 | 日志/截图 | **Trace Viewer**（DOM 快照+视频+网络） | 时间旅行调试最强 |
| 生态成熟度 | 最成熟 | 快速增长 | 前端社区强 |
| 适用 | 兼容性广/旧系统/多语言 | 现代 SPA/新项目/CI | 前端组件测试 |

### 1.2 选型面试话术

> 新项目、现代前端（React/Vue SPA）优先 Playwright：自动等待减少 Flaky、Trace Viewer 定位快、原生网络 Mock、并行简单；
> 需要兼容 IE/旧版浏览器、团队已沉淀大量 Selenium、或用 Java 深度定制时继续 Selenium；
> Cypress 适合前端团队的组件/集成测试，但跨浏览器和跨标签能力弱。

### 1.3 Selenium 4 架构（面试常问）

```
测试代码（Client） → W3C WebDriver 协议（HTTP/JSON）
    → 浏览器驱动（ChromeDriver/GeckoDriver）
    → 浏览器（Chrome/Firefox...）
```

**Selenium 4 新特性：**

- 完整 W3C 标准；
- 相对定位器（`above/below/near/toLeftOf`）；
- 新增 `WebDriverWait` 改进、多窗口/多标签 API；
- 内置 Selenium Grid 4（支持 Docker/K8s、分布式）；
- CDP 集成（Chrome DevTools Protocol）：网络拦截、性能指标。

**Selenium 原理追问：**

1. 为什么需要 Driver？Driver 是浏览器厂商提供的"翻译器"，把 WebDriver 协议转成浏览器操作；
2. 为什么慢？每次操作都要经历协议序列化 → HTTP → Driver → 浏览器；
3. Grid 的作用？中心节点分发测试到不同浏览器/系统的 Node。

### 1.4 Playwright 架构与核心能力

```
测试代码 → Playwright Driver（Node 进程）→ 浏览器（CDP/native protocol）
                    ↑ WebSocket
```

**核心概念：**

| 概念 | 说明 |
| --- | --- |
| Browser | 浏览器实例（chromium/firefox/webkit） |
| BrowserContext | 隔离上下文，相当于隐身窗口，cookie/存储隔离 |
| Page | 页面 |
| Locator | 惰性定位器，操作时才解析，支持链式 |
| Trace | 执行轨迹：截图、DOM 快照、网络、控制台 |

**必会代码：**

```python
from playwright.sync_api import sync_playwright, expect

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    context = browser.new_context(viewport={"width": 1280, "height": 720})
    page = context.new_page()

    # 网络拦截 / Mock
    page.route("**/api/user", lambda route: route.fulfill(
        status=200, json={"name": "mock_user"}))

    page.goto("https://example.com/login")
    page.get_by_label("用户名").fill("admin")
    page.get_by_label("密码").fill("123456")
    page.get_by_role("button", name="登录").click()
    expect(page.get_by_text("欢迎")).to_be_visible()

    # 等待接口响应
    with page.expect_response("**/api/order") as resp_info:
        page.get_by_role("button", name="提交订单").click()
    assert resp_info.value.status == 200

    browser.close()
```

**Trace Viewer：**

```bash
pytest --tracing=on
playwright show-trace trace.zip
```

**移动端模拟：**

```python
iphone = p.devices["iPhone 13"]
context = browser.new_context(**iphone, locale="zh-CN", geolocation=..., permissions=["geolocation"])
```

**弱网模拟（CDP）：**

```python
client = context.new_cdp_session(page)
client.send("Network.emulateNetworkConditions", {
    "offline": False, "latency": 400,
    "downloadThroughput": 50 * 1024, "uploadThroughput": 20 * 1024})
```

### 1.5 高频操作：iframe / 多窗口 / 弹窗 / 上传下载

```python
# iframe
frame = page.frame_locator("#payment-iframe")
frame.get_by_role("button", name="确认支付").click()

# 新窗口
with page.expect_popup() as popup_info:
    page.get_by_text("查看详情").click()
popup = popup_info.value

# 原生弹窗
page.on("dialog", lambda dialog: dialog.accept())

# 文件上传
page.get_by_label("上传发票").set_input_files("invoice.pdf")

# 下载
with page.expect_download() as download_info:
    page.get_by_text("导出报表").click()
download_info.value.save_as("report.xlsx")

# 多标签页
second = context.new_page()
```

### 1.6 动态元素与不稳定治理

| 问题 | 对策 |
| --- | --- |
| 元素渲染慢 | 语义定位 + 自动等待；避免 sleep |
| 列表懒加载 | 监听接口响应 / 判断"元素数量不再增长" |
| 动画干扰 | 禁用动画、等待 `networkidle` |
| 随机弹窗 | 全局弹窗拦截器（优惠券、引导页） |
| 定位器脆弱 | 与前端约定 `data-testid` |
| 验证码 | 测试环境关闭/固定验证码/Mock 接口 |

---

## 二、移动端自动化

### 2.1 Appium 2.x

**架构变化（2.x 重点）：**

```
Appium Client (Python/Java)
      │ W3C WebDriver 协议
      ▼
Appium Server 2.x（只保留核心，功能插件化）
      ├── XCUITest Driver (iOS)
      ├── UiAutomator2 Driver (Android)
      ├── Espresso Driver (Android)
      └── 其他平台 Driver
      ▼
设备（真机/模拟器）
```

**2.x 关键点：**

- Driver 独立安装：`appium driver install uiautomator2`；
- 新增插件机制（元素定位、图像识别、报告）；
- 按平台解耦，升级更灵活；
- 支持 W3C 标准，旧版 JSONWP 参数逐步废弃。

```python
from appium import webdriver
from appium.options.android import UiAutomator2Options

options = UiAutomator2Options()
options.platform_name = "Android"
options.device_name = "Pixel_6"
options.app = "/path/app.apk"
options.automation_name = "UiAutomator2"
options.app_package = "com.example.app"
options.app_activity = ".MainActivity"
options.no_reset = False

driver = webdriver.Remote("http://127.0.0.1:4723", options=options)
driver.find_element("id", "com.example.app:id/login").click()
driver.quit()
```

**Appium 原理追问：**

1. Appium 如何操作 Android？通过 UiAutomator2 把 WebDriver 命令转换为 UiAutomator/UiAutomator2 API 调用；
2. 如何操作 iOS？XCUITest（WebDriverAgent 作为服务端）；
3. 为什么 Appium 启动慢？需要安装/启动 UiAutomator server、每次会话初始化；
4. 元素定位原理？UiAutomator dump 页面层级（XML），按属性查找。

### 2.2 移动端元素定位与等待

| 定位方式 | 说明 | 推荐度 |
| --- | --- | --- |
| accessibility id | 无障碍 ID，跨平台 | ⭐⭐⭐ |
| id | 资源 ID（Android） | ⭐⭐⭐ |
| xpath | 灵活但慢 | ⭐⭐ |
| class name | 类型 | ⭐⭐ |
| uiautomator | Android 原生选择器 | ⭐⭐ |
| iOS predicate / class chain | iOS 专用 | ⭐⭐⭐ |

```python
# 显式等待
from selenium.webdriver.support.ui import WebDriverWait

WebDriverWait(driver, 10).until(
    EC.presence_of_element_located(("id", "com.example:id/title")))

# 组合场景：滑动查找
driver.find_element(
    "android uiautomator",
    'new UiScrollable(new UiSelector().scrollable(true))'
    '.scrollIntoView(new UiSelector().text("设置"))')
```

### 2.3 uiautomator2（Python 生态高频）

```python
import uiautomator2 as u2

d = u2.connect("emulator-5554")      # 或 IP
d.app_start("com.example.app")
d(text="登录").click()
d(resourceId="com.example:id/input").set_text("admin")
d.swipe(500, 1500, 500, 500, 0.1)
d.screenshot("screen.png")
d.app_stop("com.example.app")
```

**uiautomator2 vs Appium：**

| 维度 | uiautomator2 | Appium |
| --- | --- | --- |
| 平台 | 仅 Android | Android + iOS |
| 速度 | 快（常驻服务） | 较慢 |
| 生态 | Python | 多语言、企业级 |
| 适用 | Android 专项/稳定场景 | 跨平台统一框架 |

### 2.4 鸿蒙（HarmonyOS NEXT）自动化（2025 新考点）

- 官方框架：**DevEco Testing Hypium**（ArkTS/JS），UI 测试 + 专项测试（性能、功耗、稳定性）；
- 设备连接工具：**hdc**（对标 adb），支持安装 hap、hilog 日志、shell；
- 社区方案：**hmdriver2**（Python，无侵入，API 对齐 uiautomator2）；
- Appium 对 HarmonyOS NEXT 支持有限，官方建议 Hypium。

```python
# hmdriver2 示例
from hmdriver2.driver import Driver

d = Driver("127.0.0.1:8710")
d.start_app("com.example.app")
d(text="登录").click()
d.input_text("admin")
```

**面试题：鸿蒙和 Android 自动化差异？**

- 设备命令 adb → hdc；
- 日志 logcat → hilog；
- 安装包 apk → hap；
- 官方 UI 框架 UiAutomator → Hypium；
- 生态成熟度低，需自建工具链。

### 2.5 小程序自动化

- 微信开发者工具 `miniprogram-automator`（官方）；
- 真机：微信小程序自动化 SDK / Airtest；
- 难点：多端（微信/支付宝/抖音）、WebView 混合、真机兼容。

### 2.6 移动专项测试（面试高频）

| 专项 | 指标 | 工具 |
| --- | --- | --- |
| 启动时间 | 冷启动/热启动耗时 | adb am start -W、Perfetto |
| 流畅度 | 掉帧率、卡顿次数 | gfxinfo、Perfetto、Systrace |
| 内存 | PSS、内存泄漏 | dumpsys meminfo、MAT |
| CPU | 占用率、热点函数 | top、Perfetto |
| 流量 | 前后台流量 | tcpdump、Network Profiler |
| 电量 | 功耗排行 | Battery Historian |
| 弱网 | 弱网/断网/切换 | 弱网工具、Charles、Network Link Conditioner |
| 兼容性 | 机型/系统/分辨率 | 云真机（WeTest、Testin） |
| 稳定性 | Crash/ANR 率 | Monkey、Fastbot |

**Monkey 测试：**

```bash
adb shell monkey -p com.example.app --throttle 300 \
  --pct-touch 40 --pct-motion 30 --pct-nav 20 \
  --ignore-crashes --ignore-timeouts -v 100000
```

**Fastbot（字节开源，模型驱动）** 比随机 Monkey 覆盖率更高，支持截图/崩溃收集。

### 2.7 弱网测试（面经高频）

```
弱网维度：延迟、带宽、丢包、抖动、断网、网络切换（WiFi↔4G）
工具：Charles（Throttle）、Clumsy（Windows）、ATC（Android）、
      Playwright CDP、iOS Network Link Conditioner
关注点：
  - 请求超时与重试策略
  - 弱网兜底 UI（骨架屏、错误提示、重试按钮）
  - 数据一致性（断网重连续传、订单状态）
  - 弱网下的崩溃/ANR
```

---

## 三、UI 自动化框架设计要点

### 3.1 移动端 PO 模式

```python
class BasePage:
    def __init__(self, driver):
        self.driver = driver

    def find(self, locator, timeout=10):
        return WebDriverWait(self.driver, timeout).until(
            EC.presence_of_element_located(locator))

    def click(self, locator):
        self.find(locator).click()

class LoginPage(BasePage):
    USERNAME = ("id", "com.example:id/username")
    PASSWORD = ("id", "com.example:id/password")
    SUBMIT = ("id", "com.example:id/login_btn")

    def login(self, user, pwd):
        self.find(self.USERNAME).send_keys(user)
        self.find(self.PASSWORD).send_keys(pwd)
        self.click(self.SUBMIT)
        return HomePage(self.driver)
```

### 3.2 设备管理

- 设备池：多台真机/模拟器注册表；
- 会话分配：任务队列 + 锁，避免设备抢占；
- 环境重置：`noReset=false` / 恢复出厂快照 / 清除 App 数据；
- 云端真机：本地无设备时的补充。

### 3.3 UI 自动化投入原则

> **金字塔原则：** 大量接口测试（70%）+ 少量核心 UI 用例（10%–20%）+ 单元测试（开发负责）。
> UI 只覆盖：核心业务主流程、跨端交互、视觉关键路径、接口无法验证的场景。

---

## 四、面试真题演练

### Q1：Selenium 和 Playwright 怎么选？

见 1.2。补充：存量项目迁移可渐进式（双轨并行 → 核心模块迁移 → 全量）。

### Q2：元素定位不到，可能有哪些原因？

1. 页面还没加载完（等待问题）；
2. 在 iframe 里；
3. 在新窗口；
4. 定位器写错/动态 ID；
5. 元素被遮挡/不可见；
6. Shadow DOM；
7. 分辨率/设备差异。

### Q3：Playwright 的自动等待等的是什么？

等元素：attached → visible → stable（位置稳定）→ receives events（可点击）→ enabled。每一步超时可配置。

### Q4：Appium 和 uiautomator2 的区别？

见 2.3 对比表 + 结论：跨平台选 Appium，Android 高效场景选 uiautomator2。

### Q5：如何测试一个 App 的兼容性？

机型矩阵（高中低端、主流品牌）× 系统版本 × 分辨率 × 网络环境；使用云真机平台批量执行；重点关注布局错乱、崩溃、性能差异。

### Q6：UI 自动化怎么接入 CI？

Docker 镜像固化浏览器/驱动 → Jenkins/GitLab CI 触发 → 并行执行 → 失败截图/Trace 上传 → 报告通知；夜间全量、合并请求只跑冒烟。

---

## 五、总结

| 场景 | 首选 |
| --- | --- |
| 现代 Web 新项目 | Playwright |
| 旧系统/IE/多语言 | Selenium 4 |
| 前端组件测试 | Cypress |
| Android 高效自动化 | uiautomator2 |
| 跨平台移动端 | Appium 2.x |
| 鸿蒙 NEXT | Hypium / hmdriver2 |
| 云真机 | WeTest / Testin / BrowserStack |

> UI 自动化的胜负手不是"会不会写脚本"，而是**稳定性治理 + 分层取舍 + CI 集成**。
