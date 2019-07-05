# Helm

[官网](https://helm.sh)

## 安装

首先确保安装helm节点的机器上能够通过kubectl命令调度服务

```
# 下载helm-client二进制文件，我们这里用的是v2.14.0版本
wget https://get.helm.sh/helm-v2.14.0-linux-amd64.tar.gz

# 解压
tar zxvf helm-v2.14.0-linux-amd64.tar.gz

# 将二进制文件放到/bin下
mv linux-amd64/helm /bin/helm

# 安装helm-server
helm init --tiller-image hekai/gcr.io_kubernetes-helm_tiller_v2.14.0 --stable-repo-url http://mirror.azure.cn/kubernetes/charts/

# 验证服务是否正常，输出如下则正常
helm version
Client: &version.Version{SemVer:"v2.14.0", GitCommit:"05811b84a3f93603dd6c2fcfe57944dfa7ab7fd0", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.14.0", GitCommit:"05811b84a3f93603dd6c2fcfe57944dfa7ab7fd0", GitTreeState:"clean"}
```

helm相关操作命令参考官方文档



## 错误解决

```
helm ls 
# 报错Error: configmaps is forbidden: User "system:serviceaccount:kube-system:default" cannot list resource "configmaps" in API group "" in the namespace "kube-system"
// tiller没有正确的角色权限

# 解决
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```



