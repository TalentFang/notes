# Shell 脚本语法实战笔记

> 基于 maxs-ops 工程中 36 个 Shell 脚本的语法模式整理

---

## 一、脚本骨架与日志函数

### 1.1 标准 logger 函数（几乎所有脚本都有）

```bash
#!/bin/bash

function logger() {
  TIMESTAMP=$(date +'%Y-%m-%d %H:%M:%S')
  case "$1" in
    debug)
      echo -e "$TIMESTAMP \033[36mDEBUG\033[0m $2"   # 青色
      ;;
    info)
      echo -e "$TIMESTAMP \033[32mINFO\033[0m $2"    # 绿色
      ;;
    warn)
      echo -e "$TIMESTAMP \033[33mWARN\033[0m $2"    # 黄色
      ;;
    error)
      echo -e "$TIMESTAMP \033[31mERROR\033[0m $2"   # 红色
      ;;
    *)
      ;;
  esac
}
```

- `\033[36m` / `\033[0m`：ANSI 转义序列控制终端颜色
- `case/esac`：条件分支结构，`;;` 结束每个分支

### 1.2 exec_ansible 包装函数

```bash
function exec_ansible() {
  logger info "start exec $*" | tee -a /data/install.log
  cd /data/maxs-ops
  ansible-playbook "$@" >> /data/install.log
  if [ $? -ne 0 ]; then
      logger error "$* 执行有错误" | tee -a /data/install.log
      exit 1
  fi
  logger info "end exec $1 $2 $3 $4 $5" | tee -a /data/install.log
  cd - > /dev/null
}
```

- `tee -a`：同时输出到终端和追加到日志文件
- `cd -`：返回之前的工作目录，`> /dev/null` 隐藏输出

### 1.3 主入口模式

```bash
function main() {
  # ... 业务逻辑
}

main "$@"     # 将脚本所有参数传给 main
```

---

## 二、命令行参数解析

### 2.1 getopts（推荐）

```bash
while getopts "am:u:e" opt; do
  case $opt in
    a) collect_all=true ;;
    m) module_type=$OPTARG ;;
    u) IFS=',' read -ra units <<< "$OPTARG" ;;
    e) error_logs=true ;;
    \?) echo "无效选项: -$OPTARG" >&2; exit 1 ;;
    :) echo "选项 -$OPTARG 需要参数" >&2; exit 1 ;;
  esac
done
shift $((OPTIND-1))    # 移除已处理选项，剩下位置参数
```

- `"am:u:e"`：`a`/`e` 无参，`m:`/`u:` 需要参数
- `$OPTARG`：当前选项的参数值
- `OPTIND`：当前选项索引，`shift` 后位置参数只剩非选项部分
- `\?`：无效选项，`:`：缺少参数

### 2.2 位置参数 + 参数数量检查

```bash
if [ "$#" -lt 1 ]; then
    usage
    exit 2
fi
```

---

## 三、字符串与变量操作

### 3.1 字符串比较的 `x` 后缀技巧

```bash
# 防止空字符串导致语法错误
if [ "${var}x" == "xx" ]; then
    echo "var 为空"
fi

if [ "$1x" != "startx" ]; then
    echo "不是 start"
fi
```

为什么加 `x`：如果 `$1` 为空，`[ "" != "start" ]` 是合法的，但 `[ != "start" ]` 会语法错误。加 `x` 后即使为空也是 `[ "x" != "startx" ]`，始终合法。

### 3.2 常用字符串操作速查

```bash
# 变量替换
${var/:/=}           # 第一个冒号替换为等号
${var//:/=}          # 所有冒号替换为等号

# 前后缀删除
${var##*_}           # 删除最后一个下划线之前的所有内容（取最后一部分）
${var%%#*}           # 删除第一个 # 之后的所有内容（取最前一部分）
${var%_$last_part}   # 从末尾删除 _xxx（取前面部分）

# 默认值
${var:-default}       # var 为空或未设置时返回 default
${var:=default}       # 同上，但会赋值

# 转大小写
echo "$str" | tr '[:upper:]' '[:lower:]'
echo "$str" | tr '[:lower:]' '[:upper:]'

# 获取长度
${#array[@]}          # 数组长度
${#var}               # 字符串长度
```

### 3.3 引号规则

```bash
name="hello world"
echo "$name"      # hello world（变量展开）
echo '$name'      # $name（原样输出，不展开）
echo `date`       # 执行 date 命令（旧语法）
echo $(date)      # 执行 date 命令（推荐语法）
```

**原则**：`$(...)` > `` ` ` ``（反引号），双引号包裹含变量的字符串，单引号用于纯字符串。

---

## 四、条件判断

### 4.1 文件测试

```bash
if [ -f "$file" ]; then ... fi      # 文件是否存在
if [ -d "$dir" ]; then ... fi       # 目录是否存在
if [ -e "$path" ]; then ... fi      # 路径是否存在（文件或目录）
if [ ! -f "$file" ]; then ... fi    # 文件不存在
if [ -x "$script" ]; then ... fi    # 文件是否可执行
if [ -s "$file" ]; then ... fi      # 文件存在且非空
```

