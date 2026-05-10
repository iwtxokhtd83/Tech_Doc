# Linux 常用操作命令讲义

> 本讲义按主题分章整理 Linux 命令行最常用、最高频的操作与排障命令。每章给出**语法**、**常用参数**、**实战示例**与**笔试题**。
>
> 约定：`$` 为普通用户提示符，`#` 为 root 提示符；示例多基于主流发行版（Ubuntu/CentOS/Debian）。

## 目录

1. [命令行基础与 Shell](#1-命令行基础与-shell)
2. [文件与目录操作](#2-文件与目录操作)
3. [查看与编辑文件](#3-查看与编辑文件)
4. [查找与过滤：find / grep / sed / awk](#4-查找与过滤find--grep--sed--awk)
5. [权限、用户与用户组](#5-权限用户与用户组)
6. [进程与任务管理](#6-进程与任务管理)
7. [系统资源与性能监控](#7-系统资源与性能监控)
8. [磁盘与文件系统](#8-磁盘与文件系统)
9. [网络命令](#9-网络命令)
10. [打包与压缩](#10-打包与压缩)
11. [包管理与软件安装](#11-包管理与软件安装)
12. [服务管理：systemd](#12-服务管理systemd)
13. [Shell 脚本基础](#13-shell-脚本基础)
14. [计划任务：cron / at](#14-计划任务cron--at)
15. [SSH 与远程传输](#15-ssh-与远程传输)
16. [日志与排障](#16-日志与排障)
17. [实用小技巧与一行命令](#17-实用小技巧与一行命令)
18. [综合笔试练习](#18-综合笔试练习)

---

## 1. 命令行基础与 Shell

### 1.1 常用 Shell

- `bash`：最常见，GNU 默认
- `zsh`：macOS 默认，插件丰富
- `sh`：POSIX 标准，脚本通用
- `fish`：更现代，不完全 POSIX 兼容

查看当前 shell：`echo $SHELL`；切换：`chsh -s /bin/zsh`。

### 1.2 命令组成

```
command  [options]  [arguments]
ls       -la        /etc
```

- **短选项**：`-l`；可合并 `-la`
- **长选项**：`--all`；可带值 `--color=auto`
- **参数**：操作对象

### 1.3 通配符（Globbing）

| 字符 | 匹配 |
|------|------|
| `*` | 任意长度任意字符（不含 `/`） |
| `?` | 单个字符 |
| `[abc]` | 方括号内任一字符 |
| `[a-z]` | 范围 |
| `{a,b,c}` | 大括号展开，生成多个参数 |
| `**` | 递归（需 `shopt -s globstar`） |

```bash
ls *.log                # 当前目录所有 .log
ls /var/log/**/*.log    # 递归（需 globstar）
rm file{1,2,3}.txt      # rm file1.txt file2.txt file3.txt
```

### 1.4 重定向与管道

| 语法 | 含义 |
|------|------|
| `>` | 覆盖写 stdout |
| `>>` | 追加 stdout |
| `<` | 重定向 stdin |
| `2>` | stderr |
| `&>` 或 `>&` | stdout + stderr |
| `2>&1` | stderr 合并到 stdout |
| `|` | 管道：前者 stdout → 后者 stdin |
| `|&` | 管道且合并 stderr（bash 4+） |

```bash
cmd > out.log 2>&1         # 两种流都写入 out.log
cmd &> out.log             # 等价简写（bash）
cmd 2>/dev/null            # 丢弃错误
cmd1 | tee out.log | cmd2  # 管道同时落盘
```

### 1.5 快捷键（bash/readline）

| 快捷键 | 作用 |
|--------|------|
| `Ctrl+A / Ctrl+E` | 行首 / 行尾 |
| `Ctrl+W` | 删除前一个单词 |
| `Ctrl+U / Ctrl+K` | 删到行首 / 行尾 |
| `Ctrl+R` | 反向搜索历史 |
| `Ctrl+L` | 清屏（等同 `clear`） |
| `Ctrl+C` | 中断当前命令 |
| `Ctrl+D` | EOF / 退出 shell |
| `Ctrl+Z` | 挂起到后台 |
| `!!` | 上一条命令 |
| `!$` | 上一条命令最后参数 |
| `!abc` | 最近以 abc 开头的命令 |
| `Alt+.` | 粘贴上一条最后参数 |

### 1.6 获取帮助

```bash
ls --help              # 简短帮助
man ls                 # 完整手册
info ls                # GNU 扩展手册
tldr ls                # 社区简化示例（需 tldr 工具）
type ls                # 查看命令类型（内建/别名/外部）
which ls               # 外部命令路径
apropos keyword        # 按关键字搜索 man
```

### 📝 笔试题 1-1：`>` 和 `>>` 的区别？

`>` 会**截断**（覆盖）目标文件；`>>` **追加**。常见陷阱：`cmd > file` 会先清空文件再写入，即使 `cmd` 失败也会导致 `file` 被清空。

### 📝 笔试题 1-2：`2>&1 >file` 与 `>file 2>&1` 的区别？

- `>file 2>&1`：先把 stdout 改到 file，再把 stderr 复制到"当前的 stdout"（即 file）。**两者都进 file**。
- `2>&1 >file`：先把 stderr 复制到"当前 stdout"（终端），再把 stdout 改到 file。**stderr 仍打到终端**。

顺序很重要，这是 Shell 里经典坑。

---

## 2. 文件与目录操作

### 2.1 路径与导航

```bash
pwd                    # 当前工作目录
cd ~                   # 家目录
cd -                   # 上一个目录
cd ..                  # 上一级
pushd /tmp; popd       # 目录栈
```

绝对路径以 `/` 开始；相对路径以当前目录为基准。`~user` 表示某用户家目录。

### 2.2 列目录 `ls`

```bash
ls              # 普通列出
ls -l           # 长格式
ls -la          # 含隐藏
ls -lh          # 人类可读大小
ls -ltr         # 按修改时间升序
ls -S           # 按大小排序
ls -R           # 递归
ls --color=auto
```

`ls -l` 输出解读：

```
-rw-r--r-- 1 user group 2048 Jan  1 10:00 file.txt
│└─┬─┘└─┬─┘ │  │    │    │      │         │
│  │    │   │  │    │    │      │         └ 文件名
│  │    │   │  │    │    │      └ 修改时间
│  │    │   │  │    │    └ 大小
│  │    │   │  │    └ 所属组
│  │    │   │  └ 所有者
│  │    │   └ 硬链接数
│  │    └ 其他人权限
│  └ 组权限
└ 所有者权限  (首字符：- 普通文件, d 目录, l 符号链接, b/c 设备, s socket)
```

### 2.3 创建与删除

```bash
mkdir dir                  # 单级
mkdir -p a/b/c             # 递归创建
mkdir -m 755 dir           # 指定权限

rm file                    # 删文件
rm -r dir                  # 递归删目录
rm -f file                 # 强制（不提示）
rm -rf /path               # ⚠️ 危险命令，慎用
rmdir dir                  # 只能删空目录
```

### 2.4 复制与移动

```bash
cp src dst                 # 复制文件
cp -r dir1 dir2            # 递归复制目录
cp -a src dst              # 归档模式（保留权限/链接/时间戳）
cp -u src dst              # 仅在源较新或目标不存在时复制
cp -v src dst              # 详细输出

mv a b                     # 重命名或移动
mv -i a b                  # 覆盖前提示
mv -n a b                  # 不覆盖已存在文件
```

### 2.5 链接

- **硬链接**：指向同一 inode，不能跨文件系统，不能对目录
- **软链接（符号链接）**：类似快捷方式，独立 inode，可跨文件系统

```bash
ln target hardlink              # 硬链接
ln -s /abs/path softlink        # 软链接
readlink -f softlink            # 查看最终指向
unlink file                     # 删除链接（不追踪）
```

### 2.6 inode 与删除机制

- 文件名存储在目录项，数据存储在 inode
- `rm` 减少硬链接计数，计数为 0 且无进程打开时才真正释放
- 进程仍持有句柄时删除会"看似空间没释放"，用 `lsof | grep deleted` 查

### 📝 笔试题 2-1：硬链接和软链接的区别？

| 维度 | 硬链接 | 软链接 |
|------|--------|--------|
| inode | 与源相同 | 独立 |
| 跨文件系统 | ❌ | ✅ |
| 可链接目录 | ❌ | ✅ |
| 源删除后 | 仍可访问（引用计数 > 0） | 变成 dangling |
| 存储内容 | 目录项 | 目标路径字符串 |

### 📝 笔试题 2-2：`rm -rf /` 加了一个空格变成 `rm -rf / home/user/tmp` 会怎样？

递归删除 **根目录** 及后面列出的目录，系统基本被摧毁。生产环境务必：

- 用 `trash-cli` 等替代
- Shell 加保护：`alias rm='rm -i'`
- 危险路径拦截：`set -o noclobber` 等

---

## 3. 查看与编辑文件

### 3.1 全量查看

```bash
cat file              # 输出全文
cat -n file           # 带行号
tac file              # 倒序输出
nl file               # 带行号（跳过空行）
```

### 3.2 分页查看

```bash
less file             # 推荐，可前后翻页、搜索
more file             # 简单分页，只能向前
```

`less` 内部：
- `/pattern`：向下搜索；`n`/`N` 下一个/上一个
- `G` / `g`：末尾 / 开头
- `q`：退出
- `-N`：显示行号；`&pat`：只显匹配行

### 3.3 首尾

```bash
head file             # 前 10 行
head -n 20 file       # 前 20 行
head -c 100 file      # 前 100 字节

tail file             # 后 10 行
tail -n +5 file       # 从第 5 行到末尾
tail -f file          # 实时跟踪（文件轮转会断）
tail -F file          # 跟踪并处理轮转
```

### 3.4 统计

```bash
wc -l file            # 行数
wc -w file            # 单词数
wc -c file            # 字节数
wc -m file            # 字符数（多字节）
```

### 3.5 比较

```bash
diff a b              # 行级差异
diff -u a b           # 统一格式（常用于 patch）
diff -r dir1 dir2     # 递归比较目录

cmp a b               # 字节级比较
md5sum file           # MD5 指纹
sha256sum file        # SHA-256
```

### 3.6 编辑器

- **nano**：最易上手，底部有快捷键提示
- **vim**：强大但陡峭，三种模式（普通/插入/命令）
- **emacs**：可扩展性极强

Vim 急救：

| 操作 | 命令 |
|------|------|
| 进入插入 | `i`、`a`、`o` |
| 回到普通模式 | `Esc` |
| 保存退出 | `:wq` 或 `ZZ` |
| 不保存退出 | `:q!` |
| 撤销 / 重做 | `u` / `Ctrl+R` |
| 行首 / 行尾 | `0` / `$` |
| 复制行 / 粘贴 | `yy` / `p` |
| 删除行 | `dd` |
| 搜索 | `/pattern`，`n` 下一个 |

### 📝 笔试题 3-1：`tail -f` 与 `tail -F` 的区别？

- `tail -f`：按文件描述符跟踪；日志被 `logrotate` 切换时会**继续跟旧文件**，新文件不显示
- `tail -F`：按文件名跟踪；发现文件被替换后重新打开。**生产日志监控必用**

---

## 4. 查找与过滤：find / grep / sed / awk

### 4.1 find

按**文件属性**查找，支持表达式组合。

```bash
find /path -name "*.log"            # 按名字（区分大小写）
find . -iname "*.LOG"               # 忽略大小写
find . -type f                      # 文件；d 目录；l 链接
find . -size +10M                   # 大于 10MB；-10M 小于
find . -mtime -7                    # 7 天内修改；+7 超过 7 天前
find . -mmin -30                    # 30 分钟内
find . -user alice
find . -perm 644
find . -name "*.tmp" -delete        # 删除
find . -name "*.sh" -exec chmod +x {} \;    # 对每个结果执行
find . -name "*.log" -exec rm {} +  # 批量传参（更高效）

# 结合 xargs（更推荐，避免 exec 的参数长度限制）
find . -name "*.log" | xargs rm
find . -name "*.log" -print0 | xargs -0 rm   # 处理文件名含空格
```

常见时间选项：`-atime` 访问、`-mtime` 修改、`-ctime` inode 变更。

### 4.2 locate

基于事先索引，速度极快但不实时。需要 `updatedb` 维护：

```bash
sudo updatedb
locate passwd
```

### 4.3 grep：文本搜索

```bash
grep "pattern" file
grep -i "pattern" file         # 忽略大小写
grep -v "pattern" file         # 反向（不含）
grep -c "pattern" file         # 计数
grep -n "pattern" file         # 显示行号
grep -l "pattern" *.log        # 只显示命中的文件名
grep -r "pattern" dir/         # 递归
grep -w "word" file            # 按整词匹配
grep -A 3 "err" log            # 命中行后 3 行
grep -B 3 "err" log            # 前 3 行
grep -C 3 "err" log            # 前后各 3 行
grep -E "a|b" file             # 扩展正则（或用 egrep）
grep -P "\d+" file             # PCRE（perl 正则）
grep -F "literal" file         # 按字面匹配（不解析正则）
```

**组合示例**：

```bash
ps aux | grep -v grep | grep nginx
dmesg | grep -iE "error|fail|warn"
grep -rni "TODO" src/
```

### 4.4 sed：流编辑器

逐行处理，基于**地址 + 命令**。

```bash
sed 's/old/new/' file          # 每行第一个匹配
sed 's/old/new/g' file         # 全部匹配
sed -i 's/old/new/g' file      # 原地修改
sed -i.bak 's/old/new/g' file  # 修改前备份
sed '/pattern/d' file          # 删除匹配行
sed '2,5d' file                # 删除 2-5 行
sed -n '10,20p' file           # 打印 10-20 行
sed 's/^/#/' file              # 每行开头加 #
sed '$a\end' file              # 末尾添加一行 end
```

> macOS 的 sed 与 GNU sed 参数略有差异（`-i` 必须带后缀），跨平台脚本要小心。

### 4.5 awk：列文本处理

按记录（行）和字段处理：

```bash
awk '{print $1, $3}' file      # 打印第 1、3 列
awk -F: '{print $1}' /etc/passwd    # 按 : 分列
awk 'NR==1' file               # 第 1 行
awk 'NR>1 && $3>100' file      # 条件过滤
awk '{sum+=$1} END{print sum}' file          # 求和
awk '{a[$1]++} END{for(k in a) print k,a[k]}' file  # 频次
awk '/error/{print NR, $0}' file              # 带行号打印匹配行

# 经典组合：统计日志中状态码
awk '{print $9}' access.log | sort | uniq -c | sort -rn | head
```

内置变量：`NR` 行号、`NF` 当前行字段数、`$0` 整行、`$1..$n` 字段、`FS` 输入分隔、`OFS` 输出分隔。

### 4.6 sort / uniq / cut / tr

```bash
sort file                # 升序
sort -r file             # 降序
sort -n file             # 数值排序
sort -u file             # 去重
sort -k2,2 -t: file      # 按第 2 列

uniq file                # 去相邻重复（常与 sort 连用）
uniq -c file             # 计数
uniq -d file             # 只显示重复行

cut -d: -f1,3 /etc/passwd    # 按 : 取 1、3 列
cut -c1-10 file              # 按字符位置 1-10

tr 'a-z' 'A-Z' < file        # 大写
tr -d '\r' < file            # 去掉 \r
tr -s ' ' < file             # 压缩连续空格
```

### 📝 笔试题 4-1：统计日志文件中 IP 访问次数 Top 10

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head
```

### 📝 笔试题 4-2：把 `config.yaml` 中所有 `debug: true` 改为 `debug: false`

```bash
sed -i.bak 's/debug: true/debug: false/g' config.yaml
```

### 📝 笔试题 4-3：查找 `/var/log` 中最近 1 天修改过的大于 100MB 的文件

```bash
find /var/log -type f -mtime -1 -size +100M
```

---

## 5. 权限、用户与用户组

### 5.1 权限模型

每个文件的权限分 3 组 × 3 位：**所有者 / 所属组 / 其他**，每组 `rwx`。

| 字符 | 数字 | 文件含义 | 目录含义 |
|------|------|----------|----------|
| r | 4 | 读内容 | 列出条目 |
| w | 2 | 写内容 | 增删改条目 |
| x | 1 | 执行 | 进入（cd） |

组合：`rwxr-xr--` = `754`。

### 5.2 特殊权限

| 位 | 数字 | 显示 | 作用 |
|----|------|------|------|
| SUID | 4000 | `s`（所有者 x 位） | 以文件**所有者**权限执行。典型：`/usr/bin/passwd` |
| SGID | 2000 | `s`（组 x 位） | 以**所属组**权限执行；作用于目录时新建文件继承组 |
| Sticky | 1000 | `t`（其他 x 位） | 目录内文件只能被**所有者**删除。典型：`/tmp` |

```bash
chmod 4755 file     # 加 SUID
chmod u+s file
chmod g+s dir       # 目录 SGID
chmod +t dir        # Sticky
```

### 5.3 常用命令

```bash
chmod 755 file          # 数字
chmod u+x,g-w file      # 符号
chmod -R 644 dir        # 递归

chown user file
chown user:group file
chown -R user:group dir

chgrp group file
umask                   # 查看缺省掩码；022 → 新文件 644
umask 027
```

### 5.4 用户与组

```bash
# 用户
useradd alice                # 创建用户（可能不建家目录，按发行版）
useradd -m -s /bin/bash alice  # 建家目录并指定 shell
passwd alice                 # 设密码
usermod -aG docker alice     # 追加到组（-a 追加，-G 组列表）
userdel -r alice             # 删除用户及家目录

# 组
groupadd dev
groupmod -n newname oldname
groupdel dev

# 查看
id alice
groups alice
whoami
who                          # 当前登录者
w                            # 登录者 + 正在做什么
last                         # 历史登录
```

### 5.5 切换身份

```bash
su                    # 切换到 root（需 root 密码）
su - alice            # 以 alice 身份，加 `-` 加载其环境
sudo command          # 提权执行单条（需 sudoers 授权）
sudo -i               # 进入 root shell
sudo -u alice cmd     # 以 alice 身份执行
visudo                # 编辑 /etc/sudoers（有语法校验）
```

### 5.6 关键文件

- `/etc/passwd`：用户信息（`name:x:uid:gid:comment:home:shell`）
- `/etc/shadow`：密码哈希（仅 root 可读）
- `/etc/group`：组信息
- `/etc/sudoers`：sudo 规则，用 `visudo` 编辑

### 📝 笔试题 5-1：umask 022 意味着什么？

新建**文件**默认权限 = `666 - 022 = 644`；**目录** = `777 - 022 = 755`。注意文件默认无 x 权限，实际按位与。

### 📝 笔试题 5-2：如何让目录下新建文件自动归属某个组？

给目录加 **SGID**：

```bash
chgrp dev /project
chmod g+s /project
```

后续在 `/project` 下创建的文件都会继承组 `dev`。

---

## 6. 进程与任务管理

### 6.1 进程查看

```bash
ps                  # 当前 shell 的进程
ps aux              # 所有进程，BSD 风格
ps -ef              # 所有进程，SysV 风格
ps -eo pid,ppid,user,%cpu,%mem,cmd --sort=-%cpu | head

pgrep nginx         # 按名字找 PID
pgrep -u alice      # 某用户
pidof nginx         # 按名字找 PID（更严格）
```

### 6.2 顶部视图

```bash
top                 # 交互式
# 内部快捷键：
#   P  按 CPU 排序
#   M  按内存排序
#   k  杀进程；输入 PID + 信号
#   1  显示每颗 CPU
#   q  退出

htop                # 更友好（需安装）
```

### 6.3 发信号与杀进程

```bash
kill PID                # 默认 SIGTERM(15)
kill -9 PID             # SIGKILL，强制
kill -STOP PID          # 暂停
kill -CONT PID          # 继续
killall nginx           # 按名字批量
pkill -f "java.*app"    # 按命令行正则
```

**常用信号**：

| 信号 | 编号 | 作用 | 能否捕获 |
|------|------|------|----------|
| SIGHUP | 1 | 重新加载配置（惯例） | ✅ |
| SIGINT | 2 | `Ctrl+C` 中断 | ✅ |
| SIGQUIT | 3 | `Ctrl+\` 退出并 core dump | ✅ |
| SIGKILL | 9 | 强杀 | ❌（不可屏蔽） |
| SIGTERM | 15 | 优雅退出 | ✅ |
| SIGSTOP | 19 | 暂停 | ❌ |
| SIGCONT | 18 | 继续 | ✅ |
| SIGUSR1/2 | 10/12 | 用户自定义 | ✅ |

### 6.4 前后台与作业控制

```bash
cmd &                # 后台运行
jobs                 # 当前 shell 的作业
fg %1                # 作业 1 放回前台
bg %1                # 作业 1 在后台继续
Ctrl+Z               # 挂起到后台（停止状态）
disown %1            # 脱离当前 shell，不受退出影响
nohup cmd &          # 忽略 HUP 信号，即使断开 SSH 仍运行，输出到 nohup.out
setsid cmd           # 开启新会话，完全脱离终端
```

### 6.5 进程树

```bash
pstree -p            # 显示父子进程树，带 PID
pstree -u alice
```

### 6.6 文件与进程的关系

```bash
lsof                          # 所有打开的文件
lsof -i:80                    # 占 80 端口的进程
lsof -u alice                 # alice 打开的文件
lsof -p PID
lsof | grep deleted           # 已删但被占用的文件
fuser -v /var/log/app.log     # 谁在打开此文件
```

### 📝 笔试题 6-1：`kill -9` 和 `kill` 的区别？

默认 `kill` 发 `SIGTERM(15)`，进程可以捕获并做清理（优雅关闭）；`kill -9` 发 `SIGKILL`，**内核直接终止**，进程无机会清理。应优先 `SIGTERM`，万不得已再 `-9`。

### 📝 笔试题 6-2：僵尸进程（Zombie）是什么？

子进程退出后，父进程未 `wait` 回收其状态信息，残留的进程表条目。`ps` 中 STAT 为 `Z`。处理：

- 让父进程正确 `wait`
- 父进程异常时，init(1) 会接管孤儿进程并回收
- 实在消除不了可杀父进程，让 init 来收

---

## 7. 系统资源与性能监控

### 7.1 CPU / 内存

```bash
uptime                # 系统负载（1/5/15 分钟）
free -h               # 内存
vmstat 1 5            # CPU + 内存 + IO 汇总，每秒一次共 5 次
mpstat -P ALL 1       # 每 CPU（需 sysstat）
top / htop
```

`free -h` 中 `available` 比 `free` 更准确地反映"可用"内存（含可回收的 cache）。

### 7.2 磁盘 I/O

```bash
iostat -xz 1          # 每秒扩展 IO 统计（需 sysstat）
iotop                 # 按进程看 IO
dstat                 # 综合性能（CPU/IO/Net/...）
pidstat -d 1          # 每进程 IO
```

### 7.3 文件系统用量

```bash
df -h                 # 各挂载点用量
df -i                 # inode 用量
du -sh /path          # 目录总大小
du -sh * | sort -h    # 当前目录各子目录大小
ncdu /path            # 交互式磁盘用量浏览
```

### 7.4 网络流量

```bash
ss -s                 # 连接汇总
iftop                 # 实时接口流量
nethogs               # 按进程看流量
nload                 # 进出口流量图
sar -n DEV 1          # 接口流量历史（sysstat）
```

### 7.5 Load Average 解读

`uptime` 的三个数字：过去 1/5/15 分钟的平均"可运行 + 不可中断"进程数。

- 机器 4 核，load=4 → 刚好饱和
- load 持续 > 核数 → 过载
- load 高但 CPU 低 → 多为 IO 等待（`vmstat` 的 `b` 列大）

### 📝 笔试题 7-1：系统变慢，排查顺序？

**USE 方法**（Utilization / Saturation / Errors）：

1. `uptime`、`top`：先看负载与 CPU
2. `vmstat 1`：看 CPU、IO 等待（`wa`）、内存
3. `iostat -xz 1`：看磁盘 `%util`、`await`、`avgqu-sz`
4. `free -h`：内存、是否 swap
5. `ss -s`、`ss -ant | wc -l`：连接数
6. `dmesg -T | tail`：内核报错
7. 应用日志 + `strace`/`perf` 定位具体进程

---

## 8. 磁盘与文件系统

### 8.1 设备与挂载

```bash
lsblk                     # 块设备树
lsblk -f                  # 含文件系统类型和 UUID
blkid                     # UUID 列表
fdisk -l                  # 分区表
parted -l                 # 更现代，支持 GPT

mount                     # 已挂载列表
mount /dev/sdb1 /mnt      # 挂载
mount -o ro,noexec /dev/sdb1 /mnt
umount /mnt
umount -l /mnt            # 懒卸载（处理"device is busy"）
```

**/etc/fstab** 开机自动挂载：

```
UUID=xxxx  /data  ext4  defaults,noatime  0 2
```

### 8.2 分区与格式化

```bash
fdisk /dev/sdb            # MBR/GPT 分区
parted /dev/sdb
mkfs.ext4 /dev/sdb1       # 格式化
mkfs.xfs /dev/sdb1
mkswap /dev/sdb2; swapon /dev/sdb2

tune2fs -l /dev/sdb1      # ext4 信息
xfs_info /data            # xfs 信息
```

### 8.3 LVM 简介

```bash
pvcreate /dev/sdb         # 物理卷
vgcreate vg0 /dev/sdb     # 卷组
lvcreate -L 10G -n lv_data vg0   # 逻辑卷
mkfs.ext4 /dev/vg0/lv_data
lvextend -L +5G /dev/vg0/lv_data
resize2fs /dev/vg0/lv_data       # 在线扩容（ext4）
```

### 8.4 磁盘快照与检查

```bash
fsck /dev/sdb1            # 检查 ext 文件系统（卸载状态）
xfs_repair /dev/sdb1      # xfs
dd if=/dev/zero of=/tmp/1G.bin bs=1M count=1024   # 造测试文件
dd if=/dev/sdb of=disk.img bs=4M status=progress  # 磁盘镜像
```

### 📝 笔试题 8-1：`df -h` 看不到空间不足但 `Cannot create file` 报 `No space left`？

**inode 用完**了。用 `df -i` 确认 inode 使用率。常见于日志小文件过多。解决：清理小文件、格式化时指定更高 inode 密度。

---

## 9. 网络命令

### 9.1 接口与地址

```bash
ip addr                   # 查看 IP
ip -br a                  # 简洁版
ip link set eth0 up       # 启用接口
ip addr add 10.0.0.2/24 dev eth0
ip route                  # 路由表
ip route add default via 10.0.0.1
ip neigh                  # ARP 邻居表

# 老工具（依赖 net-tools，部分发行版默认不装）
ifconfig
route -n
arp -a
```

### 9.2 连通性与路径

```bash
ping host                 # 发 ICMP Echo
ping -c 4 host
ping -s 1400 -M do host   # 探测 MTU（不分片）

traceroute host           # UDP 方式（可能被防火墙阻）
traceroute -I host        # ICMP
mtr host                  # ping + traceroute 综合
```

### 9.3 端口与连接

```bash
ss -tnlp                  # TCP + 监听 + 数字 + 进程
ss -tnp                   # TCP + 已连接
ss -s                     # 汇总统计
ss -antp '( sport = :80 or dport = :443 )'

# 老工具
netstat -tnlp
netstat -an | grep :80

# 端口探测
nc -zv host 80
telnet host 80
curl -v telnet://host:80
```

### 9.4 HTTP / 下载

```bash
curl https://example.com
curl -I https://example.com                 # HEAD
curl -v https://example.com                 # verbose
curl -L https://example.com                 # 跟随跳转
curl -X POST -d 'a=1' https://api/x
curl -H 'Content-Type: application/json' -d '{"a":1}' https://api/x
curl -o out.html https://...
curl -u user:pass https://...

wget https://example.com/file
wget -c https://...        # 断点续传
wget -r -np -nH https://site/docs/   # 递归抓取
```

### 9.5 DNS

```bash
nslookup example.com
dig example.com
dig @8.8.8.8 example.com
dig +short example.com
dig example.com MX
dig -x 8.8.8.8            # 反向解析
host example.com
getent hosts example.com  # 走系统解析器（含 /etc/hosts）
```

### 9.6 防火墙

```bash
# firewalld（CentOS/RHEL/Fedora）
firewall-cmd --list-all
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --reload

# ufw（Ubuntu）
ufw status
ufw allow 80/tcp
ufw enable

# iptables（底层）
iptables -L -n -v
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# nftables（现代替代）
nft list ruleset
```

### 📝 笔试题 9-1：连不上 `host:8080`，排查步骤？

```bash
ping host                              # 1. 基础连通
traceroute host                        # 2. 路径
nc -zv host 8080                       # 3. 端口可达
ss -tnlp | grep 8080    # 在服务端   # 4. 服务是否监听
iptables -L / firewall-cmd --list-all  # 5. 防火墙
curl -v http://host:8080               # 6. 应用层
```

---

## 10. 打包与压缩

### 10.1 tar

```bash
tar -cvf a.tar dir/          # 打包
tar -xvf a.tar               # 解包
tar -tvf a.tar               # 查看内容
tar -czvf a.tar.gz dir/      # gzip 压缩
tar -cjvf a.tar.bz2 dir/     # bzip2
tar -cJvf a.tar.xz dir/      # xz（压缩率高）
tar -xzvf a.tar.gz -C /tmp/  # 解到指定目录
tar --exclude='*.log' -czvf a.tar.gz dir/
```

参数速记：`c` create / `x` extract / `t` list / `v` verbose / `f` file / `z` gzip / `j` bzip2 / `J` xz。

### 10.2 单文件压缩

```bash
gzip file                    # → file.gz（源被删）
gzip -k file                 # 保留源
gunzip file.gz
zcat file.gz                 # 不解压查看

bzip2 file                   # → file.bz2
xz file                      # → file.xz
```

### 10.3 zip / unzip

```bash
zip -r out.zip dir/
zip -e out.zip file          # 加密
unzip out.zip
unzip -l out.zip             # 列表
unzip -d /tmp out.zip        # 解到指定目录
```

### 10.4 rsync（增量复制）

不是压缩但常用于分发：

```bash
rsync -avh src/ dst/                    # 本地
rsync -avh --delete src/ host:/path/    # 远程，并同步删除
rsync -avhP --exclude='*.log' src/ dst/ # 显示进度、排除
```

`-a` 归档模式（递归+权限+时间+链接）；`-v` 详细；`-h` 人类可读；`-z` 传输压缩；`-P` 进度+断点续传。

### 📝 笔试题 10-1：`tar -czf` 和 `tar -cjf` 压缩率和速度？

- `gzip (z)`：速度快，压缩率一般
- `bzip2 (j)`：压缩率更高，速度慢
- `xz (J)`：最高压缩率，速度最慢
- `zstd (--zstd)`：兼顾速度和压缩率，现代首选

---

## 11. 包管理与软件安装

### 11.1 Debian/Ubuntu：apt

```bash
sudo apt update                       # 更新索引
sudo apt upgrade                      # 升级
sudo apt install nginx
sudo apt remove nginx
sudo apt purge nginx                  # 连配置一起删
sudo apt autoremove                   # 清理依赖
apt search keyword
apt show nginx
dpkg -l                               # 已装列表
dpkg -L nginx                         # nginx 装了哪些文件
dpkg -S /usr/sbin/nginx               # 某文件属于哪个包
sudo dpkg -i pkg.deb                  # 安装 deb
```

### 11.2 RHEL/CentOS：yum / dnf

```bash
sudo yum install nginx            # CentOS 7
sudo dnf install nginx            # CentOS 8+ / Fedora
sudo yum update
sudo yum remove nginx
yum search keyword
yum info nginx
rpm -qa | grep nginx
rpm -qf /usr/sbin/nginx           # 文件归属
rpm -ql nginx                     # 包文件列表
sudo rpm -ivh pkg.rpm             # 安装 rpm
```

### 11.3 通用

```bash
# 从源码
./configure && make && sudo make install

# Snap / Flatpak / AppImage：容器化应用
snap install code --classic
flatpak install flathub org.gimp.GIMP
```

### 11.4 语言生态包管理

| 语言 | 工具 |
|------|------|
| Python | `pip`, `pipx`, `conda`, `poetry`, `uv` |
| Node.js | `npm`, `pnpm`, `yarn` |
| Java | `maven`, `gradle` |
| Go | `go mod` |
| Rust | `cargo` |
| Ruby | `gem`, `bundler` |

---

## 12. 服务管理：systemd

现代发行版普遍用 systemd（PID 1）。早期用 SysV `init` / `service` / `chkconfig`。

### 12.1 常用命令

```bash
systemctl status nginx
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx              # 热重载（如果支持）
systemctl enable nginx              # 开机自启
systemctl disable nginx
systemctl enable --now nginx        # 启用 + 立即启动
systemctl is-active nginx
systemctl is-enabled nginx
systemctl list-unit-files --type=service
systemctl list-units --failed       # 出错服务
systemctl daemon-reload             # 修改 unit 后重载配置
```

### 12.2 日志：journalctl

```bash
journalctl                    # 全部
journalctl -u nginx           # 某服务
journalctl -u nginx -f        # 跟随
journalctl -u nginx --since "1 hour ago"
journalctl --since today
journalctl -p err             # 错误级别（0-7）
journalctl -b                 # 本次启动
journalctl -b -1              # 上一次启动
journalctl --disk-usage       # 占用
```

### 12.3 编写一个简单 unit

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My App
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server
Restart=on-failure
RestartSec=3s

[Install]
WantedBy=multi-user.target
```

部署：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp
```

### 12.4 Target（运行级别）

- `multi-user.target`：多用户命令行（类似 runlevel 3）
- `graphical.target`：图形界面（runlevel 5）
- `rescue.target`：救援模式
- `emergency.target`：紧急模式

```bash
systemctl get-default
systemctl set-default multi-user.target
systemctl isolate rescue.target
```

### 📝 笔试题 12-1：`systemctl reload` 与 `restart` 的区别？

- `reload`：**不中断进程**，发信号让服务重新加载配置（如 nginx 收到 SIGHUP）
- `restart`：停止再启动，**有短暂不可用**

---

## 13. Shell 脚本基础

### 13.1 脚本模板

```bash
#!/usr/bin/env bash
set -euo pipefail              # 错误即退 / 未定义变量报错 / 管道错误传播
IFS=$'\n\t'                    # 更安全的 IFS

echo "Hello $USER"
```

### 13.2 变量

```bash
name="Alice"
echo "Hello $name"
echo "Hello ${name}s"          # 大括号边界
echo 'No expansion $name'      # 单引号不展开

readonly PI=3.14               # 只读
unset name

# 默认值
echo "${var:-default}"         # var 未设则取 default
echo "${var:=default}"         # 并赋值
echo "${var:?error msg}"       # 未设则报错退出
echo "${var:+alt}"             # 已设则取 alt

# 字符串操作
echo "${#str}"                 # 长度
echo "${str:1:3}"              # 子串
echo "${file%.txt}"            # 去掉 .txt 后缀
echo "${file##*/}"             # 取最后一段（basename）
echo "${file%/*}"              # 取路径（dirname）
echo "${str/old/new}"          # 替换第一个
echo "${str//old/new}"         # 替换全部
```

### 13.3 条件判断

```bash
# 数值：-eq -ne -lt -le -gt -ge
if [ "$n" -eq 0 ]; then ...; fi
if (( n > 0 )); then ...; fi         # 算术比较

# 字符串
if [ "$s" = "ok" ]; then ...; fi
if [[ "$s" == ok* ]]; then ...; fi   # 支持通配
if [[ "$s" =~ ^ok ]]; then ...; fi   # 正则

# 文件
if [ -f file ]; then ...; fi         # 是文件
if [ -d dir ]; then ...; fi
if [ -e path ]; then ...; fi         # 存在
if [ -r file ]; then ...; fi         # 可读
if [ -s file ]; then ...; fi         # 非空

# 逻辑
if [ ... ] && [ ... ]; then ...
if [[ cond1 && cond2 ]]; then ...
```

> `[[ ]]` 是 bash 扩展，功能更强更安全；`[` 是 POSIX。

### 13.4 循环

```bash
for i in 1 2 3; do echo $i; done
for i in {1..10}; do echo $i; done
for i in $(seq 1 10); do echo $i; done
for f in *.log; do echo "$f"; done

n=0
while [ $n -lt 10 ]; do echo $n; ((n++)); done

until [ $n -eq 10 ]; do ((n++)); done

# 读文件
while IFS= read -r line; do
    echo "$line"
done < file.txt
```

### 13.5 函数

```bash
greet() {
    local name="$1"           # local 限制作用域
    echo "Hello, $name"
    return 0
}
greet "Alice"

# 参数：$1 $2 ... $# 个数 $@ 所有 $* 所有（合一） $? 上条退出码 $$ 当前 PID
```

### 13.6 参数处理

```bash
while [ $# -gt 0 ]; do
    case "$1" in
        -h|--help)    usage; exit 0 ;;
        -v|--verbose) VERBOSE=1; shift ;;
        -f|--file)    FILE="$2"; shift 2 ;;
        --)           shift; break ;;
        -*)           echo "unknown: $1"; exit 1 ;;
        *)            break ;;
    esac
done

# 或用 getopts（仅短选项）
while getopts "hvf:" opt; do
    case "$opt" in
        h) usage; exit 0 ;;
        v) VERBOSE=1 ;;
        f) FILE="$OPTARG" ;;
    esac
done
```

### 13.7 调试

```bash
bash -x script.sh        # 显示每条命令
set -x; ...; set +x      # 段内调试
set -e                   # 命令失败即退出
set -u                   # 未定义变量报错
set -o pipefail          # 管道任一失败视为失败
trap 'echo fail at $LINENO' ERR     # 错误钩子
trap cleanup EXIT                    # 退出钩子
```

### 📝 笔试题 13-1：`$@` 和 `$*` 的区别？

- 未加引号：几乎一样，按 IFS 分割
- 加引号：
  - `"$@"`：展开为 `"$1" "$2" ...`，**保留参数边界**
  - `"$*"`：展开为 `"$1c$2c..."`，用 IFS 的第一个字符连接

**脚本中透传参数用 `"$@"`**。

### 📝 笔试题 13-2：`./script.sh` 与 `source script.sh` 的区别？

- `./script.sh`：开**子 shell** 执行，脚本内变量不影响当前 shell
- `source`（或 `.`）：在**当前 shell** 执行，变量、cd、函数直接影响当前环境

因此切换 `conda`/`nvm` 环境要 `source`。

---

## 14. 计划任务：cron / at

### 14.1 cron

每用户一张 crontab：

```bash
crontab -e           # 编辑本用户
crontab -l           # 查看
crontab -r           # 清空（慎用）
sudo crontab -e -u alice     # 为他人编辑
```

cron 表达式 5 字段：

```
*  *  *  *  *   command
│  │  │  │  │
│  │  │  │  └ 星期 (0-7, 0 和 7 均表示周日)
│  │  │  └── 月份 (1-12)
│  │  └───── 日   (1-31)
│  └──────── 时   (0-23)
└─────────── 分   (0-59)
```

示例：

```cron
0 2 * * *       /opt/backup.sh           # 每天 02:00
*/5 * * * *     /opt/check.sh            # 每 5 分钟
0 */2 * * *     /opt/sync.sh             # 每 2 小时整
0 9 * * 1-5     /opt/report.sh           # 工作日 09:00
15 0 1 * *      /opt/monthly.sh          # 每月 1 日 00:15
```

系统级 crontab：`/etc/crontab`、`/etc/cron.d/`、`/etc/cron.{daily,hourly,weekly,monthly}/`。

**常见陷阱**：

- cron 的 `PATH` 非常短，脚本里用**绝对路径**或显式 `PATH=/usr/local/bin:...`
- 标准输出/错误默认邮件发送，应重定向：`>> /var/log/x.log 2>&1`
- 时区差异：时间按系统时区
- `%` 在 cron 里要转义：`\%`

### 14.2 at：一次性任务

```bash
echo "/opt/run.sh" | at 02:00           # 今天 02:00
echo "cmd" | at now + 1 hour
atq                                      # 列队列
atrm JOB_ID
```

### 14.3 systemd timer

替代 cron 的现代方式，可与 unit 配合，示例：

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily backup

[Timer]
OnCalendar=daily
Persistent=true
Unit=backup.service

[Install]
WantedBy=timers.target
```

启用：`systemctl enable --now backup.timer`；查看：`systemctl list-timers`。

### 📝 笔试题 14-1：cron 任务不执行怎么排查？

1. `systemctl status cron`（或 `crond`）
2. `grep CRON /var/log/syslog` / `journalctl -u cron`
3. 命令是否用绝对路径？`PATH` 是否足够？
4. 脚本末尾加 `>> /tmp/cron.log 2>&1` 观察
5. 是否注释、时区、是否放在正确用户的 crontab
6. 是否单引号吞了 `%`

---

## 15. SSH 与远程传输

### 15.1 ssh 登录

```bash
ssh user@host
ssh -p 2222 user@host
ssh -i ~/.ssh/id_ed25519 user@host
ssh -v user@host                    # 调试信息
ssh -o StrictHostKeyChecking=no user@host   # 跳过指纹（慎用）
```

### 15.2 密钥登录

```bash
# 本地生成（推荐 ed25519）
ssh-keygen -t ed25519 -C "alice@pc"
# 分发公钥
ssh-copy-id user@host
# 手动等价：把 ~/.ssh/id_ed25519.pub 内容追加到 host 上 ~/.ssh/authorized_keys
```

权限要求（否则 sshd 拒绝）：

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_ed25519
```

### 15.3 配置文件 ~/.ssh/config

```
Host prod
    HostName 10.0.0.5
    User deploy
    Port 2222
    IdentityFile ~/.ssh/id_prod
    ServerAliveInterval 60
    ServerAliveCountMax 3

Host bastion
    HostName bastion.example.com
    User alice

Host inner
    HostName 10.1.1.5
    User deploy
    ProxyJump bastion        # 跳板机
```

然后：`ssh prod`、`ssh inner`（会自动走 bastion）。

### 15.4 文件传输

```bash
scp file user@host:/path/         # 上传
scp user@host:/path/file ./       # 下载
scp -r dir/ user@host:/path/      # 递归
scp -P 2222 file host:/tmp        # 自定端口（scp 用大 P）

# rsync（更快、支持断点与差异）
rsync -avhP file user@host:/path/
rsync -avhP -e "ssh -p 2222" src/ host:/path/

# sftp 交互式
sftp user@host
# 命令：ls/cd/get/put/mput/mget/bye
```

### 15.5 隧道与端口转发

```bash
# 本地转发：本地 :8080 → 远端 host 的 3306
ssh -L 8080:localhost:3306 user@host

# 远程转发：把远端 :9000 映射回本地 :80
ssh -R 9000:localhost:80 user@host

# 动态代理（SOCKS）
ssh -D 1080 user@host
```

### 15.6 保持连接

- `~/.ssh/config` 的 `ServerAliveInterval 60` 每 60s 发心跳
- 本地多路复用，加速后续连接：

```
Host *
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m
```

### 📝 笔试题 15-1：公钥登录不成功的常见原因？

- 家目录/`.ssh`/`authorized_keys` 权限过宽
- 公钥未正确贴入 `authorized_keys`（尤其多行/末尾空行）
- `sshd_config` 中 `PubkeyAuthentication` 被关、`AuthorizedKeysFile` 路径不对
- SELinux 标签（`restorecon -R ~/.ssh/`）
- 服务端日志：`journalctl -u sshd` / `/var/log/auth.log`

---

## 16. 日志与排障

### 16.1 日志位置

- **journald**：`journalctl -u service`
- 传统 `rsyslog` 文件：
  - `/var/log/messages` 或 `/var/log/syslog`：系统综合
  - `/var/log/auth.log` / `/var/log/secure`：认证
  - `/var/log/dmesg`：内核环缓冲
  - `/var/log/kern.log`
  - `/var/log/nginx/`、`/var/log/mysql/` 等应用

### 16.2 内核与硬件

```bash
dmesg | tail
dmesg -T                  # 人类时间
dmesg -w                  # 跟随
uname -a                  # 内核版本
lscpu                     # CPU 信息
lsmem / free -h           # 内存
lspci                     # PCI 设备
lsusb                     # USB 设备
```

### 16.3 日志轮转：logrotate

配置通常在 `/etc/logrotate.d/`：

```
/var/log/myapp/*.log {
    daily
    rotate 14
    size 100M
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    postrotate
        systemctl reload myapp > /dev/null 2>&1 || true
    endscript
}
```

常见字段：`daily`/`weekly`、`rotate N` 保留份数、`compress` 压缩、`copytruncate` 拷贝+清空（适合应用不会重开日志的场景）。

### 16.4 性能/调用追踪

```bash
strace -p PID                      # 跟系统调用
strace -c -p PID                   # 汇总
strace -e openat,read cmd

ltrace -p PID                      # 库调用
lsof -p PID                        # 打开的 fd

perf top                           # 实时热点
perf record -g -p PID; perf report # 采样后分析

# eBPF 工具 (bpfcc-tools)
execsnoop / opensnoop / tcpconnect / tcpretrans
```

### 16.5 核心转储（core dump）

- `ulimit -c unlimited` 允许生成
- 位置：`/proc/sys/kernel/core_pattern`（很多系统用 `systemd-coredump`，`coredumpctl list/info/dump`）
- 结合 `gdb /path/to/bin core` 分析

### 📝 笔试题 16-1：如何实时监控某服务日志并只显示 ERROR？

```bash
journalctl -u myapp -f | grep -E --line-buffered "ERROR|FATAL"
# 或文件
tail -F /var/log/myapp.log | grep --line-buffered -i error
```

`--line-buffered` 避免管道缓冲导致输出滞后。

---

## 17. 实用小技巧与一行命令

### 17.1 快速操作

```bash
# 替换上一条命令中的字符串
^old^new^

# 上次命令的最后参数
ls /tmp
cd !$

# 用历史命令
!ssh           # 最近以 ssh 开头

# 计算
echo $((1+2*3))
bc <<< "scale=4; 22/7"

# 日期
date
date +%Y%m%d-%H%M%S
date -d "1 day ago" +%F
date -d @1700000000       # 时间戳 → 人类时间

# 临时挂载 tmpfs
sudo mount -t tmpfs -o size=1G tmpfs /mnt/ramdisk
```

### 17.2 文本一行流

```bash
# 去重并保持顺序
awk '!seen[$0]++' file

# 每行添加行号
nl file

# 列对齐
column -t file

# JSON 美化
jq . data.json
curl -s api | jq '.items[] | {id,name}'

# YAML 解析（yq）
yq '.services[].image' docker-compose.yml

# 随机
shuf file | head -1       # 随机一行
shuf -i 1-100 -n 5        # 1-100 取 5 个
openssl rand -hex 16      # 随机十六进制
```

### 17.3 批量处理

```bash
# 批量重命名
for f in *.jpeg; do mv "$f" "${f%.jpeg}.jpg"; done

# 并行执行（xargs）
cat urls.txt | xargs -n1 -P8 curl -sO

# GNU parallel
parallel -j8 'convert {} {.}.webp' ::: *.png
```

### 17.4 性能测试

```bash
time cmd                        # 墙钟+user+sys
hyperfine 'cmd1' 'cmd2'        # 基准测试

# 磁盘顺序写/读
dd if=/dev/zero of=test bs=1M count=1024 conv=fdatasync
dd if=test of=/dev/null bs=1M
```

### 17.5 安全删除 / 查找敏感

```bash
shred -u file              # 覆盖后删除
# 敏感文件体检
find / -name "id_rsa" 2>/dev/null
find / -perm -4000 2>/dev/null   # 所有 SUID
```

---

## 18. 综合笔试练习

### 18.1 选择题

**Q1** 下列哪条命令会**清空** `file` 的内容？
A. `cat file > file`  B. `> file`  C. `truncate -s 0 file`  D. 以上都会

<details><summary>答案</summary>D。B 和 C 直接清空；A 会被 shell 先截断后再读，内容清零。</details>

**Q2** `chmod 750 dir` 后，**其他人**可以？
A. 读写执行  B. 读执行  C. 什么都不能  D. 只能读

<details><summary>答案</summary>C。750 其他人为 0。</details>

**Q3** 下列关于硬链接说法正确的是？
A. 可以跨文件系统
B. 可以链接目录
C. 和源文件共享 inode
D. 删除源文件后硬链接失效

<details><summary>答案</summary>C。</details>

**Q4** 以下哪条让 shell 脚本在命令失败时立即退出？
A. `set -e`  B. `set -x`  C. `set -u`  D. `set -o pipefail`

<details><summary>答案</summary>A。B 调试，C 未定义变量报错，D 管道错误传播。</details>

**Q5** `tail -F` 比 `tail -f` 的优势是？
A. 更快  B. 能跟踪日志轮转  C. 支持多文件  D. 节省内存

<details><summary>答案</summary>B。</details>

### 18.2 判断题

1. `cd /` 与 `cd` 等价。 ❌（`cd` 回家目录）
2. `0` 退出码表示成功。 ✅
3. `find -exec rm {} +` 比 `\;` 更高效。 ✅（批量传参）
4. `grep` 默认支持 PCRE。 ❌（需 `-P`）
5. `kill -9` 可被进程屏蔽。 ❌
6. `/etc/shadow` 普通用户可读。 ❌
7. 软链接删除源后仍可打开。 ❌（dangling）
8. `nohup cmd &` 即使 SSH 断开仍运行。 ✅

### 18.3 简答题

**Q1** 描述一次 "磁盘空间告警" 的排查步骤。

```bash
df -h                               # 哪个挂载点满
df -i                               # 是否 inode 满
du -h --max-depth=1 /var | sort -h  # 定位大目录
ncdu /var                            # 交互定位
lsof | grep deleted                  # 已删但被占用
find / -size +1G -type f 2>/dev/null
```

处理：清理 / 压缩 / 轮转日志 / 扩容 LVM。

**Q2** Linux 里"孤儿进程"和"僵尸进程"的区别？

- **孤儿进程**：父进程先退出，子进程被 init(1) 收养，仍正常运行
- **僵尸进程**：子进程已退出，父进程未 `wait` 回收其退出状态，仅保留 PCB 占着进程表

**Q3** 如何永久修改某目录的默认权限（新建文件继承）？

三件套：

1. `setfacl -d -m u::rwx,g::rwx,o::--- /dir`（默认 ACL）
2. 目录加 `SGID`：新建文件归属目录所属组
3. 设置 `umask`（全局或在服务启动脚本中）

**Q4** 如何优雅地重启一个高可用服务而不丢请求？

- **滚动更新**：多实例逐个重启，配合负载均衡健康检查
- **优雅停机**：监听 `SIGTERM`，停止接收新连接、处理完 in-flight、再退出
- **端口复用** `SO_REUSEPORT` 可让新老进程并存接管
- `systemctl reload` 走 SIGHUP 热加载（如 nginx）

### 18.4 编程/排障题

**Q1** 写一条命令：统计 `access.log` 中 5xx 响应的 URL 及次数，按降序输出前 10。

```bash
awk '$9 ~ /^5/ {print $7}' access.log | sort | uniq -c | sort -rn | head
```

**Q2** 监控某进程 CPU 占用，每秒刷新，超过 80% 发邮件（伪代码）。

```bash
while :; do
    cpu=$(ps -p "$PID" -o %cpu= | tr -d ' ')
    if (( $(echo "$cpu > 80" | bc -l) )); then
        echo "high cpu: $cpu" | mail -s "alert" ops@x.com
    fi
    sleep 1
done
```

**Q3** 找出 `/var/log` 里**包含** `panic` 并且修改时间在 24 小时内的文件。

```bash
find /var/log -type f -mtime -1 -exec grep -l "panic" {} +
```

**Q4** 某进程打开了一个日志文件但占用过大，rm 后空间没释放，怎么办？

空间回收要等**所有持有者**关闭 fd。快速方法：

- 能优雅重启：`systemctl restart app`
- 不便重启：清空文件而非 rm，`: > /var/log/app.log` 或 `truncate -s 0 /var/log/app.log`
- 使用 `copytruncate` 的 logrotate 配合，避免再次出现

---

## 📚 复习与实操建议

1. **必须练习**：`find / grep / sed / awk / xargs` 的组合是 Linux "生产力引擎"
2. **搭个实验机**：VirtualBox / WSL / 云 Mini 实例，敢删敢玩
3. **读 man**：`man 7 signal`、`man 5 crontab` 等章节比博客更准
4. **看源码与日志**：故障多从 `journalctl -xe`、`dmesg -T`、应用自身日志入手
5. **沉淀别名**：把高频命令做成 alias 或 shell 函数（如 `alias ll='ls -lah'`）
6. **开启历史**：`HISTSIZE=10000`、`HISTTIMEFORMAT='%F %T '` 记录时间，排障利器

> 祝 Linux 驾驭顺利！
