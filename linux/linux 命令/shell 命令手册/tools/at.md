`at` 命令是 Linux 下用于**一次性定时任务**的经典工具，它从标准输入或指定文件读取命令，在未来的某个时间点执行一次，执行完毕后任务即完成。

---

## 安装与启动

大多数 Linux 发行版默认未安装 `at`，需要手动安装并启动守护进程 `atd`：

**Debian/Ubuntu**
```bash
sudo apt update && sudo apt install -y at
sudo systemctl enable --now atd
```

**RHEL/CentOS/Rocky**
```bash
sudo dnf install -y at
sudo systemctl enable --now atd
```

安装后可用 `rpm -q at`（RHEL 系）或 `dpkg -l at`（Debian 系）确认包已安装，用 `systemctl status atd` 确认服务运行中。

---

## 基本语法

```
at [-V] [-q 队列] [-f 文件] [-mldbv] 时间
at -c 作业 [作业...]
atq [-V] [-q 队列] [-v]
atrm [-V] 作业 [作业...]
batch [-V] [-q 队列] [-f 文件] [-mv] [时间]
```


核心三个命令的分工：

| 命令 | 作用 |
|---|---|
| `at` | 在指定时间执行命令 |
| `atq` | 列出待执行作业 |
| `atrm` | 删除作业 |

---

## 时间格式详解

`at` 支持相当灵活的时间指定方式，扩展了 POSIX.2 标准。

### 1. 绝对时间

| 格式 | 示例 | 说明 |
|---|---|---|
| `HH:MM` | `at 04:00` | 24 小时制；若该时间已过，则顺延到第二天 |
| 小时数 | `at 23` | 等同于 `23:00` |
| `midnight` | `at midnight` | 午夜 00:00 |
| `noon` | `at noon` | 中午 12:00 |
| `teatime` | `at teatime` | 下午 16:00 |
| `AM/PM` 后缀 | `at 5pm` | 下午 5 点 |



### 2. 日期格式

日期必须跟在时间之后：

```bash
at 10am Jul 31              # 7月31日上午10点
at 10am Jul 31 2026         # 指定年份
at 10am 073126              # MMDDYY 格式
at 10am 07/31/26            # MM/DD/YY 格式
at 10am 31.07.26            # DD.MM.YY 格式
```


### 3. 相对时间（`now +`）

这是最常用的方式：

```bash
at now + 1 minute
at now + 2 hours
at now + 3 days
at now + 1 week
```


### 4. `today` / `tomorrow` 后缀

```bash
at 1am tomorrow
at 4pm today
at 4pm + 3 days             # 三天后的下午4点
```


### 5. `-t` 选项（精确时间戳）

使用 `[[CC]YY]MMDDhhmm[.ss]` 格式：

```bash
at -t 202610010300         # 2026年10月1日 03:00
```


---

## 创建任务的两种方式

### 交互式（默认）

直接运行 `at` 加时间，会进入 `at>` 提示符，逐行输入命令，按 `Ctrl+D` 结束：

```bash
$ at 03:00 2026-10-01
at> /path/to/script.sh
at> echo "done" >> /var/log/script.log
at> <Ctrl+D>
job 3 at Thu Oct  1 03:00:00 2026
```


### 管道/文件方式（推荐用于脚本）

```bash
# 管道
echo '/path/to/script.sh >> /var/log/script.log 2>&1' | at now + 1 hour

# 从文件读取
at -f /path/to/commands.txt 03:00 2026-10-01
```


### 常用选项

| 选项 | 含义 |
|---|---|
| `-m` | 任务完成后给用户发邮件（即使无输出） |
| `-M` | 从不发送邮件 |
| `-f 文件` | 从指定文件读取命令 |
| `-q 队列` | 指定队列（队列名可为 a-z、A-Z） |
| `-l` | 等同 `atq`，列出待执行作业 |
| `-d` | 等同 `atrm`，删除作业 |
| `-c 作业号` | 查看指定作业的完整内容 |
| `-v` | 显示作业执行时间 |



---