### 4.2 字符串测试

```bash
if [ -z "$var" ]; then ... fi       # 字符串为空
if [ -n "$var" ]; then ... fi       # 字符串非空
if [ "$a" == "$b" ]; then ... fi    # 字符串相等
if [ "$a" != "$b" ]; then ... fi    # 字符串不等
```

### 4.3 数值比较

```bash
if [ "$#" -lt 2 ]; then ... fi      # 小于
if [ $? -ne 0 ]; then ... fi        # 不等于
if [ "$n" -eq 0 ]; then ... fi      # 等于
if [ $# -gt 0 ]; then ... fi        # 大于
if [ "$elapsed" -ge "$test_duration" ]; then ... fi   # 大于等于
```

### 4.4 双括号 `[[ ]]`（bash 扩展，功能更强）

```bash
# 正则匹配
if [[ "$choice" =~ ^[Yy]$ ]]; then ... fi

# 模式匹配（通配符）
if [[ "$filename" != *.sh ]]; then ... fi
if [[ "$filename" != *demo* ]]; then ... fi

# 逻辑组合
if [[ -z "$a" || -n "$b" ]]; then ... fi
if [[ ! "$first_choice" =~ ^[Yy]$ ]]; then ... fi

# root 权限检查
if [[ $EUID -ne 0 ]]; then
    echo "需要 root 权限"
    exit 1
fi
```

### 4.5 与 C 语言风格的数值运算 `(( ))`

```bash
if (( used_plus_gb < threshold_gb )); then
    return 0
else
    return 1
fi

# 算术计算
diff=$((end - start))
minutes=$((diff / 60))
num=$((iops))        # 将变量转为数值
let num+=1            # 自增
GB_TO_KB=$((1024 * 1024))
```

---

## 五、循环结构

### 5.1 for 循环

