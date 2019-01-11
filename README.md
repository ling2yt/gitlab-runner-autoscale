# gitlab-runner-autoscale
# ubuntu16.04

# ==============================  prepare  =================================

## 安装docker
```
curl -sSL https://get.docker.com/ | sh
```
More：https://docs.docker.com/install/

## 安装docker-machine(仅docker-machine的executor需要安装)
```
base=https://github.com/docker/machine/releases/download/v0.16.0 &&
 curl -L $base/docker-machine-$(uname -s)-$(uname -m) >/tmp/docker-machine &&
 sudo install /tmp/docker-machine /usr/local/bin/docker-machine
 ```
More：https://docs.docker.com/machine/install-machine/
## 主机shell方式运行gitlab-runner
### 安装gitlab-runner
```
bcurl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
sudo apt-get install gitlab-runner
```
More：https://docs.gitlab.com/runner/install/linux-repository.html

### registry gitlab-runner

```
sudo gitlab-runner register
##enter the gitlab-runner info
##select the Runner executor (shell, docker, docker+machine)
```
More：https://docs.gitlab.com/runner/register/index.html#gnu-linux

## 主机docker方式运行gitlab-runner
### 下载gitlab-runner镜像
```
sudo docker pull gitlab/gitlab-runner
```
### 启动容器
```
sudo docker run -d --name gitlab-runner --restart always -v /srv/gitlab-runner/config:/etc/gitlab-runner -v /var/run/docker.sock:/var/run/docker.sock gitlab/gitlab-runner:latest
```
/etc/gitlab-runner中存放gitlab-runner配置文件，/var/run/docker.sock表明gitlab-runner使用宿主机docker资源

### 注册gitlab-runner
```
sudo docker exec -it gitlab-runner gitlab-ci-multi-runner register
##enter the gitlab-runner info
##select the Runner executor(shell, docker, docker+machine)
```
More: https://docs.gitlab.com/runner/register/index.html#gnu-linux
如有需要将gitlab-runner加入docker用户组
```
usermod -a -G docker gitlab-runner
```
# ========================== configure gitlab-runner ====================================

## shell
使用shell方式运行ci，

代码存放路径为gitlab-runner运行机器或gitlab-runner运行容器中的/home/gitlab-runner,配置多于1个concurrent的情况下，每个job会在/home/gitlab-runner/builds/{{.runnerId}}/{{.JobId}}/的路径下存放代码

不需大批量修改config.toml配置，基本关注concurrnet, limit标签即可

More: https://docs.gitlab.com/runner/configuration/autoscale.html#how-concurrent-limit-and-idlecount-generate-the-upper-limit-of-running-machines

vm和job关系：

## docker
配置executor为docker，使用docker

需要重点关注config.toml中的[runners.docker]和[runners.cache]标签，定义了执行ci过程中使用的镜像、cache等配置

代码下载路径为 gitlab-runner-helper镜像启动的容器中的/builds文件夹, gitlab-runner-helper image 可以通过config.toml中的helper_image标签配置

保存代码，使用fetch的方式节省同步代码时间的方案：使用config.toml配置文件中的volume标签挂载/builds文件夹到宿主机

vm和job关系：

docker+machine
docker+machine的方式支持动态开启aws EC2 instance，spot instance。

流程：



vm和job关系：

## commons：

新启动的instance cert保存在 /root/.docker/machine/machines/{{.machineId}}中，可以用来ssh登录新申请的instance，

查看已管理的docker-machine：docker-machine ls，通过设置 eval $(docker-machine env {{.machineId}})将docker环境指向对应的instance中的docker环境

支持保存ci中间状态（不同stage之间需要共享的数据）到cache，支持自动弹性申请 EC2／spot instance的gitlab-runner配置文件: /etc/gitlab-runner/config.toml ( download: config.toml )

优点：根据配置,自动启动对应aws instance，spot instance定时自动回收

缺点：启动新instance时，新实例中所有环境重新部署，代码重新git clone，时间代价较大

More：

runner autoscale in aws： https://docs.gitlab.com/runner/configuration/runner_autoscale_aws/index.html



# =============================== end =====================================

## 注：

可以通过ps -aux | grep gitlab-runner 查看gitlab-runner相关进程和默认启动指令

gitlab-runner的默认启动命令使用--user标签定义gitlab-runner的默认执行用户为gitlab-runner

gitlab-runner启动指令中的--syslog标签，指定gitlab-runner日志会写入系统日志，可以在/var/log/syslog查看报错等日志信息

ci中同步代码的方式在CI / CD Settings → General pipelines中配置，如果配置为git fetch但实际代码不存在，会在同步时改为git clone

CI／CD Settings中的gitlab-runner中配置tag，使得job根据tag分配到runner

cache中存储的内容在ci.yml中配置：
```
#image: ubuntu:16.04
stages:
#  - test2
  - test
  - testbuild
#  - test2
#  - test3
cache:
  paths:
  - /builds/regitory/Project/
```
