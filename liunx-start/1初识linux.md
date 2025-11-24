#### 1.1 安装linux

linux提供了调度各种硬件的内核，而ubuntu等发行版则用这些内核中的接口封装了一些系统程序，内核+系统程序组成了一个完整的操作系统。

在windows通过wsl上安装linux：
```shell
wsl --install
wsl --set-default-version 2
wsl --install -d Ubuntu-22.04
```
#### 1.2 安装SSH工具并远程连接

ubuntu中默认安装了apt作为软件包管理工具。使用`apt install`安装ssh软件
```shell
sudo apt update
sudo apt install ssh-server -y #-y表示自动对安装选项问答表示yes
```
其中`sudo apt update`是为了更新系统中所保存的一个软件包列表，确保所下载的软件为最新版本。==linux会将软件配置默认放在/etc目录下，执行文件放在/usr/bin下==。去修改ssh配置：
```bash
sudo nano /etc/ssh/ssh_config
```
nano编辑器会默认进入编辑模式，使用ctrl+O进行保存修改，使用ctrl+X退出编辑器。
```shell
# 允许密码登录
PasswordAuthentication yes
# 允许 root 登录（可选，不推荐生产环境）
PermitRootLogin yes
# 监听所有接口（关键！WSL 默认只监听 127.0.0.1）
# 找到 #ListenAddress 0.0.0.0，取消注释或添加：
ListenAddress 0.0.0.0
# 确保使用默认端口（或自定义）
Port 22
```
然后启动ssh服务：
```shell
sudo systemctl enable --now ssh #启动ssh服务
ps aux | grep sshd  #查看ssh服务是否在运行
ss -tuln | grep :22 #查看22端口上的服务
```
systemctl命令为系统控制命令，enable为其一个子命令表示设置ssh开机自启，--now表示设置开机自启后立即启动ssh。也可以直接找到ssh的可执行文件并运行：
```shell
sudo /usr/sbin/sshd
```
ps aux命令用于列出所有运行中的进程，aux代表所有用户以及规定输出格式，==| 表示管道模式，将|前的命令的输出作为后面命令的输入==，==grep是文本搜索工具，在输出中查找包含字符串sshd的行==。
ss命令用于列出当前所有占用端口的scoket ，-tuln表示只显示tcp、udp。

启动完成之后查看本机的ip：
```shell
hostname -I
172.26.127.68
```
除了`hostname -I`，也==推荐使用ip -addr来查看本机ip信息==。
然后在另一台机器中的使用命令连接：
```shell
ssh civi@172.26.127.68 #其中civi为连接使用的用户名
```
#### 1.3 liunx命令

linux中常见的命令格式为
`command [subcommand] [paramater] [-options]`
例如`apt install ssh -y`，apt为命令，install为子命令，ssh为参数，y为执行选项。
一个命令可以有多个参数，可以针对每个参数选择一个单独的执行选项
一个命令本质上是执行命令所在的二进制可执行文件
大部分命令可以通过`[命令] --help`查看参数和选项说明
#### 1.4 本章命令总结
```shell
sudo apt update #更新apt软件表
sudo apt install ssh -y #安装ssh
sudo system start ssh #启动ssh
sudo systemctl enable --now ssh #启动并开机自启ssh
ps aux | grep sshd #查找所有进程，并筛选出sshd进程
ss -tuln | grep :22  #查找socket，并筛选出监听22的socket
hostname -I  #本机ip
ip addr  #本机网络地址详细信息
```