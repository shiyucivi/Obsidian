#### 1 ubuntu安装docker
在wsl上配置以使用clash代理：
在clash中开启允许局域网
在wsl中设置代理环境变量：
```shell
export hostip=$(ip route show | grep -i default | awk '{ print $3}')
export https_proxy="http://${hostip}:7890"
export http_proxy="http://${hostip}:7890"
curl https://www.google.com #检查是否成功
```
使用自动化脚本安装docker:
```shell
curl -fsSL https://get.docker.com -o install-docker.sh
sudo sh install-docker.sh
```
#### 2 认识docker
可以将docker看作宿主机机上一个小型的被隔离的虚拟机，只不过docker使用的是宿主机的系统内核。docker只需要维护自己的文件系统与程序。
==docker存在镜像和容器两个核心概念==。镜像类似于一个静态模板，容器则是套用模板产生的运行进程。
docker容器的网络、运行环境与文件系统与宿主机的互相隔离。一个docker容器的运行相关文件位于宿主机中的特殊位置。当docker容器停止运行时其文件仍然会存在等待下次启动。
#### 3 ubuntu中下载docker镜像
从docker官方拉取镜像：
```shell
docker pull docker.io/library/nginx:latest #docker.io是docker官方仓库，library代表官方命名空间，nginx代表镜像名称，latest表示下载最新版本
#可以简化为
docker pull nginx:latest #默认从官方仓库的官方命名空间中拉取nginx
```
查看本机上的所有docker镜像：
```shell
docker images #查看所有镜像
docker rmi nginx #删除nginx镜像
```
#### 4 离线安装docker与docker镜像


#### 5 创建和运行docker容器
使用docker run生成
```shell
docker run nginx #使用nginx镜像生成并运行一个docker容器，会占用当前窗口
docker run -d -name my-nginx nginx #-d表示后台运行nginx docker容器，将容器name命名为my-nginx
docker ps #查看正在运行的docker容器，每个运行中的容器有一个container id和一个name，通过这两个属性来重启或关闭容器
docker ps -a #查看所有系统中的docker容器
docker stop my-nginx #停止运行name为my-nginx的容器
docker start my-nginx #开始运行name为my-nginx的容器
docker rm -f bab72fc5cb82 #删除id为bab72fc5cb82的docker容器并强制停止容器进程
```
使用docker run生成的容器会保存在宿主机中，下次只需要用docker start运行容器即可。重复运行docker run会生成多个容器。
默认情况下docker容器的网络与宿主机互相隔离，无法通过宿主机访问docker容器内的网络。可以在命令行中添加一个-p参数来对docker容器的端口映射到宿主机的一个端口：
```shell
sudo docker run -d -p 80:80 nginx #将nginx容器的80端口映射到宿主机的80端口
```
这样访问虚拟机的ip+80端口，就可以进入nginx默认页面

#### 6 进入容器
```shell
docker exec -it my-nginx /bin/bash #进入到my-nginx容器的命令行终端
docker exec -it my-nginx vi /etc/nginx/nginx.conf #不进入容器，只在容器中执行单独一条命令
exit #退出容器命令行终端
```
#### 7 挂载卷
当我们删除一个docker镜像时，镜像中所有文件都会被删除。可以通过将镜像中的文件与外部宿主机中的文件进行绑定，就可以避免这个问题。
挂载卷通常使用两种方法：
##### 7.1 绑定挂载
使用-v参数将宿主机中的文件与docker容器中的两个目录进行绑定：
```shell
docker run -d -p 80:80 -v /home/user1/html:/usr/share/nginx/html nginx
#/home/user1/html为宿主机目录，/usr/share/nginx/html为容器目录
```
这种挂载方式会将宿主机的html目录覆盖掉容器的html目录
##### 7.2 Volume命名卷挂载
使用docker volume create命令可以创建一个挂载卷目录，然后使用绑定挂载的方法将挂载卷同docker容器中的一个目录互相绑定
```shell
docker volume create docker_html  #创建了一个名为docker_html的挂载卷目录
docker run -d -p 80:80 -v docker_html:/usr/share/nginx/html nginx  #将docker_html挂载卷同nginx容器的/usr/share/nginx/html绑定
```
查看挂载卷目录的位置：
```shell
docker volume inspect docker_html #返回一个json文件，代表docker_html挂载卷的配置文件，其mountpoint字段为挂载卷真实位置
docker volume list #列出所有挂载卷
docker volume rm docker_html #删除docker_html挂载卷
docker volume prune -a #删除所有未使用的挂载卷
```
#### 8 docker-compose搭建docker容器
使用docker-compose命令与yaml配置文件来创建一个nginx容器
##### 8.1 创建一个目录作为容器目录：
```shell
cd ~ #进入当前用户目录
mkdir docker-nginx
cd docker-nginx
mkdir html
touch docker-compose.yml nginx.conf
```
##### 8.2编辑docker-compose.yml：
```yaml
version '3.8' #docker-compose文件格式版本
service:  #
  nginx:
    image: nginx:latest #挂载容器的所使用的镜像
    container_name: my-nginx #docker容器的name
    ports:
	    - "8080:80" #格式为"宿主机端口:容器端口"
    volumes:  #使用volumes挂载，挂在格式：宿主机目录:容器目录
        - ./nginx.conf:/etc/nginx/nginx.conf:ro #ro表示容器只读
        - ./html:/usr/share/nginx/html:ro
        - ./logs:/var/logs/nginx #日志目录挂载  
    restart: unless-stopped #自动重启容器
```
##### 8.3编辑nginx.conf：
```nginx
events {
    worker_connections 1024;
}
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log;
    sendfile        on;
    keepalive_timeout 65;
    server {
        listen 80;
        server_name localhost;

        location / {
            root /usr/share/nginx/html;
            index index.html index.htm;
        }
    }
}
```
创建完毕后运行：
```shell
docker-compose up -d #根据当前目录下的yaml配置文件自动下载镜像，并生成和运行容器
```
这个命令也可以用于修改yaml配置后重新加载容器。
关闭或重启docker容器：
```shell
docker-compose down #关闭当前目录下的docker容器
docker stop my-nginx #关闭name为my-nginx的容器
docker restart my-nginx #重启name为my-nginx容器
```

#### 10 docker desktop
把docker desktop中的镜像推送到一台远程服务器：
