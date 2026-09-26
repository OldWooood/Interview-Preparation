# 🐚 Shell 脚本完全指南（语法 + 实战 + 面试题）

> 从语法基础到测试工程师最常用的四类脚本：日志分析、接口巡检、测试数据准备、部署发布。
> 附 30+ 可复制示例、避坑清单与面试手撕题。

---

## 一、Shell 简介与最佳实践

常见 Shell：`bash`（最常用）、`zsh`、`sh`、`ksh`、`fish`。脚本统一用 bash：

```bash
#!/usr/bin/env bash
# 推荐开头：严格模式（详见第十二节）
set -euo pipefail
```

**执行方式：**

```bash
chmod +x script.sh && ./script.sh   # 赋权执行
bash script.sh                      # 解释器执行
source script.sh                    # 当前 shell 执行（变量生效）
```

---

## 二、变量

### 2.1 定义与使用

```bash
name="Alice"; age=18
echo "My name is $name, I am $age years old."
readonly PI=3.14     # 只读
unset name           # 删除
```

> 规则：赋值等号两边不能有空格；引用变量加双引号 `"$var"`（防单词拆分）。

### 2.2 特殊变量

| 变量 | 含义 |
| --- | --- |
| `$0` | 脚本名 |
| `$1`~`$9`、`${10}` | 位置参数 |
| `$#` | 参数个数 |
| `$*` / `$@` | 所有参数（整体 / 独立） |
| `$?` | 上条命令退出码（0 成功） |
| `$$` / `$!` | 当前进程 PID / 最近后台任务 PID |

### 2.3 参数扩展（高级）

```bash
echo "${var:-默认值}"      # 未定义或空 → 默认值
echo "${var:=默认值}"      # 未定义 → 赋值并返回
echo "${var:?必须传参}"    # 未定义 → 报错退出
echo "${var:+有值时返回}"  # 有值 → 返回替代值
file="/path/to/report.tar.gz"
echo "${file#*/}"          # 删最短前缀 → path/to/report.tar.gz
echo "${file##*/}"         # 删最长前缀 → report.tar.gz（取文件名）
echo "${file%.*}"          # 删最短后缀 → /path/to/report.tar
echo "${file%%.*}"         # 删最长后缀 → /path/to/report
echo "${name^^} ${name,,}" # 全大写 / 全小写
```

---

## 三、字符串与数组

```bash
str="Hello Shell"
echo ${#str}             # 长度 11
echo ${str:0:5}          # 截取 Hello
echo ${str/Shell/Bash}   # 替换 Hello Bash
echo ${str//l/L}         # 全部替换

arr=(apple banana cherry)
echo "${arr[1]}"         # banana
echo "${#arr[@]}"        # 元素个数
arr+=(pear)              # 追加
unset 'arr[1]'           # 删除元素
for i in "${arr[@]}"; do echo "$i"; done     # 正确遍历（带引号）

# 关联数组（bash 4+）
declare -A map=([name]="tom" [age]=18)
echo "${map[name]}"
for k in "${!map[@]}"; do echo "$k=${map[$k]}"; done
```

---

## 四、运算

```bash
a=5; b=3
echo $((a + b))                       # 8
((a++))                                # 自增
echo $((a > 4 ? 1 : 0))               # 三元
result=$(echo "scale=2; 10 / 3" | bc) # 浮点 3.33
```

---

## 五、条件判断

```bash
# 文件
[ -f file ] && echo 存在           # 普通文件
[ -d dir ]                         # 目录
[ -r/-w/-x file ]                  # 读/写/执行
[ -s file ]                        # 非空文件
[ -z "$a" ] / [ -n "$a" ]          # 空 / 非空字符串

# 数值
[ $a -eq $b ]  # =   -ne #  -lt <  -le <=  -gt >  -ge >=

# 字符串
[ "$a" = "$b" ] / [ "$a" != "$b" ]

# [[ ]] 增强（推荐）：支持正则与逻辑组合
[[ "$name" =~ ^[a-z]+$ ]] && echo 合法
[[ $a -gt 10 && $b -lt 5 ]] && echo ok
```

---

## 六、分支与循环

```bash
if [[ $1 == "start" ]]; then
  echo "starting..."
elif [[ $1 == "stop" ]]; then
  echo "stopping..."
else
  echo "Usage: $0 {start|stop}"; exit 1
fi

case "$1" in
  start|up)   echo "start" ;;
  stop|down)  echo "stop" ;;
  *)          echo "unknown"; exit 1 ;;
esac

for i in {1..5}; do echo "$i"; done
for f in *.log; do echo "处理 $f"; done
while read -r line; do echo "$line"; done < file.txt
for ((i=0; i<3; i++)); do echo "$i"; done

count=1
until [[ $count -gt 5 ]]; do ((count++)); done

# break / continue
for i in {1..10}; do
  [[ $i -eq 3 ]] && continue
  [[ $i -eq 8 ]] && break
  echo "$i"
done
```

---

## 七、函数

