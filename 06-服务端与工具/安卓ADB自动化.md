# 📱 Android ADB 与移动端自动化完全指南

> 从 ADB 原理、命令大全到自动化框架封装、性能采集、Monkey/Fastbot、多设备 CI 与问题排查。
> 适合移动端测试、测试开发面试备考。

---

## 一、ADB 是什么

**ADB（Android Debug Bridge）** 是 Android SDK 提供的客户端-服务端调试工具：

```
PC（adb client） ──USB/TCP──→ adb server（PC 后台，5037 端口）──→ adbd（设备端守护进程）
```

- `adb devices` 显示 `unauthorized` → 设备上未确认调试授权；
- `offline` → adbd 异常，重启 adb 服务或重插设备；
- 支持 USB 与无线（`adb pair` + `adb connect`）两种连接方式。

**环境准备：**

```bash
# macOS 安装（三种方式任选）
brew install --cask android-platform-tools
# 或下载 platform-tools 解压后加入 PATH
export PATH=$PATH:$HOME/Library/Android/sdk/platform-tools

adb version
adb devices -l
```

---

## 二、命令大全（按类别）

### 2.1 设备管理

```bash
adb devices -l                      # 列表（含型号）
adb -s <serial> shell ...           # 指定设备
adb connect 192.168.1.10:5555       # 无线连接
adb disconnect 192.168.1.10:5555
adb kill-server && adb start-server # 重启服务
adb reboot                          # 重启设备
adb root / adb remount              # root（测试机）
```

### 2.2 应用管理

```bash
adb install -r app.apk              # 覆盖安装
adb install -t app.apk              # 允许 test 包
adb uninstall com.example.app
adb shell pm list packages | grep example
adb shell pm clear com.example.app  # 清数据（等价"恢复初始状态"）
adb shell am start -n com.example.app/.MainActivity
adb shell am force-stop com.example.app
adb shell am start -W -n com.example.app/.MainActivity   # 输出启动耗时
adb shell dumpsys package com.example.app | grep versionName
```

### 2.3 输入与手势

```bash
adb shell input tap 500 1500
adb shell input swipe 500 1800 500 500 300     # 上滑 300ms
adb shell input swipe 200 500 800 500 2000     # 长按
adb shell input text "hello"                   # 不支持中文/空格（空格用 %s）
adb shell input keyevent 4                     # 返回
adb shell input keyevent 3                     # Home
adb shell input keyevent 26                    # 电源
adb shell input keyevent 66                    # 回车
```

**输入中文的三种方案：**

1. 安装 ADBKeyboard 输入法，广播输入：`adb shell am broadcast -a ADB_INPUT_TEXT --es msg "中文"`；
2. 用 uiautomator2 的 `set_text`（内部走剪贴板/IME）；
3. Appium 的 `send_keys`（底层处理编码）。

### 2.4 屏幕与文件

```bash
adb exec-out screencap -p > screen.png         # 截图（推荐 exec-out）
adb shell screenrecord /sdcard/demo.mp4        # 录屏（默认 3 分钟上限）
adb pull /sdcard/demo.mp4 .
adb push local.txt /sdcard/
adb shell ls -l /sdcard/
```

### 2.5 日志

```bash
adb logcat -c                                  # 清空
adb logcat -v time | grep -i "exception"
adb logcat --pid=$(adb shell pidof -s com.example.app)
adb logcat -b crash                            # crash 缓冲
adb shell dumpsys dropbox --print | head       # 系统崩溃记录
```

### 2.6 性能采集

```bash
# 启动耗时
adb shell am start -W -n com.example.app/.MainActivity
# 输出：TotalTime / WaitTime

# 内存
adb shell dumpsys meminfo com.example.app | grep -E "TOTAL|Native|Dalvik"

# CPU
adb shell top -n 1 -b | grep com.example.app

# 帧率/卡顿
adb shell dumpsys gfxinfo com.example.app

# 电量
adb shell dumpsys batterystats com.example.app > battery.txt
# 配合 Battery Historian 可视化

# 流量（需 uid）
adb shell dumpsys package com.example.app | grep userId=
adb shell cat /proc/net/xt_qtaguid/stats | grep <uid>
```

