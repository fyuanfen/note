pm2 这个进程管理器是越来越好用了，最新的版本已经开始支持 docker，不过我不看好，docker 镜像内的 node 进程挂掉就应该让集群管理器去处理，简单有效，并且也准守 Fail fast 这个准则。

正文开始，我把最近用到的几个点总结下，（之前的 #4 也有一些总结） 。

使用 yml 配置文件
还在使用 json 文件来配置么？放弃吧，yml 更友好，更具可读性，配置项也跟 json 保持一致。
http://pm2.keymetrics.io/docs/usage/application-declaration/

设置最大内存
不知道你是否了解 node.js 的内存限制，一般默认的大小为 1.76G（64bit 机器），一旦超过，就会自动崩溃，并且在\*nix 平台上生成 core.pid 文件，所以有时候你会看到 working dir 中生成了一堆的 core 文件，不妨用 file 命令看下，相信你会发现都是 node 进程生成的。

解决方案很简单，pm2 设置 max-memory-restart 参数，这个值必须小于 1.76G。当然了，如果你确实需要大内存，可以设置 node 参数：--max-old-space-size=xxxM，这个值可以随便设置，但建议多测试后再决定，可能大内存限制下会出些问题。

拓展一下
关于 core 文件，本来是用来调试段错误的，但是如果你不是写 C++ AddOn，或者调试什么 node 的 bug，那么可以禁用掉：用 Linux 本身的方案来解决。

限制大小：ulimit -c filesize（单位是 kb），超过会被裁减；
更改生成的 core 路径以及文件名；
默认情况下，/proc/sys/kernel/core_pattern 为 core， /proc/sys/kernel/core_uses_pid 为 1，即生成的 core 文件默认为 working dir 中的 core.pid。
如何修改：

%% 单个%字符
%p dump 进程的进程 ID
%u dump 进程的用户 ID
%g dump 进程的组 ID
%s 导致 core dump 的信号
%t core dump 的时间
%h 主机名
%e 程序文件名

echo '/data/core-files/core.%t.%p' > /proc/sys/kernel/core_pattern
禁用 pid 拓展名

echo 0 > /proc/sys/kernel/core_uses_pid
当然了，都是临时的，永久修改需要编辑/etc/sysctl.conf，加入如下两行：
kernel.core_pattern = /data/core-files/core.%t.%p
kernel.core_uses_pid = 0

自启动脚本
不知道你是否每次重启之后是怎么启动 pm2 的，但是如果不是自动启动的就有点落后了。如果用 init.d 或者 systemd，那也不错，也就维护有点小麻烦，每次调整都得更新脚本。其实 pm2 已经有了相关的命令来帮你完成这一系列的操作：

生成自启动脚本
pm2 startup centos（ubuntu, centos, redhat, gentoo, systemd, darwin, amazon 之一，具体请看文档）
生成 dump.pm2 文件
pm2 save（或者 dump）
经过这一步之后，机器重启之后，就能保证 pm2 自动启动了。

crash 恢复
pm2 resurrect
不过，这个命令不适合手工去用的，可以设置个 crontab 脚本，一旦检测到 pm2 daemon 的进程不存在，则执行这个命令。
