## 一、核心思路

ADB 是 Android Debug Bridge，可以通过命令行远程控制设备：

| 操作       | adb 命令                                                                      |
| -------- | --------------------------------------------------------------------------- |
| 点击       | `adb shell input tap x y`                                                   |
| 滑动       | `adb shell input swipe x1 y1 x2 y2 [duration]`                              |
| 输入文字     | `adb shell input text "hello"`                                              |
| 返回       | `adb shell input keyevent 4`                                                |
| 截图       | `adb exec-out screencap -p > screen.png`                                    |
| 获取当前活动窗口 | `adb shell dumpsys window | grep mCurrentFocus` |
| 查找 UI 元素 | `adb shell uiautomator dump /sdcard/view.xml` + `adb pull /sdcard/view.xml` |

---

## 二、Python 实现举例

以下代码只使用 `subprocess` 调用 ADB，实现简单的自动化场景：
比如：**启动应用 → 点击按钮 → 输入文字 → 截图保存**

```python
import subprocess
import time
from xml.etree import ElementTree as ET

# 统一的 adb 执行方法
def adb(cmd: str) -> str:
    """执行adb命令并返回结果"""
    full_cmd = f"adb {cmd}"
    result = subprocess.run(full_cmd, shell=True, capture_output=True, text=True)
    return result.stdout.strip()

# 点击坐标
def tap(x: int, y: int):
    adb(f"shell input tap {x} {y}")

# 滑动
def swipe(x1: int, y1: int, x2: int, y2: int, duration_ms=300):
    adb(f"shell input swipe {x1} {y1} {x2} {y2} {duration_ms}")

# 输入文字
def input_text(text: str):
    # 注意需要转义空格
    safe_text = text.replace(" ", "%s")
    adb(f"shell input text {safe_text}")

# 截图
def screenshot(filename="screen.png"):
    adb(f"exec-out screencap -p > {filename}")
    print(f"截图已保存为 {filename}")

# 获取当前Activity
def get_current_activity():
    return adb("shell dumpsys window | grep mCurrentFocus")

# 解析UI层级
def dump_ui_tree():
    adb("shell uiautomator dump /sdcard/view.xml")
    adb("pull /sdcard/view.xml .")
    tree = ET.parse("view.xml")
    root = tree.getroot()
    return root

# 示例：打开微信 -> 点击搜索框 -> 输入文字 -> 截图
if __name__ == "__main__":
    # 启动应用
    adb("shell am start -n com.tencent.mm/.ui.LauncherUI")
    time.sleep(3)

    # 点击右上角搜索按钮（假设坐标）
    tap(1000, 180)
    time.sleep(1)

    # 输入文字
    input_text("ChatGPT")
    time.sleep(1)

    # 截图
    screenshot()

    print("当前Activity：", get_current_activity())
```

---

## 三、延伸：查找控件位置（可视化点击）

如果你想「根据控件文字/ID 自动点击」，可以这样：

```python
def find_element_bounds(text_keyword: str):
    root = dump_ui_tree()
    for node in root.iter("node"):
        text = node.attrib.get("text", "")
        if text_keyword in text:
            bounds = node.attrib["bounds"]
            # 格式：[x1,y1][x2,y2]
            parts = bounds.replace("[", "").replace("]", ",").split(",")
            x1, y1, x2, y2 = map(int, parts[:4])
            # 点击中心点
            x, y = (x1 + x2) // 2, (y1 + y2) // 2
            print(f"找到元素 '{text_keyword}'，坐标：({x},{y})")
            return x, y
    print(f"未找到包含 '{text_keyword}' 的控件。")
    return None

# 示例：自动点击包含“登录”的按钮
pos = find_element_bounds("登录")
if pos:
    tap(*pos)
```
---

## 四、延伸：OCR和CV

如果你想「根据控件文字/ID 自动点击」，可以这样：

```python
import cv2
import numpy as np
from paddleocr import PaddleOCR
from PIL import Image

# 1. OCR 识别点击文字
def click_by_text(keyword: str, screenshot_path="screen.png", threshold=0.8):
    """
    根据文字查找位置并点击
    """
    results = ocr.ocr(screenshot_path, cls=True)
    for line in results:
        for (_, (text, confidence)) in line:
            if keyword in text and confidence > threshold:
                # PaddleOCR返回的坐标是四个点，取中心点
                points = _[0]
                x = int((points[0][0] + points[2][0]) / 2)
                y = int((points[0][1] + points[2][1]) / 2)
                tap(x, y)
                print(f"识别到文字「{text}」，置信度 {confidence:.2f}")
                return True
    print(f"未找到文字：{keyword}")
    return False

# 2. 模板匹配（OpenCV）
def click_by_template(template_path: str, screenshot_path="screen.png", threshold=0.8):
    """
    根据模板图片匹配目标并点击
    """
    screen = cv2.imread(screenshot_path)
    template = cv2.imread(template_path)

    if screen is None or template is None:
        print("❌ 图片加载失败，请检查路径。")
        return False

    result = cv2.matchTemplate(screen, template, cv2.TM_CCOEFF_NORMED)
    loc = np.where(result >= threshold)

    if len(loc[0]) == 0:
        print("未找到匹配的模板。")
        return False

    # 取第一个匹配结果
    y, x = loc[0][0], loc[1][0]
    h, w = template.shape[:2]
    cx, cy = x + w // 2, y + h // 2
    tap(cx, cy)
    print(f"匹配成功，点击模板中心点 ({cx}, {cy})，相似度 {result[y, x]:.2f}")
    return True
```
---

## 五、可实现的自动化类型

✅ 能做的：

* 启动 App、退出、切换
* 模拟手势操作
* 自动截图、OCR识别
* 基于 XML 的简单元素定位

❌ 不适合的：

* 复杂的 UI 层级深度定位
* 需要跨应用的输入法/安全键盘
* 精确同步的性能测试
---

## 六、进一步优化建议

| 功能   | 可扩展方向                                |
| ---- | ------------------------------------ |
| 元素识别 | PaddleOCR + OpenCV 多模态融合（优先OCR，备选模板） |
| 自动重试 | 如果未识别到目标，自动重试多次                      |
| 日志系统 | 记录每次操作与截图                            |
| 多设备  | `adb -s <device_id>` 适配多台设备          |
