# Systemd & journald 命令速查手册

本文参考 阮一峰 [Systemd 入门教程：命令篇](http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html) 进行精简，详细的内容可以阅读原文。

## 1. systemd管理系统

- **systemctl** reboot/poweroff/halt/suspend/hibernate/hybrid-sleep/rescue
- **systemd-analyze**
  - systemd-analyze                                                   # 查看启动耗时
  - systemd-analyze blame                                       # 查看每个服务的启动耗时
  - systemd-analyze critical-chain                            # 显示瀑布状的启动过程流
  - systemd-analyze critical-chain atd.service        # 显示指定服务的启动流
  - systemd-analyze dot <unit>                                # 以dot文件格式输出unit之间的依赖关系（依赖和被依赖）


- **hostnamectl**
  - hostnamectl                                                            # 显示当前主机的信息
  - hostnamectl set-hostname <name>                   # 设置主机名


- **localectl**
  - localectl                                                                    # 查看本地化设置
  - localectl set-locale LANG=en_GB.utf8                 # 设置本地化参数
  - localectl set-keymap en_GB                                  # 设置本地化参数

## 2. Unit
Systemd可以管理的所有系统资源，不同的资源统称为Unit。

- Unit类型：
  - Service unit：系统服务
  - Target unit：多个 Unit 构成的一个组
  - Device Unit：硬件设备
  - Mount Unit：文件系统的挂载点
  - Automount Unit：自动挂载点
  - Path Unit：文件或路径
  - Scope Unit：不是由 Systemd 启动的外部进程
  - Slice Unit：进程组
  - Snapshot Unit：Systemd 快照，可以切回某个快照
  - Socket Unit：进程间通信的 socket
  - Swap Unit：swap 文件
  - Timer Unit：定时器

- 相关命令：
  - **systemctl list-units**                                  # 查看当前系统所有的Unit
    - --all                                   # 列出所有Unit，包括没有找到配置文件的或者启动失败的
    - --all  --state=inactive      # 列出所有没有运行的 Unit
    - --failed                             # 列出所有加载失败的 Unit
    - --type=service                 # 列出所有正在运行的、类型为 service 的 Unit

  - **systemctl status**                                                            # 查看系统状态和单个 Unit 的状态

    - systemctl status <unit>                                        # 显示某个Unit的状态
    - systemctl -H root@hostname status <unit>     # 显示远程主机上的某个Unit状态

  - systemctl is-active/is-failed/is-enabled <unit>        # 查询Unit状态

  - 管理Unit命令：

    - systemctl enable/disable <unit>                        # 设置/取消开机启动
    - systemctl start/stop/restart/kill/reload <unit> 
    - systemctl ***daemon-reload***                                   #  **【重要】** 重载所有修改过的Unit配置文件
    - systemctl show <unit>   [-p <property_name>]     # 显示某个Unit的全部/某个属性
    - systemctl set-property <unit> <property_name>=<value>  # 设置某个Unit的指定属性
    - systemctl list-unit-files                                            # 列出所有配置文件
      - systemctl list-unit-files --types=service         # 列出指定类型的配置文件
    - systemctl get-default                                               # 操作系统启动时的默认Target
    - systemctl set-default <unit_target>                      # 设置操作系统启动时的默认Target
    - systemctl isolate <unit_target>                             #  关闭前一个 Target 里面所有不属于后一个 Target 的进程

  - Unit依赖关系

    - systemctl list-dependencies <unit>                    # 列出Unit的所有依赖(wanted/required)

    - systemctl list-dependencies --all <unit>             # 列出Unit的所有依赖，并展开Target 类型的Unit依赖

### 3. Unit配置文件

- service 配置文件 

  - 配置服务重启行为：**[Service]** 区块
    - **KillMode** : 定义 Systemd 如何停止 Unit 服务，取值如下：
      - control-group（默认值）：当前控制组里面的所有子进程，都会被杀掉
      - process：只杀主进程
      - mixed：主进程将收到 SIGTERM 信号，子进程收到 SIGKILL 信号
      - none：没有进程会被杀掉，只是执行服务的 stop 命令。
    - **Restart**: 定义了 Unit 退出后，Systemd 的重启方式，取值如下：
      - no（默认值）：退出后不会重启
      - on-success：只有正常退出时（退出状态码为0），才会重启
      - on-failure：非正常退出时（退出状态码非0），包括被信号终止和超时，才会重启
      - on-abnormal：只有被信号终止和超时，才会重启
      - on-abort：只有在收到没有捕捉到的信号终止时，才会重启
      - on-watchdog：超时退出，才会重启
      - always：不管是什么退出原因，总是重启
    - **RestartSec**: Systemd 重启服务之前，需要等待的秒数
  - 其他信息请阅读原文

