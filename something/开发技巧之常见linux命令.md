根据指定端口号找到占用该端口号的进程id：<br />
lsof -i:8081 | awk ‘NR==2{print $2}’ | xargs kill -9 <br />
netstat -anlp | grep 8081

查看git的安装目录
which git
