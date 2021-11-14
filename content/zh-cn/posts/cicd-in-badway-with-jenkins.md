---
title: "完全不推荐的方式来使用 Jenkins + GitLab 构建 CI/CD 系统"
date: 2021-10-22T17:59:31+08:00
description: 除了某个比赛的选手之外千万不要看，不要学
categories: ['DevOps']
tags: ['gitlab','devops','jenkins','gitlab']
---

除了某个比赛的选手之外千万不要看，不要学

根据某两位数字所的实验案例得来此文，没办法，谁让我要去比赛呢，

<!--more-->

GitLab 其实自己就有 CI/CD 系统，大可不必使用 Jenkins，毕竟我日常在用的，

Anyway，实验环境如下

## 实验环境

除后端存储有专用存储机外，所有环境跑在 vSphere 虚拟化平台中，均为虚拟机。部分配置为我自家网络配置，可随机应变

| FQDN                 | 服务器                                         | 系统环境                              | 配置                        |
| -------------------- | ---------------------------------------------- | ------------------------------------- | --------------------------- |
| master.ddupan.top    | Kubernetes 1.18 社区版，单节点                 | CentOS 7.5                            | 4 vcpus，8G mem             |
| jenkins.ddupan.top   | Docker-CE，运行 Jenkins 与 GitLab              | CentOS 7.5                            | 8 vcpus 8G mem              |
| har.ddupan.top       | Docker-CE，运行 Harbor 容器仓库                | CentOS 7.5                            | 4 vcpus 8G mem              |
| repo.ddupan.top      | Apache httpd，提供内网 YUM Repository          | CentOS 7.9                            | 1 vcpus 1G mem              |
| nas.ddupan.top       | TrueNAS SCALE，提供所有内容的后端存储          | TrueNAS SCALE beta （基于 Debian 11） | 物理服务器，40 core 64G mem |
| homesrvdc.ddupan.top | 运行 DHCP 与 DNS 服务，提供动态更新的 DNS 服务 | Windows Server 2019                   | 4 vcpus 4G mem              |

DHCP 与 DNS 为可选项，如内网不存在 DNS 则需要自行维护每台虚拟机的 `/etc/hosts` 文件

某数字所提供的资源包存放在 `nas` 主机 的`/mnt/main/files/chinaskills/`目录下，且使用 NFS 与 SMB 双协议进行 NAS 存储共享。

## 番外篇: HTTP 版 YUM 源搭建

以下为 CentOS 上使用 Apache httpd 搭建网络 YUM 源的方法。

安装系统与初始配置不再赘述。

如使用 NFS 共享挂载，避免麻烦需要关闭 SELinux

安装 `nfs-utils`以挂载 NFS 存储

```sh
yum -y install nfs-utils

# 挂载存储到 /mnt

mount nas:/mnt/main/files/chinaskills /mnt

# 安装 httpd
yum -y install httpd
# 建立到默认 Web 根目录的软连接
ln -s /mnt /var/www/html/chinaskills
# 更改软连接所属
chown apache:apache /var/www/html/chinaskills
# 删除默认欢迎页面，可选
rm -rf /etc/httpd/conf.d/welcome.conf
# 启动服务
systemctl start httpd
systemctl enable httpd
# 防火墙放行 httpd 服务
firewall-cmd --add-service http --permanent
firewall-cmd --reload
```

## Harbor 仓库搭建

 某所提供了安装脚本，但我打算重新整理过程，实际上也非常简单。

配置 YUM 源为内网某所提供的源，以及 CentOS 光盘源，以下均一致不再赘述

```sh
# 安装 Docker-CE
yum -y install docker-ce

# 第一次启动 Docker Daemon
systemctl start docker
systemctl enable docker

# 修改 Docker Daemon 配置
tee /etc/docker/daemon.json < EOF
{
	insecure-registries: ['0.0.0.0/0'],
	exec-opts: ['native.cgroupdriver=systemd']
}
EOF

# 重启 Docker Daemon
systemctl restart docker

# 从 repo 下载所需文件
wget http://repo/chinaskills/paas/docker-compose/v1.25.5-docker-compose-Linux-x86_64
mv ./v1.25.5-docker-compose-Linux-x86_64 /usr/bin/docker-compose
chmod +x /usr/bin/docker-compose

wget http://repo/chinaskills/paas/harbor/harbor-offline-installer-v2.1.0.tgz
tar -xvf harbor-offline-installer-v2.1.0.tgz

# 准备部署 Harbor
cd harbor
cp harbor.yml.tmpl harbor.yml
vim harbor.yml
# 在此配置文件中修改主机名为实际的主机名或 IP 地址，不能使用 127.0.0.1
# 且注释 https 部分配置

# 确认 docker 与 docker-compose 已可以使用，开始部署 Harbor
./prepare
./install.sh
```

