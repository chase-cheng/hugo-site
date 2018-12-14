---
title: "Kubernetes 集群搭建"
date: 2018-12-10T10:37:23+08:00
weight: 04
---

kubeadm v1.12.2    
kubernetes v1.12

## 初始化集群
可根据需求自定义 kubeadm yaml 文件
官网文档： https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1alpha3

```
# kubeadm init --config kubeadm.yaml
kubeadm init  --kubernetes-version=v1.12.2 --pod-network-cidr=10.244.0.0/16 
```
**注意：pod-network-cidr 设置的网段与后续要创建的 flannel 的 Network 一致**

如果报错提示	[WARNING Hostname]: hostname "xxxxxx" could not be reached ,可在 /etc/hosts 配置主机名和ip    

执行完之后，会提示执行如下命令：
To start using your cluster, you need to run the following as a regular user:

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
记录下打印出来的 kubeadm join 命令,后续加节点需要用到
You can now join any number of machines by running the following on each node
as root:

```
kubeadm join 172.16.32.115:6443 --token a6ea9z.6o60cjpmndt32f06 --discovery-token-ca-cert-hash sha256:f2026367a4f77f315dd2d67572a9cea2e2c51cb5cecd9297a868116f3eed3715
```

## 检查 master 组件是否起动完成
查看 kube-controller, scheduler, etcd 是否 healthy;
```
kubectl get cs
```

## 检查 kube-proxy 组件是否起动完成
查看 kube-proxy 是否 running;
```
kubectl -n kube-system get po
```

## 去掉污点
如果需要调度pod到master节点，则去掉污点
其中 master 为 hostname
```
kubectl taint nodes master  node-role.kubernetes.io/master-
```

## 初始化网络插件 flannel 配置
[kube-flannel.yaml](https://raw.github.com/chase-cheng/resource/master/yamls/kube-flannel.yaml)  
```
kubectl apply -f https://raw.github.com/chase-cheng/resource/master/yamls/kube-flannel.yaml
```

## 添加slave节点
```
kubeadm join 172.16.32.115:6443 --token a6ea9z.6o60cjpmndt32f06 --discovery-token-ca-cert-hash sha256:f2026367a4f77f315dd2d67572a9cea2e2c51cb5cecd9297a868116f3eed3715
```

## 检查各节点是否正常
```
kubectl get no
```

## 集群重置
如搭建报错，执行如下命令删除集群重新搭建
```
kubeadm reset
# 如已成功安装flannel 插件，还需清除 cni 插件
ifconfig cni0 down
ip link delete cni0
ifconfig flannel.1 down
ip link delete flannel.1
rm -rf /var/lib/cni/
```

