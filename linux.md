
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