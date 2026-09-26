
# 🐚 Shell 脚本语法大全

## 一、Shell 简介

Shell 是一种命令行解释器，用于与操作系统交互。常见的 Shell 有：
- `bash`（最常见）
- `zsh`
- `sh`
- `ksh`
- `fish`

编写脚本时通常使用 `bash`：
```bash
#!/bin/bash
# 这是一个简单的bash脚本
echo "Hello, World!"
````

---

## 二、脚本基础结构

### 1. 脚本开头（Shebang）

```bash
#!/bin/bash
```

表示该脚本由 `bash` 解释执行。

### 2. 注释

```bash
# 单行注释
: '
多行注释
'
```

### 3. 脚本执行方式

```bash
# 方式1：赋予执行权限后运行
chmod +x script.sh
./script.sh

# 方式2：直接用bash执行
bash script.sh
```

---

## 三、变量与常量

### 1. 定义变量

```bash
name="Alice"
age=18
```

### 2. 使用变量

```bash
echo "My name is $name, I am $age years old."
```

### 3. 只读变量（常量）

```bash
readonly PI=3.14
```

### 4. 删除变量

```bash
unset name
```

### 5. 特殊变量

| 变量          | 含义               |
| ----------- | ---------------- |
| `$0`        | 脚本名称             |
| `$1` ~ `$9` | 第1到第9个参数         |
| `$#`        | 参数个数             |
| `$*`        | 所有参数（作为一个整体）     |
| `$@`        | 所有参数（独立展开）       |
| `$?`        | 上一条命令的退出状态（0为成功） |
| `$$`        | 当前脚本的进程ID        |
| `$!`        | 最近一个后台任务的PID     |

---

## 四、字符串操作

```bash
str="Hello Shell"
echo ${#str}          # 字符串长度
echo ${str:0:5}       # 截取子串 -> Hello
echo ${str/Shell/Bash} # 替换 -> Hello Bash
```

---

## 五、数组

### 1. 定义与访问

```bash
arr=(apple banana cherry)
echo ${arr[1]}        # 输出 banana
```

### 2. 添加与删除

```bash
arr[3]="pear"
unset arr[1]
```

### 3. 遍历数组

```bash
for i in "${arr[@]}"; do
  echo "$i"
done
```

---

## 六、运算

### 1. 整数运算

```bash
a=5
b=3
echo $((a + b))
```

### 2. 使用 `expr`

```bash
expr 5 + 3
expr $a \* $b
```

### 3. 浮点运算（使用 `bc`）

```bash
result=$(echo "scale=2; 10 / 3" | bc)
echo $result
```

---

## 七、条件判断

### 1. 文件测试

```bash
[ -f file.txt ]  # 文件存在
[ -d dir ]       # 目录存在
[ -r file ]      # 可读
[ -w file ]      # 可写
[ -x file ]      # 可执行
```

### 2. 字符串判断

```bash
[ "$a" = "$b" ]
[ "$a" != "$b" ]
[ -z "$a" ]  # 空字符串
[ -n "$a" ]  # 非空
```

### 3. 数值比较

```bash
[ $a -eq $b ]
[ $a -ne $b ]
[ $a -lt $b ]
[ $a -le $b ]
[ $a -gt $b ]
[ $a -ge $b ]
```

### 4. 逻辑运算

```bash
[ $a -gt 10 ] && echo "a>10"
[ $a -lt 5 ] || echo "a<=5"
```

---

## 八、分支与循环

### 1. if 判断

```bash
if [ $a -gt 10 ]; then
  echo "a > 10"
elif [ $a -eq 10 ]; then
  echo "a = 10"
else
  echo "a < 10"
fi
```

### 2. case 语句

```bash
case "$1" in
  start)
    echo "Starting..."
    ;;
  stop)
    echo "Stopping..."
    ;;
  *)
    echo "Usage: $0 {start|stop}"
    ;;
esac
```

### 3. for 循环

```bash
for i in {1..5}; do
  echo "Number $i"
done
```

### 4. while 循环

```bash
count=1
while [ $count -le 5 ]; do
  echo "Count: $count"
  ((count++))
done
```

### 5. until 循环

```bash
count=1
until [ $count -gt 5 ]; do
  echo "Count: $count"
  ((count++))
done
```

---

## 九、函数

### 1. 定义函数

```bash
my_func() {
  echo "Hello from function"
}
```

### 2. 带参数函数

```bash
greet() {
  echo "Hello, $1!"
}
greet "Alice"
```

### 3. 返回值

```bash
sum() {
  return $(( $1 + $2 ))
}
sum 3 4
echo $?   # 输出 7
```

---

## 十、输入与输出

### 1. 读取输入

```bash
read -p "Enter your name: " name
echo "Hi, $name!"
```

### 2. 输出重定向

```bash
echo "Log Info" > log.txt     # 覆盖写
echo "Append Info" >> log.txt # 追加写
```

### 3. 输入重定向

```bash
while read line; do
  echo "Line: $line"
done < file.txt
```

---

## 十一、命令与进程控制

### 1. 执行命令

```bash
date
ls -l
```

### 2. 命令替换

```bash
now=$(date)
echo "Current time: $now"
```

### 3. 管道

```bash
ps aux | grep nginx
```

### 4. 后台运行

```bash
./script.sh &
```

---

## 十二、错误处理

```bash
set -e   # 遇到错误立即退出
set -u   # 使用未定义变量时报错
trap 'echo "Error occurred at line $LINENO"' ERR
```

---

## 十三、实用技巧

### 1. 判断命令是否存在

```bash
if command -v git >/dev/null 2>&1; then
  echo "git installed"
else
  echo "git not found"
fi
```

### 2. 日志函数

```bash
log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}
log "Script started"
```

### 3. 临时文件

```bash
tmpfile=$(mktemp)
echo "temp data" > "$tmpfile"
```

---

## 十四、脚本调试

```bash
bash -x script.sh  # 打印执行过程
bash -n script.sh  # 检查语法错误
set -x             # 开启调试模式
set +x             # 关闭调试模式
```

---

## 十五、示例：批量重命名文件

```bash
#!/bin/bash
# 批量给文件名添加前缀
prefix="new_"
for file in *.txt; do
  mv "$file" "${prefix}${file}"
done
```

---

## 十六、参考资料

* [GNU Bash Manual](https://www.gnu.org/software/bash/manual/)
* [Shell Script Tutorial](https://www.shellscript.sh)
* 《鸟哥的Linux私房菜》