### 2.7 网络与系统

```bash
adb shell dumpsys connectivity
adb shell settings put global http_proxy 10.0.0.1:8888   # 设代理（抓包）
adb shell settings put global http_proxy :0              # 取消
adb shell dumpsys window | grep mCurrentFocus            # 当前 Activity
adb shell dumpsys activity activities | grep mResumedActivity
adb shell getprop ro.build.version.release               # 系统版本
adb shell wm size / wm density                           # 分辨率/密度
adb shell svc wifi disable / enable
```

### 2.8 dumpsys 常用子命令

| 命令 | 用途 |
| --- | --- |
| `dumpsys activity` | Activity 栈、启动信息 |
| `dumpsys window` | 当前窗口/焦点 |
| `dumpsys meminfo` | 内存 |
| `dumpsys gfxinfo` | 渲染帧数据 |
| `dumpsys battery` | 电池状态 |
| `dumpsys package` | 应用信息、权限 |
| `dumpsys notification` | 通知栏 |
| `dumpsys dropbox` | 崩溃日志 |

---

## 三、UI 自动化方案对比（移动端）

| 方案 | 平台 | 侵入性 | 速度 | 能力 | 适用 |
| --- | --- | --- | --- | --- | --- |
| ADB 原生 | Android | 无 | 中 | 基础操作 + XML 定位 | 简单脚本、应急 |
| uiautomator2 | Android | 低（装 Agent） | 快 | 丰富 API、常驻服务 | Android 为主 |
| Appium 2.x | Android/iOS | 低 | 较慢 | 跨平台、生态成熟 | 跨平台框架 |
| Airtest | Android/iOS/游戏 | 低 | 中 | 图像识别 + Poco | 游戏、相机/视频 |
| Hypium | HarmonyOS | 低 | 中 | 官方 UI 框架 | 鸿蒙应用 |
| hmdriver2 | HarmonyOS NEXT | 无 | 快 | Python、API 对齐 uiautomator2 | 鸿蒙社区方案 |

**选择建议：** 跨平台选 Appium；Android 效率优先选 uiautomator2；鸿蒙用 Hypium；图像/游戏场景 Airtest 兜底。

---

## 四、基于 ADB 的 Python 封装（升级版）

```python
import subprocess, time, re
from xml.etree import ElementTree as ET

class Adb:
    def __init__(self, serial=None):
        self.serial = serial
        self.prefix = f"adb -s {serial} " if serial else "adb "

    def run(self, cmd, timeout=30):
        result = subprocess.run(
            f"{self.prefix}{cmd}", shell=True, capture_output=True,
            text=True, timeout=timeout)
        return result.stdout.strip()

    # 基础操作
    def tap(self, x, y):        self.run(f"shell input tap {x} {y}")
    def swipe(self, *args):     self.run(f"shell input swipe {' '.join(map(str, args))}")
    def key(self, code):        self.run(f"shell input keyevent {code}")
    def text(self, s):          self.run(f"shell input text {s.replace(' ', '%s')}")
    def screenshot(self, path="screen.png"):
        subprocess.run(f"{self.prefix}exec-out screencap -p > {path}",
                       shell=True, check=True)

    # Activity / 日志
    def current_activity(self):
        out = self.run("shell dumpsys window | grep mCurrentFocus")
        m = re.search(r"(\S+)/(\S+)\s*\}", out)
        return m.group(0) if m else out

    def logcat(self, keyword=None):
        cmd = "logcat -d -v time"
        if keyword:
            cmd += f" | grep -i {keyword}"
        return self.run(cmd)

    # UI 树
    def dump_ui(self):
        self.run("shell uiautomator dump /sdcard/view.xml")
        self.run("pull /sdcard/view.xml .", timeout=60)
        return ET.parse("view.xml").getroot()

    # 元素查找（含滚动查找）
    def find(self, by, value, timeout=10):
        deadline = time.time() + timeout
        while time.time() < deadline:
            root = self.dump_ui()
            for node in root.iter("node"):
                if node.attrib.get(by, "") == value and \
                   node.attrib.get("clickable") == "true":
                    return node
            time.sleep(1)
        raise TimeoutError(f"未找到 {by}={value}")

    @staticmethod
    def bounds_center(bounds):
        x1, y1, x2, y2 = map(int, re.findall(r"\d+", bounds))
        return (x1 + x2) // 2, (y1 + y2) // 2

    def click(self, by, value, timeout=10):
        node = self.find(by, value, timeout)
        self.tap(*self.bounds_center(node.attrib["bounds"]))

    def scroll_find(self, by, value, max_swipes=5):
        for _ in range(max_swipes):
            try:
                return self.find(by, value, timeout=2)
            except TimeoutError:
                self.swipe(500, 1600, 500, 600, 300)
                time.sleep(1)
        raise TimeoutError(f"滚动 {max_swipes} 次仍未找到 {value}")

# 使用示例
if __name__ == "__main__":
    d = Adb()
    d.run("shell am start -n com.example.app/.MainActivity")
    time.sleep(2)
    d.click("resource-id", "com.example.app:id/login_btn")
    d.click("text", "用户名")
    d.text("autotest_user")
    d.screenshot("after_login.png")
    print("当前 Activity:", d.current_activity())
    print("最近异常日志:", d.logcat("exception")[:500])
```

