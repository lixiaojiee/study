根据指定端口号找到占用该端口号的进程id：
lsof -i:8081 | awk ‘NR==2{print $2}’ | xargs kill -9

netstat -anlp | grep 8081