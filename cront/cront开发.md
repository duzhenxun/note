/bin/bash -c "ls -l"

cmd->golang->pipe

pipe()创建2个文件描述符，fd[0]可读，fd[1]可写
fork() 创建子进程 fd[1]被继承到子进程
dup2() 重定向子进程 stdout/stderr到fd[1]
exec() 在当前进程内，加载并执行二进制程序

Cron基本格式
|分|时|日|月|周|shell命令
|-|-|-|-|-|-|
|*/5|*|*|*|*|echo hello >/tmp/x.log|
|1-5|*|*|*|*|echo /usr/bin/python /data/x.py|
|0|10,22|*|*|*|echo hello|tail -1|

go开源Cronexpr库
Parse() 解析与校验Cron表达式
Next() 根据当前时间，计算下一次调度时间

