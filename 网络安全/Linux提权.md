
## 了解系统环境

- 系统版本：使用的是哪种发行版？（Ubuntu、Debian、FreeBSD、Fedora、SUSE、Red Hat、CentOS 等）

- `内核版本`：特定内核版本中可能存在针对漏洞的公开漏洞利用程序。

- `运行中的服务`：了解主机上运行的服务非常重要，尤其是以 root 权限运行的服务。以 root 权限运行的配置错误或存在漏洞的服务很容易被利用来提升权限。

## 前置检查

### 列出进程

```shell
└─$ ps aux | grep root                                                                             
root           1  0.1  0.0  24168 14160 ?        Ss   13:18   0:01 /sbin/init splash
root           2  0.0  0.0      0     0 ?        S    13:18   0:00 [kthreadd]
root           3  0.0  0.0      0     0 ?        S    13:18   0:00 [pool_workqueue_release]
root           4  0.0  0.0      0     0 ?        I<   13:18   0:00 [kworker/R-kvfree_rcu_reclaim]
root           5  0.0  0.0      0     0 ?        I<   13:18   0:00 [kworker/R-rcu_gp]
root           6  0.0  0.0      0     0 ?        I<   13:18   0:00 [kworker/R-sync_wq]
root           7  0.0  0.0      0     0 ?        I<   13:18   0:00 [kworker/R-slub_flushwq]
root           8  0.0  0.0      0     0 ?        I<   13:18   0:00 [kworker/R-netns]
root          10  0.0  0.0      0     0 ?        I<   13:18   0:00 [kworker/0:0H-events_highpri]

```


### 查看主目录

检查各个用户目录，并检查`.bash_history`之类的文件是否可读，是否包括特殊命令，并检查是否可以获得用户 SSH 密钥。

```shell
ls /home
```

```shell

└─$ ls -la /home/killua 
total 332
drwx------ 39 killua killua  4096 Apr 30 13:27 .
drwxr-xr-x  3 root   root    4096 Mar 28 19:44 ..
-rw-r--r--  1 killua killua   220 Mar 28 19:44 .bash_logout
-rw-rw-r--  1 killua killua   147 Mar 28 21:37 .bash_profile
-rw-r--r--  1 killua killua  5755 Apr  3 16:29 .bashrc
-rw-r--r--  1 killua killua  3526 Mar 28 19:44 .bashrc.original
drwx------  8 killua killua  4096 Apr 29 18:49 .BurpSuite
drwxrwxr-x 24 killua killua  4096 Apr 30 13:27 .cache
drwxrwxr-x  4 killua killua  4096 Apr  9 16:21 .cargo
drwxr-xr-x 30 killua killua  4096 Apr 30 13:18 .config
drwx------  3 killua killua  4096 Mar 28 20:55 .dbus

```

### SSH目录

```shell
ls -l ~/.ssh

total 8
-rw------- 1 mrb3n mrb3n 1679 Aug 30 23:37 id_rsa
-rw-r--r-- 1 mrb3n mrb3n  393 Aug 30 23:37 id_rsa.pub

```

## bash历史记录

用户可能在命令行中将密码作为参数传递，操作 `git` 仓库，设置 `cron` 任务等等。

```shell
└─$ history
    1  whoami
    2  su root
    3  sudo root
    4  su root
    5  sudo apt 
    6  sudo apt update
    7  sudo apt upgrade
    8  sudo apt autoremove
    9  sudo apt install code-insiders

```

### passwd和shadow

 可能有机会获得用户的哈希值。

```shell
cat /etc/passwd
cat /etc/shadow
```


### cron jobs

计划任务通常用于执行维护和备份任务。结合其他错误配置（例如相对路径或弱权限），它们可以在计划的 Cron 作业运行时利用来提升权限。

```shell
ls -la /etc/cron.daily/
```


### 文件系统驱动器

如果可以安装额外的驱动器或存在未安装的文件系统，可能可以利用提升权限的敏感文件、密码或备份。

```shell
└─$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme0n1     259:0    0 953.9G  0 disk 
├─nvme0n1p1 259:1    0   100M  0 part /boot/efi
├─nvme0n1p2 259:2    0    16M  0 part 
├─nvme0n1p3 259:3    0   200G  0 part 
├─nvme0n1p4 259:4    0 557.5G  0 part 
├─nvme0n1p5 259:5    0   900M  0 part 
├─nvme0n1p6 259:6    0 185.4G  0 part /
└─nvme0n1p7 259:7    0   9.9G  0 part [SWAP]

```

### 可写目录与文件

```shell

find / -path /proc -prune -o -type d -perm -o+w 2>/dev/null

find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null

```