将提供的离线镜像包全部上传到 Harbor，供下面使用

```sh
# 挂载 NFS 后端存储
yum -y install nfs-utils
mount nas:/mnt/main/files/chinaskills/paas /mnt
cd /mnt

# 导入并上传镜像
for i in $(ls images | grep tar)
do
	docker load -i images/$i
done
docker login har.ddupan.top -u admin -p Harbor12345
for i in $(docker images ls --format '{{.Repository}}:{{.Tag}}' | grep -v goharbor)
do
	docker tag $i har.ddupan.top/library/$i
	docker push har.ddupan.top/library/$i
done
```

使用域名或 IP 可以登录 Harbor，默认用户名 `admin`，密码`Harbor12345`

## Kubernetes 部署

同上，某数字所提供了一键脚本，同样将手动步骤整理如下

建议你们都看一下，因为某所的脚本不完美，有一些地方需要修正

```sh
# 安装 Docker-CE 作为容器运行时
yum -y install docker-ce

# 第一次启动 Docker Daemon
systemctl start docker
systemctl enable docker

# 配置 Docker Daemon
tee /etc/docker/daemon.json < EOF
{
	insecure-registries: ['0.0.0.0/0'],
	exec-opts: ['native-cgroupdriver=systemd']
}
EOF

# 关闭系统交换分区
swapoff -a
sed -i 's/.*swap.*/#&/' /etc/fstab

# 配置系统模块以解决 RHEL 系系统网桥流量绕过 iptables 问题
# 某所脚本未将系统模块加载写进开机自动加载，重启会有些问题
echo br_netfilter > /etc/modules.load.d/99-k8s.conf
systemctl restart systemd-modules-load.service

# 配置内核参数，使网桥流量经过 iptables
echo 'net.bridge.bridge-nf-iptables = 1' >> /etc/sysctl.conf
echo 'net.bridge.bridge-nf-ip6tables = 1' >> /etc/sysctl.conf
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
sysctl -p

# 安装 Kubernetes 工具

yum -y install kubeadm kubelet kubectl

systemctl start kubelet 
systemctl enable kubelet

# 启动 Kubernetes 集群
kubeadm init --image-repository har.ddupan.top/library --pod-network-cidr=10.244.0.0/16 --kubernetes-version=1.18.1
mkdir $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 安装 flannel 网络插件
wget http://repo/chinaskills/paas/yaml/flannel/kube-flannel.yaml
sed -i 's/quay.io\/coreos/har.ddupan.top\/library/g' kube-flannel.yaml
kubectl apply -f kube-flannel.yaml

# 我不使用 dashboard，因此不安装
# 配置 master node 以便在单节点环境下使 K8s 可将工作负载调度到 master 节点上
kubectl taint nodes --all node-role.kubernetes.io/master-

# 配置 kubectl 命令补全
yum -y install bash-completion
kubectl completion bash > /etc/bash_completion.d/kubectl
source /usr/share/bash-completion/bash_completion
```

## Jenkins 与 GitLab 搭建

某所提供的 Jenkins 与 GitLab 是容器版本，大大缩短了搭建时间与难度

Jenkins 需要将主机的 Docker Daemon Unix Socket 挂载进容器以使用 Docker，Kubernetes 我也采取了此方式来减少搭建难度

GitLab 可以将配置文件挂载到主机以对配置进行永久存储，但我选择不挂载，因为就此环境而言完全无必要

两者同时启动需要 5G 多的内存，而 Jenkins 构建时总共需要 6G 多的内存，因此如果实验环境内存不足，则容器会异常退出，如电脑配置较低，则不要在此节点上关闭内存交换