**XML 节点常用属性：**

| 属性 | 说明 |
| --- | --- |
| text / content-desc | 文本 / 无障碍描述（推荐） |
| resource-id | 资源 ID（稳定） |
| class | 控件类型 |
| bounds | 坐标范围 `[x1,y1][x2,y2]` |
| clickable / enabled / focused | 是否可点击/可用/焦点 |
| package | 所属应用 |

---

## 五、OCR 与图像识别兜底

适合场景：游戏、地图、相机、Canvas 渲染等无控件树的页面。

```python
# OCR 点击（PaddleOCR）
def click_by_text(ocr, screenshot_path, keyword, threshold=0.8):
    results = ocr.ocr(screenshot_path, cls=True)
    for line in results:
        for box, (text, confidence) in line:
            if keyword in text and confidence > threshold:
                x = int((box[0][0] + box[2][0]) / 2)
                y = int((box[0][1] + box[2][1]) / 2)
                return x, y
    return None

# 模板匹配（OpenCV）
import cv2, numpy as np
def match_template(screen_path, template_path, threshold=0.8):
    screen = cv2.imread(screen_path)
    template = cv2.imread(template_path)
    result = cv2.matchTemplate(screen, template, cv2.TM_CCOEFF_NORMED)
    loc = np.where(result >= threshold)
    if len(loc[0]) == 0:
        return None
    y, x = loc[0][0], loc[1][0]
    h, w = template.shape[:2]
    return x + w // 2, y + h // 2
```

**策略：** 优先控件定位（稳定）→ OCR（文本类）→ 模板匹配（图像类）→ 兜底人工；多方案融合提高成功率。

---

## 六、移动端性能采集脚本

```bash
#!/usr/bin/env bash
# 启动耗时统计（冷启动 5 次取平均）
APP="com.example.app/.MainActivity"
adb shell am force-stop "${APP%%/*}"
for i in $(seq 1 5); do
  adb shell am start -W -n "$APP" | grep TotalTime
  sleep 2
done

# 内存采样（每 5 秒一次，持续 1 分钟）
for i in $(seq 1 12); do
  adb shell dumpsys meminfo com.example.app | grep TOTAL
  sleep 5
done
```

**指标与工具对照：**

| 专项 | 命令/工具 |
| --- | --- |
| 启动 | `am start -W`、Perfetto |
| 流畅度 | `dumpsys gfxinfo`、Perfetto/Systrace |
| 内存 | `dumpsys meminfo`、MAT |
| CPU | `top`、Perfetto |
| 电量 | `dumpsys batterystats`、Battery Historian |
| 流量 | `/proc/net/xt_qtaguid/stats`、tcpdump |
| 稳定性 | Monkey、Fastbot + `logcat/ dropbox` |

---

## 七、Monkey 与 Fastbot

```bash
# 随机 Monkey：10 万次事件，忽略崩溃继续
adb shell monkey -p com.example.app --throttle 300 \
  --pct-touch 40 --pct-motion 30 --pct-nav 20 \
  --ignore-crashes --ignore-timeouts -v 100000
```

