# kubeasz安装多主k8s集群

参考官方文档:https://github.com/easzlab/kubeasz/blob/master/docs/setup/00-planning_and_overall_intro.md

kubeasz安装版本地址：
https://github.com/easzlab/kubeasz/releases/

增加节点    
node:  https://github.com/easzlab/kubeasz/blob/master/docs/op/op-node.md   
master: https://github.com/easzlab/kubeasz/blob/master/docs/op/op-master.md

## 环境准备

四台干净的虚拟机

| IP             | 主机名  | 角色                            |
| -------------- | ------- | ------------------------------- |
| 192.168.122.11 | master1 | etcd, master1,ansible的管理节点 |
| 192.168.122.12 | master2 | etcd, master2                   |
| 192.168.122.13 | node1   | etcd, node1                     |
| 192.168.122.14 | node2   | node2                           |

1, 主机名配置与绑定

~~~powershell
192.168.122.11 master1
192.168.122.12 master2
192.168.122.13 node1
192.168.122.14 node2
~~~

2, 关闭防火墙，selinux

3, 时间同步

4, yum源保持默认

## 软件包安装

在ansible的管理节点(master1)上安装

~~~powershell
[root@master1 ~]# yum install git python3-pip -y
~~~

~~~powershell
[root@master1 ~]# pip3 install --upgrade pip
[root@master1 ~]# pip install ipython

[root@master1 ~]# pip3 install ansible==2.6.18 netaddr==0.7.19 -i https://mirrors.aliyun.com/pypi/simple/
~~~

## ssh免密

~~~powershell
[root@master1 ~]# ssh-keygen
~~~

~~~powershell
[root@master1 ~]# ssh-copy-id master1
[root@master1 ~]# ssh-copy-id master2
[root@master1 ~]# ssh-copy-id node1
[root@master1 ~]# ssh-copy-id node2
~~~


> 版本：不同版本安装方式不一样，当前版本为==v1.18==.


## 下载安装工具easzup

~~~powershell
[root@master1 ~]# curl -C- -fLO --retry 3 https://github.com/easzlab/kubeasz/releases/download/2.2.1/easzup

[root@master1 ~]# chmod +x easzup
[root@master1 ~]# ll -h easzup
-rwxr-xr-x 1 root root 13K Aug  6 23:36 easzup
~~~

## 安装并配置

使用下面命令下载相关脚本与docker镜像（网速正常情况下，5-10分钟左右）

~~~powershell
[root@master1 ~]# ./easzup -D
~~~

拷贝ansible清单文件

~~~powershell
[root@master1 ~]# cd /etc/ansible && cp example/hosts.multi-node hosts
~~~

配置结果如下

~~~powershell
[root@master1 ansible]# egrep -v '^#|^$' hosts
[etcd]
192.168.122.11 NODE_NAME=etcd1						
192.168.122.12 NODE_NAME=etcd2
192.168.122.13 NODE_NAME=etcd3			# ectd集群三台上(不要用主机名，用IP，否则会ectd服务启动不了，如下图所示)
[kube-master]
192.168.122.11
192.168.122.12							# k8s主节点的IP，不要用主机名，否则kube-api启动不了
[kube-node]
192.168.122.13
192.168.122.14								
[harbor]
[ex-lb]
[chrony]
[all:vars]
CONTAINER_RUNTIME="docker"
CLUSTER_NETWORK="calico"			# 网络方案建议使用calico
PROXY_MODE="ipvs"
SERVICE_CIDR="10.2.0.0/16"
CLUSTER_CIDR="10.3.0.0/16"			# 服务网络与pod网络可不改，我这里为了方便就改了一下（不要和集群节点网络一致)
NODE_PORT_RANGE="20000-40000"
CLUSTER_DNS_DOMAIN="cluster.local."
bin_dir="/opt/kube/bin"
ca_dir="/etc/kubernetes/ssl"
base_dir="/etc/ansible"
~~~