## 查看、删除与查看作业内容

```bash
# 查看待执行作业（普通用户只看自己的，root 看所有人的）
atq
# 输出示例：
# 3       Thu Oct  1 03:00:00 2026 a username

# 删除作业
atrm 3

# 查看作业的完整内容（包括环境变量、工作目录等）
at -c 3
```


`at -c` 输出的内容不仅包含命令本身，还包括作业执行时的环境变量设置和当前工作目录，这对调试非常有用。

---

## 输出与邮件处理

`at` 任务在后台由 `atd` 执行，**没有控制终端**，命令的标准输出和标准错误默认会通过邮件发送给用户。如果系统未配置邮件服务，输出会丢失。

**推荐做法：主动重定向输出到日志文件**

```bash
echo '/path/to/script.sh >> /var/log/script.log 2>&1' | at now + 1 hour
```

如果确实需要邮件通知：
```bash
echo '/path/to/script.sh' | at -m now + 1 hour   # 无论有无输出都发邮件
echo '/path/to/script.sh' | at now + 1 hour      # 仅在有输出时发邮件
```


---

## 环境变量与工作目录

`at` 作业执行时会**继承创建任务时的环境变量、当前工作目录和 umask**，但不会继承终端相关的变量（如 `TERM`、`DISPLAY`）。

这意味着：
- 脚本中使用的命令**必须写绝对路径**，因为 `PATH` 可能不是你预期的值
- 如果需要特定环境变量，应在脚本内部显式设置或 `source` 相关配置文件

---

## 权限控制

`/etc/at.allow` 和 `/etc/at.deny` 控制哪些用户可以使用 `at`：

| 情况 | 效果 |
|---|---|
| `/etc/at.allow` 存在 | **仅**文件中列出的用户可使用 `at`，`at.deny` 被忽略 |
| `/etc/at.allow` 不存在，`/etc/at.deny` 存在 | 不在 `at.deny` 中的用户均可使用 |
| `at.deny` 为空文件 | 所有用户均可使用 |
| 两个文件都不存在 | **仅 root** 可使用 `at` |

文件格式：每行一个用户名，不允许空格。

---

## `batch` 命令：系统不忙时执行

`batch` 是 `at` 的姊妹命令，不在固定时间执行，而是在**系统平均负载降到允许水平以下**时执行。默认阈值是 **0.8**（不同版本可能为 1.5）。

```bash
echo 'updatedb' | batch
# 或交互式
batch
at> updatedb
at> <Ctrl+D>
```

`batch` 同样用 `atq` 查看、`atrm` 删除。

---

## 常见问题排查

**任务未执行**

1. 确认 `atd` 正在运行：`systemctl status atd`
2. 确认任务已入队：`atq`
3. 查看系统日志：`/var/log/syslog` 或 `/var/log/messages`
4. 确认用户有权限使用 `at`（检查 `/etc/at.allow` 和 `/etc/at.deny`）


**任务执行了但脚本没效果**

- 最常见原因：`PATH` 环境变量不同，脚本中命令未用绝对路径
- 解决方法：在脚本开头显式设置 `PATH`，或所有命令都用绝对路径


**没有收到输出邮件**

- 系统可能未配置 MTA（如 Postfix）
- 优先将输出重定向到文件，而不是依赖邮件


**查看任务详情**

```bash
at -c 作业号    # 查看作业的完整脚本内容、环境变量、工作目录
```


---

## 总结

| 场景 | 推荐方式 |
|---|---|
| 一次性定时任务 | `at` |
| 系统不忙时执行一次性任务 | `batch` |
| 查看待执行作业 | `atq` |
| 删除作业 | `atrm 作业号` |
| 调试作业 | `at -c 作业号` |
| 输出处理 | **主动重定向到文件**，不要依赖邮件 |
| 路径问题 | 脚本内使用**绝对路径** |

`at` 与 `cron` 的分工很明确：`at` 只执行**一次**，`cron` 执行**周期性**任务。需要一次性在某个时间点运行命令时，`at` 是最直接的选择。