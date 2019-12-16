
* `systemctl start 服务名(文件名)` 启动服务
* `systemctl status 服务名` 查看错误信息
* `systemctl restart 服务名` 重启服务
* `systemctl stop 服务名` 停止服务
* `systemctl enable 服务名` 开机启动服务
* `systemctl disable 服务名` 禁止开启启动服务
* `systemctl list-units --type=service` 查看所有已启动的服务
* `rpm -i xxx.rpm` 安装软件
* `rpm -e xxx` 卸载软件
* `rpm -qa | grep xxx` 查看rpm安装的软件

#### awk

> test:test2:test3   text.txt

* `FS` 输入分隔符 默认空格
* `OFS` 输出分隔符 默认空格
* `RS` 输入记录分隔符 默认换行符
* `ORS` 输出记录分隔符 默认换行符
* `NF` 当前记录中域的个数,即每行有多少列
* `NR` 已经读出的记录数
* `FILENAME` 当前文件的名字
* `awk '{print $0 }' text.txt` 打印一行记录 默认换行符为一行记录 -> **test:test2:test3**
* `awk '{print NR $0} text.txt'`  NR标示已读取的行数 从1开始,自动累加  -> **1test:test2:test3**
* `awk '{print $1 $3 }' text.txt` 表示打印第一个和第三个元素 默认以空格作为分隔符,因此该命令不打印
* `awk '$3="test3" {print $0}' text.txt` 如果分隔后的第三个元素等于指定的test3这打印这一行记录 `{print $0}` 可以不需要,默认打印当前行记录
* `awk '{print $0}' text.txt > text2.txt` 将结果重定向到text2文件中
* `awk -F: '{print $1}' text.txt` 以:作为分隔符 ->**test2**
* `awk -F'[ ]' '{print $1, $3}' text.txt` 以空格作为分隔符
* `awk -F'[,]' '{print $1, $3}' text.txt` 以逗号作为分隔符
* `awk -v FS=':' '{print $1 $3}' text.txt` 以分号作为分隔符 -> **test test3**
* `awk -v OFS="->" '{print $2, $3}' other.txt` -> **test2->test3**
* 

#### 日常命令

* `iotop -o` 当前正在写磁盘操作的所有进程ID信息
* `pidstat -u -x 3` 查看cpu使用率较高的进程
* `grep -v "^#" redis.conf` 屏蔽掉注释行信息
* `echo "" > nohup.out` 将文件清空
* `ps -p [PID] -o etime` 查看应用程序运行时长