![image-20220322161902088](picture/image-20220322161902088.png)

安装集群时： efk 、 prometheus
```
存在不确定因数，路径不确定（版本不同位置不同），
路径：/etc/ansible/roles/cluster-addon/defaults

# vim main.yml
```

确认ansible连通性

~~~powershell
[root@master1 ansible]# ansible all -m ping
~~~

## 开始集群安装

按下面步骤分步安装

~~~powershell
[root@master1 ansible]# ansible-playbook 01.prepare.yml
[root@master1 ansible]# ansible-playbook 02.etcd.yml
[root@master1 ansible]# ansible-playbook 03.docker.yml
[root@master1 ansible]# ansible-playbook 04.kube-master.yml
[root@master1 ansible]# ansible-playbook 05.kube-node.yml
[root@master1 ansible]# ansible-playbook 06.network.yml
[root@master1 ansible]# ansible-playbook 07.cluster-addon.yml
~~~

## 安装后验证

~~~powershell
[root@master1 ansible]# kubectl version
Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.3", GitCommit:"2e7996e3e2712684bc73f0dec0200d64eec7fe40", GitTreeState:"clean", BuildDate:"2020-05-20T12:52:00Z", GoVersion:"go1.13.9", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.3", GitCommit:"2e7996e3e2712684bc73f0dec0200d64eec7fe40", GitTreeState:"clean", BuildDate:"2020-05-20T12:43:34Z", GoVersion:"go1.13.9", Compiler:"gc", Platform:"linux/amd64"}

~~~



~~~powershell
[root@master1 ansible]# kubectl get componentstatus
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health":"true"}
etcd-0               Healthy   {"health":"true"}
etcd-1               Healthy   {"health":"true"}

~~~



~~~powershell
[root@master1 ansible]# kubectl get nodes
NAME             STATUS                     ROLES    AGE     VERSION
192.168.122.11   Ready,SchedulingDisabled   master   11m     v1.18.3
192.168.122.12   Ready,SchedulingDisabled   master   11m     v1.18.3
192.168.122.13   Ready                      node     8m59s   v1.18.3
192.168.122.14   Ready                      node     8m59s   v1.18.3
~~~

~~~powershell
[root@master1 ansible]# kubectl get pod --all-namespaces
NAMESPACE     NAME                                         READY   STATUS    RESTARTS   AGE
kube-system   coredns-65dbdb44db-zmsrj                     1/1     Running   0          7m26s
kube-system   dashboard-metrics-scraper-545bbb8767-rvqjl   1/1     Running   0          6m13s
kube-system   calico-kube-controllers-7fdc86d8ff-vb25c     1/1     Running   0          9m12s
kube-system   calico-node-8mzvl                            1/1     Running   0          9m12s
kube-system   calico-node-ms27x                            1/1     Running   0          9m12s
kube-system   calico-node-v4cbs                            1/1     Running   0          9m12s
kube-system   calico-node-xbq8p                            1/1     Running   0          9m12s
kube-system   kubernetes-dashboard-65665f84db-7gnvn        1/1     Running   0          6m13s
kube-system   metrics-server-869ffc99cd-qws8w              1/1     Running   0          7m12s
~~~



~~~powershell
[root@master1 ansible]# kubectl get svc --all-namespaces
NAMESPACE     NAME                        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE
default       kubernetes                  ClusterIP   10.68.0.1      <none>        443/TCP                  14m
kube-system   dashboard-metrics-scraper   ClusterIP   10.68.25.239   <none>        8000/TCP                 6m42s
kube-system   kube-dns                    ClusterIP   10.68.0.2      <none>        53/UDP,53/TCP,9153/TCP   7m50s
kube-system   kubernetes-dashboard        NodePort    10.68.242.33   <none>        443:24949/TCP            6m45s
kube-system   metrics-server              ClusterIP   10.68.143.40   <none>        443/TCP                  7m36s
~~~