```sh
# 安装 Docker-CE，不再赘述
yum -y install docker-ce
systemctl enable --now docker
tee /etc/docker/daemon.json < EOF
{
	insecure-registries: ['0.0.0.0/0'],
	exec-opts: ['native.cgroupdriver=systemd']
}
EOF

systemctl restart docker

# 安装 kubectl 工具
yum -y install kubectl

# 从 master 主机上获取 Kubeconfig
mkdir ~/.kube
scp root@master:/root/.kube/config ~/.kube/config

# 启动 Jenkins 容器，从 Harbor 镜像中拉取

docker run -d --name jenkins \
	-v $(which docker):/usr/bin/docker \ # 将本机的 docker-ce-cli 二进制程序挂载进容器
	-v $(which kubectl):/usr/local/bin/kubectl \ # 将本机的 kubectl 二进制程序挂载进容器
	-v /var/run/docker.sock:/var/run/docker.sock \ # 将本机的 Docker Unix Socket 挂载进容器以在容器中使用 Docker
	-v /home/jenkins_home:/var/jenkins_home \ # 将 Jenkins 工作目录挂载进本机
	-v /root/.kube/config:/root/.kube/config \ # 将 Kubeconfig 挂载进容器
	-v /root/.m2:/root/.m2 \ # 将 Apache Maven 本地镜像挂载进主机
	-u root \ # 指定容器用户为 root
	-p 8080:8080 \ # 映射 8080 端口到本机
	har.ddupan.top/library/jenkins:2.262-centos
	
# 启动 GitLab 容器
# 某所未在资源包中提供此镜像，按以前的尿性决定了版本号，并从 Docker Hub 中拉取，需要互联网连接

docker run -d --name gitlab -p 80:80 -p 443:443 gitlab/gitlab-ce:12.9.2-ce.0
# GitLab 启动较慢
```

 

## 配置 Jenkins 与 GitLab

### 配置 Jenkins

使用网页浏览器访问 `jenkins.ddupan.top:8080` 打开 Jenkins Dashboard

可使用 `docker logs jenkins` 或 `cat /home/jenkins_home/secrets/initialAdminPassword` 查看解锁密码

选择`安装推荐的插件`继续，安装向导结束后需要手动安装所需插件

新增一个管理员账户即可

正常来说，完整的容器工作流应当在 Jenkins 进行 Build 操作时也使用容器，但某所并没有提供所需容器，给出的解决方案是手动将 Apache Maven 部署进 Jenkins 容器中。

```sh
# 从 repo 上下载 Apache Maven
wget http://repo/chinaskills/paas/ChinaSkillMall/apache-maven-3.6.3-bin.tar.gz
tar -xvf apache-maven-3.6.3.tar.gz
cp -r apache-maven-3.6.3/ /home/jenkins_home/
mount nas:/mnt/main/files/chinaskills/paas /mnt
rm -rf /home/jenkins_home/plugins/*
cp -r /mnt/plugins/* /home/jenkins_home/plugins/


# 进入容器后继续操作
docker exec -it jenkins bash
# 此处开始为容器内部
mkdir /opt/maven/ -p
mv /var/jenkins_home/apche-maven-3.6.3/ /opt/maven/
exit
# 此处已退出容器
docker restart jenkins
```

打开 Jenkins Dashboard，配置 Jenkins

使用创建的管理员账户登录

点选左侧的系统管理，选择全局工具配置

找到 Maven，选择新增 Maven

Name 任选，但后续要用到，取消 `自动安装`的勾选，并填入容器内 MAVEN_HOME 的路径，此处为 `/opt/maven/apahce-maven-3.6.3/`

工具即配置完成，保存设置

还需进行安全设置，点选全局安全配置，授权策略中选择 `任何用户可以做任何事（没有任何限制）`

某数字所没有提供所需的 Maven 源离线包，但比赛肯定会提供，如果提供，则将离线包复制到 `/root/.m2`中

### 配置 GitLab

访问 `http://jenkins` 打开 GitLab，设置 `root` 账户的密码并登录

默认界面为英文，但 GitLab 实际有部分中文翻译，修改方法为点选右上角的 root 用户，选择 Settings，左侧点选 Preference，最下面的 Localization 设置中可以设置界面语言为简体中文

接下来新建 Project，回到首页，选择 `Create A Project`

名称可任选，但后续会用到，我选择填写 `ChinaSkillProject`

可见性一定要选择公开，这样后续无需为 Jenkins 配置 Personal Access Token

其他可以不填，直接新建项目即可

### 上传源码

如果有一点开发经验则一定不会没用过 Git，这里我也会大概说一下命令的作用，诚心诚意推荐阅读 《Pro Git》

```sh
cp -r /mnt/ChinaSkillProject ~/
cd ~/ChinaSkillProject
yum -y install git # 如果未安装

git config user.name administrator # 配置 Commit 时附带的用户名，仅对当前目录的项目有效
git config user.email admin@example.com # 配置 Commit 时附带的邮箱，仅对当前目录的项目有效
git add . # 将当前目录所有文件加入暂存区
git commit -m "Initial Commit" # Commit 暂存区中的文件，并附带 Commit 附言
git remote remove origin # 删除项目中原来附带的名为 origin 的远端存储库
git remote add http://jenkins.ddupan.top/root/chinaskillproject.git # 添加远端存储库，地址根据项目库名与用户有变化
git push -u origin master # 将当前分支代码推送至 origin 远端存储库的 master 分支，并将此次推送行为设置为默认推送行为
```