```bash
log() {
  local level="$1"; shift              # local 防污染全局
  echo "[$(date '+%F %T')] [$level] $*"
}

# 返回值 vs 输出
add() { echo $(( $1 + $2 )); }          # 通过 stdout 返回
result=$(add 3 4)

check_url() {                           # 通过 exit code 返回
  curl -sf --max-time 5 "$1" >/dev/null
}

if check_url "http://app/health"; then
  log INFO "健康检查通过"
else
  log ERROR "健康检查失败"
fi
```

---

## 八、输入输出与重定向

```bash
read -rp "Enter name: " name
echo "Hi, $name"

cmd > out.log          # 覆盖 stdout
cmd >> out.log         # 追加
cmd 2>&1               # stderr 合并到 stdout
cmd &> all.log         # 全部重定向
cmd > /dev/null 2>&1   # 丢弃输出（只看退出码）
while read -r line; do echo "$line"; done < file.txt   # 读文件
```

---

## 九、文本处理实战（测试高频）

```bash
# 1. 统计访问 IP Top10
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# 2. 统计 5xx 数量及 Top 接口
awk '$9 ~ /^5/ {c++} END {print "5xx:", c}' access.log
awk '$9 ~ /^5/ {print $7}' access.log | sort | uniq -c | sort -rn | head

# 3. 按时间段过滤日志
awk '$0 >= "[2026-09-01 10:00" && $0 <= "[2026-09-01 10:30"' app.log

# 4. 提取异常类名并聚类
grep -oE '[A-Za-z.]+Exception' app.log | sort | uniq -c | sort -rn | head

# 5. 关联 traceId 取完整上下文
grep -n "trace-abc123" app.log -A 5 -B 5

# 6. 统计接口 P95 耗时（从日志提取毫秒数）
grep "cost=" app.log | grep -oE 'cost=[0-9]+' | cut -d= -f2 \
  | sort -n | awk '{a[NR]=$1} END {print "P95:", a[int(NR*0.95)]}'

# 7. 找出重复订单号（数据巡检）
awk '{print $3}' orders.txt | sort | uniq -d

# 8. 行列转换
awk '{for(i=1;i<=NF;i++) print $i}' file.txt

# 9. 逗号分隔去重排序
tr ',' '\n' < tags.txt | sort -u

# 10. 批量替换文件内容
sed -i.bak 's/old-domain.com/new-domain.com/g' *.conf

# 11. 删除空行和注释行
sed '/^\s*#/d; /^\s*$/d' config.ini

# 12. 统计文件行数 Top10
wc -l *.log | sort -rn | head -11

# 13. 对比两个文件的差异（对账）
comm -3 <(sort a.txt) <(sort b.txt)

# 14. JSON 字段提取（jq）
curl -s http://api/health | jq -r '.data.status'

# 15. 按大小找大文件
du -ah . | sort -rh | head -20
```

---

## 十、文件与目录操作

```bash
# 查找并处理（7 天前日志）
find /var/log/app -name "*.log" -mtime +7 -exec gzip {} \;
find . -name "*.tmp" -mtime +1 -delete
find . -type f -size +100M
find . -name "*.py" | xargs grep -l "TODO"

# 安全删除（防误删）
find /data/tmp -maxdepth 1 -name "autotest_*" -print -delete

# 批量重命名
for f in *.txt; do mv "$f" "${f%.txt}.log"; done

# 目录大小
du -sh /var/log/* | sort -rh | head
```

---

## 十一、进程与并发

```bash
# 后台任务
./long_task.sh &
pid=$!
wait "$pid"                 # 等待结束
kill -9 "$pid"              # 强杀

# 并发执行（xargs -P）
cat urls.txt | xargs -P 8 -I {} curl -s -o /dev/null -w "%{http_code} {}\n" {}

# 监控并重启（简易守护）
while true; do
  if ! pgrep -f "app.jar" >/dev/null; then
    nohup java -jar app.jar > app.log 2>&1 &
    echo "[$(date)] restarted" >> watchdog.log
  fi
  sleep 10
done
```

---

## 十二、严格模式与错误处理

```bash
set -e          # 命令失败立即退出
set -u          # 使用未定义变量报错
set -o pipefail # 管道任一环节失败则整体失败

trap 'echo "[ERROR] line $LINENO, exit code $?" >&2' ERR
trap 'rm -f "$tmpfile"' EXIT    # 清理临时文件

tmpfile=$(mktemp)
```

**为什么需要 `set -euo pipefail`？**

- `set -e`：防止错误被忽略继续执行（CI 脚本必需）；
- `set -u`：拼写错误的变量直接暴露；
- `pipefail`：`cmd1 | cmd2` 中 cmd1 失败不会被 cmd2 的 0 掩盖。

---

## 十三、四类测试常用脚本

### 13.1 接口健康巡检