```bash
# 按空格分隔遍历
for item in $items; do
    echo "$item"
done

# 按文件遍历（按字母排序）
for file in `ls ${dir}/*.sql | sort`; do
    echo "$file"
done

# 按数组遍历
for ip in ${ip_list[*]}; do
    echo "$ip"
done

# 按通配符遍历
for archive in /data/ipaas_update*.tar.gz; do
    if [ -e "$archive" ]; then
        echo "处理 $archive"
    fi
done

# 按行遍历输出
for servicename in $(echo ${service_names[@]} | tr ' ' '\n' | sort); do
    echo "$servicename"
done
```

### 5.2 while 循环

```bash
# 从文件逐行读取
while IFS= read -r line; do
    echo "$line"
done < "$file"

# 读取到变量（最后一行也能读到）
while read line || [[ -n ${line} ]]; do
    echo "$line"
done < "$file"

# 重试循环
retries=0
max_retries=3
while [[ $retries -lt $max_retries ]]; do
    # 执行逻辑
    retries=$((retries + 1))
done

# 无限循环 + break
while true; do
    ops=$((ops + 1))
    if [ "$elapsed" -ge "$test_duration" ]; then
        break
    fi
done
```

### 5.3 `grep` + `|` 管道遍历

```bash
# 注意：管道后的 for 运行在子 shell 中
for ip in $(grep -Pzo "(?s)\[kafka\].*?(?=\[[^\]]+\]|\Z|#)" /data/maxs-ops/inventory/hosts | awk 'NR!=1 {print $1}'); do
    echo "$ip"
done
```

- `grep -Pzo`：Perl 正则 + 多行匹配（`(?s)` = 点号匹配换行），`-z` 将输入作为一条记录，`-o` 只输出匹配部分

---

## 六、常用文本处理命令

### 6.1 awk

```bash
echo "$line" | awk '{print $1}'                          # 打印第一列
echo "$line" | awk -F'lables=' '{print $2}'              # 指定分隔符
echo "$df_output" | awk 'NR==2 {print $2}'               # 处理第二行
cat "$file" | awk -F ':' 'END {print $2}'                # 最后一行
awk -v pattern="$key" '$0 ~ pattern {print $2; exit}' "$file"  # 变量传入
echo "$msg" | cut -d '"' -f4                             # 按双引号切分，取第4段
```

### 6.2 sed

```bash
sed -i "s/\b$origin_ip\b/$replace_ip/g" "$file"         # 替换（单词边界）
sed -i '/10.21.18.156/d' "$file"                         # 删除匹配行
sed -i '/kafka_server_jaas.conf/d' "$file"               # 删除包含字符串的行
sed -i '2iexport KAFKA_OPTS="...'" "$file"                # 第2行后插入
sed -i '/^old/c\new line' "$file"                        # 替换整行
sed -i "${l}a $ip ansible_connection=local" "$file"      # 第l行后追加
sed -i "/$localIP/{$l,${e}d}" "$file"                    # 指定行范围内删除
```

### 6.3 grep

```bash
grep -q "pattern"                       # 静默匹配（返回状态码，不输出）
grep -c "$ip" "$file"                   # 计数匹配行
grep -v '^$'                            # 排除空行
grep -v '^#'                            # 排除注释行
grep -v "hosts"                         # 排除包含 hosts 的行
grep -Pzo "(?s)\[kafka\].*?(?=\[[^\]]+\]|\Z|#)"  # 多行匹配（Perl 正则）
grep -E "\|"                            # 扩展正则（或）
grep 'pattern' | awk ... | wc -l        # 管道组合
```

---

## 七、错误处理与脚本健壮性

### 7.1 退出状态码检查

```bash
ansible-playbook "$@"
if [ $? -ne 0 ]; then
    logger error "$1 执行有错误"
    exit 1
fi
```

### 7.2 严格模式

```bash
set -eo pipefail
```
- `-e`：任何命令失败（返回非零）立即退出
- `-o pipefail`：管道中任何命令失败都算整体失败

### 7.3 交互式确认

```bash
read -p "是否继续？(输入 y/Y 确认，其他键取消): " choice
if [[ ! "$choice" =~ ^[Yy]$ ]]; then
    echo "操作已取消"
    exit 0
fi
```

---

## 八、函数返回值

```bash
# 通过 return 返回整数（0=成功）
function check_disk_threshold() {
    if (( used_plus_gb < threshold_gb )); then
        return 0    # 成功
    else
        return 1    # 失败
    fi
}

# 通过 echo 返回字符串（调用者用 $(...) 捕获）
function calculate_iops() {
    echo "$ops"
}
iops=$(calculate_iops)
```

---

## 九、数组

```bash
# 定义
units=()
valid_module_types=("system" "component" "business" "data")

# 从字符串分割创建数组
IFS=',' read -ra units <<< "$OPTARG"

# 遍历
for unit in "${units[@]}"; do
    echo "$unit"
done

# 长度
${#units[@]}

# 展开为字符串（用逗号连接）
units_string=$(IFS=,; echo "${units[*]}")

# 搜索（判断元素是否在数组中）
if [[ " ${valid_units[*]} " =~ " $unit " ]]; then
    echo "找到了"
fi
```

---

## 十、I/O 重定向速查

```bash
command >> "$LOG_FILE"          # 追加到文件
command >> "$LOG_FILE" 2>&1    # stdout 和 stderr 都追加到文件
command | tee -a "$LOG_FILE"    # 同时输出到终端和追加文件
command > /dev/null              # 丢弃 stdout
command > /dev/null 2>&1        # 丢弃 stdout 和 stderr
command >> "$LOG_FILE"  2>&1   # 追加输出 + 错误到日志
```

---

## 十一、特殊技巧汇总

```bash
# 用 key=value 格式读取文件内容
source /etc/profile
product=$(echo "$PRODUCT_NAME" | tr '[:upper:]' '[:lower:]')

# ← 这里有一个管道符号到 while 的写法（在子 shell 中执行）

# nullglob：通配符无匹配时展开为空，避免返回字面量
shopt -s nullglob
mysql_files=("$sql_dir/mysql"/*.sql)
for sql_file in "${mysql_files[@]}"; do
    echo "处理: $sql_file"
done

# heredoc 多行文本（用于 usage 说明）
cat <<EOF
用法: $0 [选项]
  -a    全量收集
  -m    模块类型
EOF

# 本地变量（限制作用域在函数内）
function foo() {
    local backup_dir="/data/inventory_backup"
    local timestamp=$(date +%Y%m%d%H%M%S)
}

# 获取本机 IP
localIP=$(hostname -I | awk 'NR==1 {print $1}')
localIP=$(ip route get 8.8.8.8 | awk 'NR==1 {print $7}')

# 时间戳
timestamp=$(date +%Y%m%d%H%M%S)
elapsed=$((current_time - start_time))
```

---

## 十二、常见模式速查

| 模式 | 代码 | 用途 |
|------|------|------|
| 参数检查 | `if [ "$#" -lt 2 ]; then usage; fi` | 检查参数数量 |
| 空值判断 | `if [ -z "$var" ]; then ...; fi` | 判断变量为空 |
| 命令成败 | `if [ $? -ne 0 ]; then exit 1; fi` | 检查上一条命令 |
| 文件存在 | `if [ ! -f "$file" ]; then touch "$file"; fi` | 文件不存在则创建 |
| 目录遍历 | `for f in $(ls "$dir"/*.sql \| sort); do ...; done` | 按序处理目录文件 |
| 逐行读 | `while IFS= read -r line; do ...; done < "$file"` | 读取文件每一行 |
| 管道取值 | `val=$(echo "$line" \| awk '{print $1}')` | 从命令提取字段 |
| 内联替换 | `sed -i "s/old/new/g" "$file"` | 文件内容替换 |
| 重试循环 | `while [[ $retries -lt $max ]]; do retries=$((retries+1)); done` | 失败重试 |