## dashboard登录

查看nodeport端口(我这里为24949)

~~~powershell
[root@master1 ansible]# kubectl get svc -n kube-system|grep kubernetes-dashboard
kubernetes-dashboard        NodePort    10.68.242.33   <none>        443:24949/TCP            38m
~~~

通过浏览器访问(IP为master的IP,端口为上面查询的端口,https协议)

![image-20220322161927217](picture/image-20220322161927217.png)

![image-20220322161949054](picture/image-20220322161949054.png)

![image-20220322162012388](picture/image-20220322162012388.png)
得到访问的token

~~~powershell
[root@master1 ~]# kubectl -n kube-system describe secret  admin-user
Name:         admin-user-token-2dfjt
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: 245ac66a-eac5-4f2d-9f22-aeec8ddb84ac

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1350 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6Ik1OX2NRbHNQZnZJUTBHYW9faThHck1qdkM3M0VITHBuV0NCZ3ZtZXZ1NU0ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTJkZmp0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIyNDVhYzY2YS1lYWM1LTRmMmQtOWYyMi1hZWVjOGRkYjg0YWMiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.LNKODuXzfcgvYGT5lRvs43kLE28fboTqtLv-0SBMR0GAJbw2Fr1BL85kSe2LsCgwuYJO73dMJSunv7lgrme2iUSIqnly81NJS5STO_TLI-JSdgqEeeb4peQiipmUS57cpk2x8tlD3SOGlGp0ccf13wDT4GqKTpg3GoO_NzfajTTD6vUW-pcPJdGRZli8OgXh5Zg2ubG5OpAXbHWXs0RB1chaIroNCcLj6tofTgD7G-PX44HL9zENDQp5Z4l-ZQWBY-qxpZX9DDR162mqUnes_DtOIPDzfmfnp5BSwFGymLrYKH7VIr0d2C2bjqKwTLurFONW6KnXejvL7-vPIbexPg
~~~



![image-20220322162028033](picture/image-20220322162028033.png)



登录成功

![image-20220322162041182](picture/image-20220322162041182.png)


## kubeasz后期添加master、node节点

- https://github.com/easzlab/kubeasz/blob/master/docs/op/op-node.md
- https://github.com/easzlab/kubeasz/blob/master/docs/op/op-master.md

```powershell 
新增kube-master节点大致流程为：tools/03.addmaster.yml
[可选]新节点安装 chrony 时间同步
新节点预处理 prepare
新节点安装 docker 服务
新节点安装 kube-master 服务
新节点安装 kube-node 服务
新节点安装网络插件相关
禁止业务 pod调度到新master节点
更新 node 节点 haproxy 负载均衡并重启
```

### 添加master节点

```powershell
需要在有ansible的master节点执行：
# ssh-copy-id 主机名
# vim /etc/ansible/hosts             # 在 “[kube-master]” 字段下面添加主机配置
# easzctl add-master 192.168.2.15    # 添加master；del-master，删除master

查看集群
[root@master1 ansible]# kubectl get nodes | grep 192.168.2.15 
NAME           STATUS                     ROLES    AGE   VERSION
192.168.2.15   Ready,SchedulingDisabled   master   76m   v1.20.2
节点已加入
```
### 删除master节点

删除kube_master节点大致流程为：(参考ezctl 中del-master函数和playbooks/33.delmaster.yml)

#### 思路：
检测是否可以删除    
迁移节点 pod    
删除 master 相关服务及文件  
删除 node 相关服务及文件    
从集群删除 node 节点    
从 ansible hosts 移除节点   
在 ansible 控制端更新 kubeconfig    
更新 node 节点 haproxy 配置 

#### 操作步骤
```powershell
# ezctl del-master  192.168.1.15  # 假设待删除节点 192.168.1.11
```
#### 验证
略

### 添加node节点

