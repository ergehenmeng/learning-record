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

