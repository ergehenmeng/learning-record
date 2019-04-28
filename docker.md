##docker

* 镜像`Image`
* 容器`Container`
* 仓库`Repository`


**docker run -d -p 80:80 --restart=always nginx:latest**

`run` 运行某个镜像

`-d` 后台运行

`-p 80:80` 指定端口映射 宿主机的80端口映射到容器的80端口 

`--restart=always` 重启模式,设置为always每次启动docker都会启动nginx容器

`-v ~/docker-demo/nginx-htmls:/usr/share/nginx/html` 用来提供数据挂载功能,类似springBoot的spring.config.location一样

 **docker exec -it 4591552a4185 bash**

`exec` 对容器执行某些操作

`-it` 让容器可以接受标准输入并分配一个伪tty

`43436153a117` 刚才启动nginx容器唯一标示符

`bash` 指定交互的程序bash

`docker kill 43436153a117` 结束运行nginx


## Docker
* docker search <镜像名称.版本号>  从镜像服务器中查找镜像
* docker pull <镜像名: tag> 拉取镜像
* docker build -t <镜像名><Dockerfile路径> 创建镜像,需编写脚本
* docker images 查看所有的镜像
* docker rmi <镜像名> 删除镜像
* docker run -name 容器名 -d -p 内部端口号:外部端口号 镜像名<.版本号>
  * -a 指定标准输入内容类型
  * -d 后台运行容器并返回id
  * -i 以交互模式运行容器
  * -t 为容器重新分配一个伪输入终端 通常与-i同时使用
  * --name 为容器指定一个名称
  * --dns 使用容器的dns服务器 默认与宿主一致
  * --dns-search 指定容器dns搜索域 默认与宿主一致
  * -h 指定容器的hostname
  * -e username设置环境变量
  * --env-file=[]从指定文件读取环境变量
  * --cpust="0-2"
* docker logs -f <容器名或id>
* docker ps 查看正在运行的容器
* docker rm $ (docker -a -q) 删除所有容器
* docker rm <容器名或ID> 删除一个容器
* docker stop <容器名或ID> 停止一个容器
* docker kill <容器名或id> 杀死一个容器
* docker start <容器名或id> 启动一个容器
* `docker pull registry.docker-cn.com/library/redis` 国内镜像加速
### docker安装

* 必须内核版本高于3.10 `uname -r` 查看
* `yum update` 将yum包更新到最新
* `yum remove docker docker-common docker-selinux docker-engine` 卸载docker旧版本
* `yum install -y yum-utils device-mapper-persistent-data lvm2` 安装yum依赖的软件包
* `yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo` 设置docker的yum源
* `yum list docker-ce --showduplicates | sort -r ` 查看仓库中所有的docker版本
* `yum install docker-ce` 安装最新的docker稳定版
* `yum install docker-ce-17.12.0.ce ` 安装指定的docker版本
* `systemctl start docker` `systemctl enable docker` 开机启动docker服务

