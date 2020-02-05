/bin/bash -c "ls -l"

cmd->golang->pipe

pipe()创建2个文件描述符，fd[0]可读，fd[1]可写
fork() 创建子进程 fd[1]被继承到子进程
dup2() 重定向子进程 stdout/stderr到fd[1]
exec() 在当前进程内，加载并执行二进制程序


*/5 * * * * echo hello