| 工具 | 原理 | 优点 | 缺点 |
| --- | --- | --- | --- |
| Monkey | 随机事件流 | 简单、系统自带 | 覆盖率低、易卡死 |
| Fastbot | 模型驱动（页面结构引导） | 覆盖率高、崩溃收集 | 需接入 SDK/工具 |

**稳定性判定：** Crash 率、ANR 率、卡死次数、异常日志聚类。

---

## 八、多设备与 CI

```python
# 多设备并行执行示意
import concurrent.futures

def run_on_device(serial, case):
    d = Adb(serial)
    return case(d)

with concurrent.futures.ThreadPoolExecutor(max_workers=4) as pool:
    futures = [pool.submit(run_on_device, s, run_login_case)
               for s in ["emulator-5554", "emulator-5556", "192.168.1.10:5555"]]
    for f in concurrent.futures.as_completed(futures):
        print(f.result())
```

**CI 集成要点：**

- 镜像预装 platform-tools 与 App；
- 设备池管理（任务队列 + 锁，避免抢占）；
- 每次用例前 `pm clear` / 重装保证环境干净；
- 失败收集：截图 + logcat + dropbox + 录屏；
- 云端真机（WeTest/Testin/BrowserStack）作为补充。

---

## 九、常见问题排查

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| `unauthorized` | 未授权调试 | 设备点"允许"，勾选始终允许 |
| `offline` | adbd 异常 | `adb kill-server` 重连/重插 |
| 设备不识别 | 驱动/线缆/端口 | 换线换口、检查 USB 调试 |
| `5037 端口占用` | 其他进程占用 | 查杀占用进程后重启 server |
| `input text` 无效 | 中文/空格/特殊字符 | ADBKeyboard、uiautomator2 |
| UI dump 慢/为空 | 页面 WebView/游戏/动画 | 用 OCR/模板、加等待重试 |
| 点击无效 | 坐标被遮挡/不可点击 | 用控件 bounds 中心，检查 enabled |
| 无线连接掉线 | 网络波动/休眠 | 保持唤醒、脚本重连 |
| 截图黑屏 | DRM/安全页面 | `screencap` 替代方案或允许截图 |

---

## 十、完整项目结构建议

```
mobile-autotest/
├── config/
│   ├── devices.yaml          # 设备池
│   └── env.yaml              # 环境与账号
├── core/
│   ├── adb.py                # ADB 封装
│   ├── driver.py             # uiautomator2/Appium 适配
│   ├── element.py            # 定位/等待/滚动查找
│   └── perf.py               # 性能采集
├── pages/                    # PO 页面对象
├── testcases/                # 用例
├── data/                     # 测试数据
├── reports/                  # 截图/日志/报告
├── conftest.py
└── pytest.ini
```

---

## 十一、面试问答

**Q1：ADB 的工作原理？**
> C/S 架构：PC 上的 client 通过 adb server（5037 端口）与设备端 adbd 通信；server 管理与设备的连接，client 发送命令。

**Q2：ADB 自动化能替代 Appium 吗？**
> 不能完全替代。ADB 适合基础操作、性能采集与应急脚本；Appium 提供跨平台、元素定位与生态。实际项目常用 uiautomator2/Appium 为主，ADB 作为补充（性能、设备管理、异常恢复）。

**Q3：怎么统计 App 启动时间？**
> `adb shell am start -W` 看 TotalTime；冷启动前先 force-stop 或清缓存；多次采样取平均/中位数；对比基线版本。

**Q4：怎么判断页面卡顿？**
> `dumpsys gfxinfo` 看掉帧率；Perfetto/Systrace 抓取具体耗时；结合体验指标（滑动响应、首屏时间）。

**Q5：多设备怎么管理？**
> 设备池 + 任务队列 + 锁；`adb -s` 指定设备；CI 中并行执行；云端真机补充机型覆盖。

---

## 十二、总结

> ADB 是移动端自动化的"底层瑞士军刀"：设备管理、操作模拟、日志、性能、稳定性一网打尽。
> 自动化策略：**控件定位为主（uiautomator2/Appium）、ADB 为底（设备与性能）、OCR/CV 兜底（无控件树场景）**。