```powershell
需要在有ansible的master节点执行：
# ssh-copy-id 主机名
# vim /etc/ansible/hosts             # 在 “[kube-node]” 字段下面添加主机配置
# easzctl add-node 192.168.3.16     # del-node，删除node

查看集群
[root@master1 ansible]# kubectl get nodes | grep 192.168.2.16 
NAME           STATUS                     ROLES    AGE   VERSION
192.168.2.16   Ready                      node     76m   v1.20.2
节点已加入
```
### 删除node节点
删除 node 节点流程：(参考ezctl 里面del-node函数 和 playbooks/32.delnode.yml)

#### 思路：  
检测是否可以删除    
迁移节点上的 pod    
删除 node 相关服务及文件    
从集群删除 node 

#### 操作步骤：
```powershell
# easzctl del-node 192.168.1.16   #假设待删除节点为 192.168.1.11
```
#### 验证    
略

## 升级集群

- https://github.com/easzlab/kubeasz/blob/master/docs/op/upgrade.md

```powershell
升级集群
https://github.com/easzlab/kubeasz/blob/master/docs/op/upgrade.md
升级集群 #未尝试
1)备份etcd
ETCDCTL_API=3 etcdctl snapshot save backup.db
查看备份文件信息
ETCDCTL_API=3 etcdctl --write-out=table snapshot status backup.db
2)到本项目的根目录kubeasz
 cd  /dir/to/kubeasz
拉取最新的代码
git pull origin master
3)下载升级目标版本的kubernetes二进制包，并替换/etc/ansible/bin/下的二进制文件
4)docker升级(略)，除非特别需要，否则不建议频繁升级docker 
5)如果接受业务中断，执行:
ansible-playbook -t upgrade_k8s,restart_dockerd 22.upgrade.yml
6)不能接受短暂中断，需要这样做:
  a)ansible-playbook -t upgrade_k8s 22.upgrade.yml 
  b)到所有node上逐一:
kubectl cordon和kubectl drain //迁移业务pod 
systemctl restart docker
kubectl uncordon //恢复pod
```

## 备份和恢复

* https://github.com/easzlab/kubeasz/blob/master/docs/op/cluster_restore.md



## 扩展
参考地址：https://www.toutiao.com/c/user/token/MS4wLjABAAAA0YFomuMNm87NNysXeUsQdI0Tt3gOgz8WG_0B3MzxsmI/?tab=article   
==v1.20 版本以上部署方式==

#### 1.环境初始化（主机名、IP、hosts文件、防火墙、selinux、时间同步、python）

#### 2.脚本安装

> 将以下脚本部署到master1。 