- target配置文件

  - 详细阅读原文

- 修改配置文件后重启

  修改配置文件以后，***需要重新加载配置文件，然后重新启动相关服务。***

  - systemctl daemon-reload
  - systemctl restart <unit>

### 5. unit dependencies 和启动时序、耗时（个人理解）
  查看unit之间的依赖关系可以使用的命令：systemctl list-dependencies <unit>、systemd-analyze blame、 systemd-analyze critical-chain <units>、 systemd-analyze dot <units>
- systemctl list-dependencies <unit> : 列出的只是required 和 wanted 的依赖关系
- systemd-analyze dot <units>: 以dot文件格式列出指定的unit(s)的依赖关系和被依赖关系
- systemd-analyze blame: 列出当前运行的unit在系统init的时候，所耗费的时间（排序）
- systemd-analyze critical-chain <units> : 显示某个unit在启动时的每一个关键节点及耗时

**注意:**
- systemctl list-dependencies <unit> 列出的只是required 和 wanted 的依赖关系，以树状显示，可以通过--all 参数展开target的依赖关系
- systemd-analyze dot <units> 列出的是这(几)个unit的所有依赖和被依赖的关系(Requires, Wanted, After, Before, Conflicts...) ；dot格式的输出，可以使用在线工具[WebGraphviz](http://www.webgraphviz.com/)来绘图（为了更好地进行展示，可以通过调整dot输出的依赖关系的顺序(按启动时序顺序)、删除重复的依赖关系(去重)、删除不关注的节点来优化图形展示结果）。

### 6. journalctl 日志管理命令

```
# 查看所有日志（默认情况下 ，只保存本次启动的日志）
$ sudo journalctl

# 查看内核日志（不显示应用日志）
$ sudo journalctl -k

# 查看系统本次启动的日志
$ sudo journalctl -b
$ sudo journalctl -b -0

# 查看上一次启动的日志（需更改设置）
$ sudo journalctl -b -1

# 查看指定时间的日志
$ sudo journalctl --since="2012-10-30 18:17:16"
$ sudo journalctl --since "20 min ago"
$ sudo journalctl --since yesterday
$ sudo journalctl --since "2015-01-10" --until "2015-01-11 03:00"
$ sudo journalctl --since 09:00 --until "1 hour ago"

# 显示尾部的最新10行日志
$ sudo journalctl -n

# 显示尾部指定行数的日志
$ sudo journalctl -n 20

# 实时滚动显示最新日志
$ sudo journalctl -f

# 查看指定服务的日志
$ sudo journalctl /usr/lib/systemd/systemd

# 查看指定进程的日志
$ sudo journalctl _PID=1

# 查看某个路径的脚本的日志
$ sudo journalctl /usr/bin/bash

# 查看指定用户的日志
$ sudo journalctl _UID=33 --since today

# 查看某个 Unit 的日志
$ sudo journalctl -u nginx.service
$ sudo journalctl -u nginx.service --since today

# 实时滚动显示某个 Unit 的最新日志
$ sudo journalctl -u nginx.service -f

# 合并显示多个 Unit 的日志
$ journalctl -u nginx.service -u php-fpm.service --since today

# 查看指定优先级（及其以上级别）的日志，共有8级
# 0: emerg
# 1: alert
# 2: crit
# 3: err
# 4: warning
# 5: notice
# 6: info
# 7: debug
$ sudo journalctl -p err -b

# 日志默认分页输出，--no-pager 改为正常的标准输出
$ sudo journalctl --no-pager

# 以 JSON 格式（单行）输出
$ sudo journalctl -b -u nginx.service -o json

# 以 JSON 格式（多行）输出，可读性更好
$ sudo journalctl -b -u nginx.serviceqq
 -o json-pretty

# 显示日志占据的硬盘空间
$ sudo journalctl --disk-usage

# 指定日志文件占据的最大空间
$ sudo journalctl --vacuum-size=1G

# 指定日志文件保存多久
$ sudo journalctl --vacuum-time=1years
```

