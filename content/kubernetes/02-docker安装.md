---
title: "docker安装"
date: 2018-12-10T10:37:23+08:00
weight: 2
---

 版本：docker17.03   
 官网安装指南: https://docs.docker.com/engine/installation/linux/centos/

## 安装
```
#如果存在，移除旧的docker
yum remove docker docker-common container-selinux docker-selinux docker-engine
rpm -e $(rpm -q docker-ce)

#安装工具
yum install -y yum-utils

#添加仓库
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum makecache
yum install -y policycoreutils-python

#下载并安装docker-ce-selinux
wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-selinux-17.03.2.ce-1.el7.centos.noarch.rpm
rpm -ivh docker-ce-selinux-17.03.2.ce-1.el7.centos.noarch.rpm

#查看docker-ce版本并且安装
yum list docker-ce --showduplicates | sort -r 
yum -y install docker-ce-17.03.2.ce
```

## 配置
### 修改service文件
主要修改 ExecStart 参数，添加一个 Environment 参数，用于设置docker的代理，拉取镜像

```
vi /usr/lib/systemd/system/docker.service
```
### docker.service 模版

```
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network.target firewalld.service

[Service]
Environment="HTTP_PROXY=http://127.0.0.1:8118" "HTTPS_PROXY=http://127.0.0.1:8118"
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd  -H unix:///var/run/docker.sock -H tcp://0.0.0.0:6071 --insecure-registry=0.0.0.0/0
ExecReload=/bin/kill -s HUP $MAINPID
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process

[Install]
WantedBy=multi-user.target

```

### 通过 daemon.json 修改文件系统和数据目录

```
mkdir -p /etc/docker
cat > /etc/docker/daemon.json <<EOF
{
	"storage-driver": "overlay2",
	"storage-opts": [
        "overlay2.override_kernel_check=true"
	],
	"graph": "/data/docker"
}
EOF
```

## 启动

```
#启动
systemctl daemon-reload
systemctl enable docker
systemctl restart docker

#查看版本，详细信息
systemctl status docker
docker info
docker version
```