```bash
#!/usr/bin/env bash
set -euo pipefail

SERVICES=(
  "order|http://order.test.local/health"
  "pay|http://pay.test.local/health"
  "user|http://user.test.local/health"
)
FAILED=0

for item in "${SERVICES[@]}"; do
  name="${item%%|*}"; url="${item##*|}"
  code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 5 "$url" || echo "000")
  if [[ "$code" == "200" ]]; then
    echo "✅ $name OK"
  else
    echo "❌ $name 返回 $code ($url)"
    FAILED=1
  fi
done
exit $FAILED
```

### 13.2 接口断言脚本（带重试）

```bash
request_with_retry() {
  local url="$1" expect="$2" tries=3 delay=2 i
  for ((i=1; i<=tries; i++)); do
    local body
    body=$(curl -s --max-time 10 "$url" || true)
    if [[ "$body" == *"$expect"* ]]; then
      echo "第 $i 次成功"; return 0
    fi
    echo "第 $i 次失败，${delay}s 后重试"; sleep "$delay"; delay=$((delay*2))
  done
  return 1
}
request_with_retry "http://api.test.local/order/1" '"status":"PAID"'
```

### 13.3 测试数据准备

```bash
#!/usr/bin/env bash
# 批量生成注册数据（CSV）
set -euo pipefail
OUT="users_$(date +%s).csv"
echo "username,phone,email" > "$OUT"
for i in $(seq 1 1000); do
  echo "autotest_${i}_$RANDOM,138$(printf '%08d' $((RANDOM % 100000000))),autotest_${i}@test.local" >> "$OUT"
done
echo "已生成 $OUT（$(wc -l < "$OUT") 行）"
```

### 13.4 部署与回滚脚本骨架

```bash
#!/usr/bin/env bash
set -euo pipefail

APP="order-service"
VERSION="${1:?用法: $0 <version> [rollback]}"
ACTION="${2:-deploy}"
DEPLOY_DIR="/opt/apps/$APP"
PREV=$(readlink -f "$DEPLOY_DIR/current" 2>/dev/null || true)

deploy() {
  echo "部署 $APP:$VERSION"
  ln -sfn "$DEPLOY_DIR/releases/$VERSION" "$DEPLOY_DIR/current"
  systemctl restart "$APP"
  sleep 5
  curl -sf "http://127.0.0.1:8080/health" >/dev/null || { echo "健康检查失败，回滚"; rollback; }
}

rollback() {
  [[ -n "$PREV" ]] || { echo "无可回滚版本"; exit 1; }
  echo "回滚到 $PREV"
  ln -sfn "$PREV" "$DEPLOY_DIR/current"
  systemctl restart "$APP"
}

case "$ACTION" in
  deploy) deploy ;;
  rollback) rollback ;;
  *) echo "未知动作: $ACTION"; exit 1 ;;
esac
```

---

## 十四、调试与静态检查

```bash
bash -n script.sh       # 只做语法检查
bash -x script.sh       # 打印执行过程
set -x / set +x         # 局部开启/关闭调试
shellcheck script.sh    # 静态检查（强烈推荐）
PS4='+ $LINENO: '       # 调试时显示行号
```

---

## 十五、面试手撕题

| 题目 | 参考命令 |
| --- | --- |
| 统计 IP Top10 | `awk '{print $1}' log \| sort \| uniq -c \| sort -rn \| head` |
| 统计 5xx | `awk '$9 ~ /^5/ {c++} END {print c}' log` |
| 删除 7 天前日志 | `find /log -mtime +7 -delete` |
| 磁盘 >80% 告警 | `df -h \| awk '$5+0>80 {print $6,$5}'` |
| 判断服务存活 | `curl -sf URL >/dev/null && echo OK \|\| echo FAIL` |
| 文件按内容去重 | `sort -u file` |
| 两文件求差集 | `comm -3 <(sort a) <(sort b)` |
| 批量重命名 | `for f in *.txt; do mv "$f" "${f%.txt}.log"; done` |
| 统计单词频次 | `tr -s '[:space:]' '\n' < f \| sort \| uniq -c \| sort -rn` |
| 提取 JSON 字段 | `curl -s url \| jq -r '.data.token'` |

---

## 十六、避坑清单

1. 变量引用不加引号 → 单词拆分/通配符展开；
2. `[ ]` 中变量为空导致语法错误 → 用 `[[ ]]` 或 `"$var"`；
3. 忘记 `set -euo pipefail` → 错误被吞；
4. `for f in $(ls)` → 文件名含空格出错，用 `for f in *`；
5. `cd` 失败继续执行 → `cd dir || exit 1`；
6. 临时文件不清理 → `trap ... EXIT`；
7. 硬编码路径/密钥 → 环境变量或配置注入；
8. 不做参数校验 → `${1:?用法}`；
9. 管道中 `grep` 无匹配导致 0 退出码 → 结合 `|| true` 或检查输出；
10. 用 `kill -9` 处理一切 → 先 TERM 再 KILL。

---

## 十七、参考资料

- [GNU Bash Manual](https://www.gnu.org/software/bash/manual/)
- [ShellCheck](https://www.shellcheck.net/)
- 《鸟哥的Linux私房菜》