```
# vim install 

#!/bin/bash
# auther: long   Notes: Original author "Boge", later modified by "dragon".
# descriptions:  the shell scripts will use ansible to deploy K8S at binary for siample

# 传参检测
[ $# -ne 6 ] && echo -e "Usage: $0 rootpasswd netnum nethosts cri cni k8s-cluster-name\nExample: bash $0 bogedevops 10.0.1 201\ 202\ 203\ 204 [containerd|docker] [calico|flannel] test\n" && exit 11

# 变量定义
export release=3.1.0    # 2.2.4, 2.2.3, 3.0.0, 3.2.0
export k8s_ver=v1.21.0  # v1.20.2, v1.19.7, v1.18.15, v1.17.17
rootpasswd=$1
netnum=$2
nethosts=$3
cri=$4
cni=$5
clustername=$6
if ls -1v ./kubeasz*.tar.gz &>/dev/null;then software_packet="$(ls -1v ./kubeasz*.tar.gz )";else software_packet="";fi
pwd="/etc/kubeasz"

# deploy机器安装相应软件包
if cat /etc/redhat-release &>/dev/null;then
    yum install git python3-pip sshpass -y && pip3 install --upgrade pip && pip install ipython && pip3 install ansible==2.6.18 netaddr==0.7.19 -i https://mirrors.aliyun.com/pypi/simple/
else
    apt install git python3-pip sshpass -y && pip3 install --upgrade pip && pip install ipython && pip3 install ansible==2.6.18 netaddr==0.7.19 -i https://mirrors.aliyun.com/pypi/simple/
fi

# 在deploy机器做其他node的ssh免密操作
for host in `echo "${nethosts}"`
do
    echo "============ ${netnum}.${host} ===========";

    if [[ ${USER} == 'root' ]];then
        [ ! -f /${USER}/.ssh/id_rsa ] &&\
        ssh-keygen -t rsa -P '' -f /${USER}/.ssh/id_rsa
    else
        [ ! -f /home/${USER}/.ssh/id_rsa ] &&\
        ssh-keygen -t rsa -P '' -f /home/${USER}/.ssh/id_rsa
    fi
    sshpass -p ${rootpasswd} ssh-copy-id -o StrictHostKeyChecking=no ${USER}@${netnum}.${host}

    if cat /etc/redhat-release &>/dev/null;then
        ssh -o StrictHostKeyChecking=no ${USER}@${netnum}.${host} "yum update -y"
    else
        ssh -o StrictHostKeyChecking=no ${USER}@${netnum}.${host} "apt-get update && apt-get upgrade -y && apt-get dist-upgrade -y"
        [ $? -ne 0 ] && ssh -o StrictHostKeyChecking=no ${USER}@${netnum}.${host} "apt-get -yf install"
    fi
done


# deploy机器下载k8s二进制安装脚本

if [[ ${software_packet} == '' ]];then
    curl -C- -fLO --retry 3 https://github.com/easzlab/kubeasz/releases/download/${release}/ezdown
    sed -ri "s+^(K8S_BIN_VER=).*$+\1${k8s_ver}+g" ezdown
    chmod +x ./ezdown
    # 使用工具脚本下载
    ./ezdown -D && ./ezdown -P
else
    tar xvf ${software_packet} -C /etc/
    chmod +x ${pwd}/{ezctl,ezdown}
fi

# 初始化一个名为my的k8s集群配置

CLUSTER_NAME="$clustername"
${pwd}/ezctl new ${CLUSTER_NAME}
if [[ $? -ne 0 ]];then
    echo "cluster name [${CLUSTER_NAME}] was exist in ${pwd}/clusters/${CLUSTER_NAME}."
    exit 1
fi

if [[ ${software_packet} != '' ]];then
    # 设置参数，启用离线安装
    sed -i 's/^INSTALL_SOURCE.*$/INSTALL_SOURCE: "offline"/g' ${pwd}/clusters/${CLUSTER_NAME}/config.yml
fi


# to check ansible service
ansible all -m ping

#---------------------------------------------------------------------------------------------------




#修改二进制安装脚本配置 config.yml

sed -ri "s+^(CLUSTER_NAME:).*$+\1 \"${CLUSTER_NAME}\"+g" ${pwd}/clusters/${CLUSTER_NAME}/config.yml

## k8s上日志及容器数据存独立磁盘步骤（参考阿里云的）

[ ! -d /var/lib/container ] && mkdir -p /var/lib/container/{kubelet,docker}

## cat /etc/fstab
# UUID=105fa8ff-bacd-491f-a6d0-f99865afc3d6 /                       ext4    defaults        1 1
# /dev/vdb /var/lib/container/ ext4 defaults 0 0
# /var/lib/container/kubelet /var/lib/kubelet none defaults,bind 0 0
# /var/lib/container/docker /var/lib/docker none defaults,bind 0 0

## tree -L 1 /var/lib/container
# /var/lib/container
# ├── docker
# ├── kubelet
# └── lost+found

# docker data dir
DOCKER_STORAGE_DIR="/var/lib/container/docker"
sed -ri "s+^(DOCKER_STORAGE_DIR:).*$+DOCKER_STORAGE_DIR: \"${DOCKER_STORAGE_DIR}\"+g" ${pwd}/clusters/${CLUSTER_NAME}/config.yml
# containerd data dir
CONTAINERD_STORAGE_DIR="/var/lib/container/containerd"
sed -ri "s+^(CONTAINERD_STORAGE_DIR:).*$+CONTAINERD_STORAGE_DIR: \"${CONTAINERD_STORAGE_DIR}\"+g" ${pwd}/clusters/${CLUSTER_NAME}/config.yml
# kubelet logs dir
KUBELET_ROOT_DIR="/var/lib/container/kubelet"
sed -ri "s+^(KUBELET_ROOT_DIR:).*$+KUBELET_ROOT_DIR: \"${KUBELET_ROOT_DIR}\"+g" ${pwd}/clusters/${CLUSTER_NAME}/config.yml
if [[ $clustername != 'aws' ]]; then
    # docker aliyun repo
    REG_MIRRORS="https://pqbap4ya.mirror.aliyuncs.com"
    sed -ri "s+^REG_MIRRORS:.*$+REG_MIRRORS: \'[\"${REG_MIRRORS}\"]\'+g" ${pwd}/clusters/${CLUSTER_NAME}/config.yml
fi
# [docker]信任的HTTP仓库
sed -ri "s+127.0.0.1/8+${netnum}.0/24+g" ${pwd}/clusters/${CLUSTER_NAME}/config.yml
## disable dashboard auto install
#sed -ri "s+^(dashboard_install:).*$+\1 \"no\"+g" ${pwd}/clusters/${CLUSTER_NAME}/config.yml

# 融合配置准备
CLUSEER_WEBSITE="${CLUSTER_NAME}k8s.gtapp.xyz"
lb_num=$(grep -wn '^MASTER_CERT_HOSTS:' ${pwd}/clusters/${CLUSTER_NAME}/config.yml |awk -F: '{print $1}')
lb_num1=$(expr ${lb_num} + 1)
lb_num2=$(expr ${lb_num} + 2)
sed -ri "${lb_num1}s+.*$+  - "${CLUSEER_WEBSITE}"+g" ${pwd}/clusters/${CLUSTER_NAME}/config.yml
sed -ri "${lb_num2}s+(.*)$+#\1+g" ${pwd}/clusters/${CLUSTER_NAME}/config.yml

# node节点最大pod 数
MAX_PODS="120"
sed -ri "s+^(MAX_PODS:).*$+\1 ${MAX_PODS}+g" ${pwd}/clusters/${CLUSTER_NAME}/config.yml



# 修改二进制安装脚本配置 hosts
# clean old ip
sed -ri '/192.168.1.1/d' ${pwd}/clusters/${CLUSTER_NAME}/hosts
sed -ri '/192.168.1.2/d' ${pwd}/clusters/${CLUSTER_NAME}/hosts
sed -ri '/192.168.1.3/d' ${pwd}/clusters/${CLUSTER_NAME}/hosts
sed -ri '/192.168.1.4/d' ${pwd}/clusters/${CLUSTER_NAME}/hosts

# 输入准备创建ETCD集群的主机位
echo "enter etcd hosts here (example: 203 202 201) ↓"
read -p "" ipnums
for ipnum in `echo ${ipnums}`
do
    echo $netnum.$ipnum
    sed -i "/\[etcd/a $netnum.$ipnum"  ${pwd}/clusters/${CLUSTER_NAME}/hosts
done

# 输入准备创建KUBE-MASTER集群的主机位
echo "enter kube-master hosts here (example: 202 201) ↓"
read -p "" ipnums
for ipnum in `echo ${ipnums}`
do
    echo $netnum.$ipnum
    sed -i "/\[kube_master/a $netnum.$ipnum"  ${pwd}/clusters/${CLUSTER_NAME}/hosts
done

# 输入准备创建KUBE-NODE集群的主机位
echo "enter kube-node hosts here (example: 204 203) ↓"
read -p "" ipnums
for ipnum in `echo ${ipnums}`
do
    echo $netnum.$ipnum
    sed -i "/\[kube_node/a $netnum.$ipnum"  ${pwd}/clusters/${CLUSTER_NAME}/hosts
done

# 配置容器运行时CNI
case ${cni} in
    flannel)
    sed -ri "s+^CLUSTER_NETWORK=.*$+CLUSTER_NETWORK=\"${cni}\"+g" ${pwd}/clusters/${CLUSTER_NAME}/hosts
    ;;
    calico)
    sed -ri "s+^CLUSTER_NETWORK=.*$+CLUSTER_NETWORK=\"${cni}\"+g" ${pwd}/clusters/${CLUSTER_NAME}/hosts
    ;;
    *)
    echo "cni need be flannel or calico."
    exit 11
esac

# 配置K8S的ETCD数据备份的定时任务
if cat /etc/redhat-release &>/dev/null;then
    if ! grep -w '94.backup.yml' /var/spool/cron/root &>/dev/null;then echo "00 00 * * * `which ansible-playbook` ${pwd}/playbooks/94.backup.yml &> /dev/null" >> /var/spool/cron/root;else echo exists ;fi
    chown root.crontab /var/spool/cron/root
    chmod 600 /var/spool/cron/root
else
    if ! grep -w '94.backup.yml' /var/spool/cron/crontabs/root &>/dev/null;then echo "00 00 * * * `which ansible-playbook` ${pwd}/playbooks/94.backup.yml &> /dev/null" >> /var/spool/cron/crontabs/root;else echo exists ;fi
    chown root.crontab /var/spool/cron/crontabs/root
    chmod 600 /var/spool/cron/crontabs/root
fi
rm /var/run/cron.reboot
service crond restart




#---------------------------------------------------------------------------------------------------
# 准备开始安装了
rm -rf ${pwd}/{dockerfiles,docs,.gitignore,pics,dockerfiles} &&\
find ${pwd}/ -name '*.md'|xargs rm -f
read -p "Enter to continue deploy k8s to all nodes >>>" YesNobbb

# now start deploy k8s cluster
cd ${pwd}/

# to prepare CA/certs & kubeconfig & other system settings
${pwd}/ezctl setup ${CLUSTER_NAME} 01
sleep 1
# to setup the etcd cluster
${pwd}/ezctl setup ${CLUSTER_NAME} 02
sleep 1
# to setup the container runtime(docker or containerd)
case ${cri} in
    containerd)
    sed -ri "s+^CONTAINER_RUNTIME=.*$+CONTAINER_RUNTIME=\"${cri}\"+g" ${pwd}/clusters/${CLUSTER_NAME}/hosts
    ${pwd}/ezctl setup ${CLUSTER_NAME} 03
    ;;
    docker)
    sed -ri "s+^CONTAINER_RUNTIME=.*$+CONTAINER_RUNTIME=\"${cri}\"+g" ${pwd}/clusters/${CLUSTER_NAME}/hosts
    ${pwd}/ezctl setup ${CLUSTER_NAME} 03
    ;;
    *)
    echo "cri need be containerd or docker."
    exit 11
esac
sleep 1
# to setup the master nodes
${pwd}/ezctl setup ${CLUSTER_NAME} 04
sleep 1
# to setup the worker nodes
${pwd}/ezctl setup ${CLUSTER_NAME} 05
sleep 1
# to setup the network plugin(flannel、calico...)
${pwd}/ezctl setup ${CLUSTER_NAME} 06
sleep 1
# to setup other useful plugins(metrics-server、coredns...)
${pwd}/ezctl setup ${CLUSTER_NAME} 07
sleep 1
# [可选]对集群所有节点进行操作系统层面的安全加固  https://github.com/dev-sec/ansible-os-hardening
#ansible-playbook roles/os-harden/os-harden.yml
#sleep 1
cd `dirname ${software_packet:-/tmp}`


k8s_bin_path='/opt/kube/bin'


echo "-------------------------  k8s version list  ---------------------------"
${k8s_bin_path}/kubectl version
echo
echo "-------------------------  All Healthy status check  -------------------"
${k8s_bin_path}/kubectl get componentstatus
echo
echo "-------------------------  k8s cluster info list  ----------------------"
${k8s_bin_path}/kubectl cluster-info
echo
echo "-------------------------  k8s all nodes list  -------------------------"
${k8s_bin_path}/kubectl get node -o wide
echo
echo "-------------------------  k8s all-namespaces's pods list   ------------"
${k8s_bin_path}/kubectl get pod --all-namespaces
echo
echo "-------------------------  k8s all-namespaces's service network   ------"
${k8s_bin_path}/kubectl get svc --all-namespaces
echo
echo "-------------------------  k8s welcome for you   -----------------------"
echo

# you can use k alias kubectl to siample
echo "alias k=kubectl && complete -F __start_kubectl k" >> ~/.bashrc

# get dashboard url
${k8s_bin_path}/kubectl cluster-info|grep dashboard|awk '{print $NF}'|tee -a /root/k8s_results

# get login token
${k8s_bin_path}/kubectl -n kube-system describe secret $(${k8s_bin_path}/kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')|grep 'token:'|awk '{print $NF}'|tee -a /root/k8s_results
echo
echo "you can look again dashboard and token info at  >>> /root/k8s_results <<<"
#echo ">>>>>>>>>>>>>>>>> You can excute command [ source ~/.bashrc ] <<<<<<<<<<<<<<<<<<<<"
echo ">>>>>>>>>>>>>>>>> You need to excute command [ reboot ] to restart all nodes <<<<<<<<<<<<<<<<<<<<"
rm -f $0
[ -f ${software_packet} ] && rm -f ${software_packet}
#rm -f ${pwd}/roles/deploy/templates/${USER_NAME}-csr.json.j2
#sed -ri "s+${USER_NAME}+admin+g" ${pwd}/roles/prepare/tasks/main.yml


```
##### 3.开始在线安装
```
chmod +x long.sh 

# 注：脚本执行语法（脚本名称 + 服务器密码 + ip网络位 + ip主机位 + 选择容器运行时 + CNI + 集群名称）  
./long.sh 0811wang 10.1.1 11\ 12\ 13\ 14\ 15 docker calico long

    脚本名称: 随意
    服务器密码：所有服务器密码统一，为后期生成密钥使用
    ip网络位： 参考案例方法
    ip主机位： 参考案例方法 （主机号+\, 以此类推）
    选择容器运行时：[containerd|docker] 
    CNI：[calico|flannel] 
    集群名称：更加业务名称（随意）
    
# 脚本基本是自动化的，除了下面几处提示按要求复制粘贴下，再回车即可

# 输入准备创建ETCD集群的主机位，复制  203 202 201 粘贴并回车
echo "enter etcd hosts here (example: 203 202 201) ↓"

# 输入准备创建KUBE-MASTER集群的主机位，复制  202 201 粘贴并回车
echo "enter kube-master hosts here (example: 202 201) ↓"

# 输入准备创建KUBE-NODE集群的主机位，复制  204 203 粘贴并回车
echo "enter kube-node hosts here (example: 204 203) ↓"

# 这里会提示你是否继续安装，没问题的话直接回车即可
Enter to continue deploy k8s to all nodes >>>

# 安装完成后重新加载下环境变量以实现kubectl命令补齐
. ~/.bashrc 

# 安装完成后重启
reboot
```
##### 4.集群验证
```
# kubectl get nodes


# kubectl get all -n key
```