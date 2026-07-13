| 工具          | 命令                | 说明                  |
| ----------- | ----------------- | ------------------- |
| **top**     | `top`             | 实时交互式查看进程 CPU 占用    |
| **htop**    | `htop`            | top 增强版，彩色界面，支持鼠标   |
| **mpstat**  | `mpstat -P ALL 1` | 查看每个 CPU 核心的使用率     |
| **vmstat**  | `vmstat 1 5`      | 查看系统整体 CPU、内存、I/O   |
| **sar**     | `sar -u 1 3`      | 系统活动报告，历史数据         |
| **pidstat** | `pidstat -u 1`    | 按进程查看 CPU 使用率       |
| **uptime**  | `uptime`          | 查看系统负载（1/5/15 分钟平均） |
| **lscpu**   | `lscpu`           | 查看 CPU 架构信息         |

# 查看整体 CPU 使用率
`top -bn1 | head -5`

# 查看每个核心使用率
`mpstat -P ALL 1 3`

# 查看进程 CPU 占用排序
`ps aux --sort=-%cpu | head -10`

# 查看系统负载
`uptime`