### 在 Jenkins 中添加流水线

在 Jenkins Dashboard 中选择新建项目，名字任选，选择创建流水线

在触发器行为中选择 `Build when a change is pushed to GitLab`，并记下后面的 WebHook 地址，后续需将其配置进 GitLab

流水线的部分，可以在 Git 项目中写 Jenkinsfile，也可以在 Jenkins 中直接写 Pipeline 脚本。

我选择直接写 Pipeline 脚本，脚本如下

```groovy
pipeline {
    agent any
    tools {
      	// 此处的名称为在 Jenkins 中设置 Maven 工具时所写的名称
        maven '3.6.3' 
    }
    stages{
        stage("Clone") {
            steps {
              	// 由于是公开项目，因此可直接拉取代码无需用户认证
                git url: "http://jenkins.ddupan.top/root/chinaskillproject.git"
            }
        }
        
        stage("Build") {
            steps {
              	// 使用 Maven 构建 jar 包，跳过测试环节
                sh 'mvn package -DskipTests'
            }
        }
        
        stage("Image Build") {
            steps {
              	// 构建所需的 docker 镜像
                sh '''docker build -t har.ddupan.top/chinaskills/piggymetrics-gateway:$BUILD_ID -f ./gateway/Dockerfile ./gateway/
                docker build -t har.ddupan.top/chinaskills/piggymetrics-config:$BUILD_ID -f ./config/Dockerfile ./config'''
            }
        }
        
        stage("Push") {
            steps {
              	// 将构建的镜像推送到 Harbor 私有镜像
                sh '''docker login har.ddupan.top -u admin -p Harbor12345
                docker push har.ddupan.top/chinaskills/piggymetrics-gateway:$BUILD_ID
                docker push har.ddupan.top/chinaskills/piggymetrics-config:$BUILD_ID'''
            }
        }
        
        stage("Deploy") {
            steps {
              	// 部署新构建的镜像至 Kubernetes 集群中
                sh "sed -i 's/sqshq\\/piggymetrics-gateway/har.ddupan.top\\/chinaskills\\/piggymetrics-gateway:$BUILD_ID/g' yaml/deployment/gateway-deployment.yaml"
                sh "sed -i 's/sqshq\\/piggymetrics-config/har.ddupan.top\\/chinaskills\\/piggymetrics-config:$BUILD_ID/g' yaml/deployment/config-deployment.yaml"
                sh '''kubectl apply -f yaml/deployment/gateway-deployment.yaml
                kubectl apply -f yaml/deployment/config-deployment.yaml
                kubectl apply -f yaml/svc/gateway-svc.yaml
                kubectl apply -f yaml/svc/config-svc.yaml'''
            }
        }
    }
}
```



### 配置 GitLab 与 Jenkins 联动

联动使用 Webhook 方式实现，当 GitLab 上的项目发生设置的行为时，会向配置的 WebHook 地址发送 HTTP request，触发 Jenkins 的预设行为。

由于 GitLab 默认不允许本地网络中的 WebHook 行为，因此要先修改设置

在最顶栏点击扳手图标，打开 GitLab 管理中心

在 设置 -> 网络 -> 外发请求中，勾选 `允许 Webhook 和服务对本地网络的请求`，并 Save Change

在项目中，选择设置 -> WebHooks，将 Jenkins 的 Webhooks 地址填入，取消勾选 `Enable SSL Verification`，并保存

测试一下 Push event，如果返回 HTTP 200 则成功，在 Jenkins 刷新也应能看到新的构建工作已经开始。

### 最后的一些工作

现在构建已经开始，不过应该会稍微需要一点时间，趁此时解决一些其他工作

首先在 Pipeline 脚本中我们选择让镜像 Push 到 Harbor 的 `chinaskills` 项目，因此要到 Harbor 中建立此项目

此项目使用的 Kubernetes Namespace 名为 `springcloud`，但显然我们的 Kubernetes 集群中还没有此命名空间，因此也需要提前建立

```sh
kubectl create namespace springcloud
```

等待构建完成，部署完成后，访问对应端口即可

```sh
kubectl get svc -n springcloud
```

## 最后的话

实际上，如果是离线环境的话，连 Dockerfile 都有需要修改的可能，因为某数字所连 java 的 Docker 镜像都没提供，所以不排除需要自己做中间镜像的可能性，不过资源包中有提供二进制的 JRE tar 包，真需要的话也不是不能搞

数字所应该不会搞更奇怪的东西了，至少我希望是这样的。