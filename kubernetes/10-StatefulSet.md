# 一、StatefulSet控制器

参考: https://kubernetes.io/zh/docs/concepts/workloads/controllers/statefulset/

StatefulSet 是用来管理有状态应用的控制器。

## 无状态应用与有状态应用

**无状态应用:**  如nginx

* 请求本身包含了响应端为响应这一请求所需的全部信息。每一个请求都像首次执行一样，不会依赖之前的数据进行响应。
* 不需要持久化的数据
* 无状态应用的多个实例之间互不依赖，可以无序的部署、删除或伸缩



**有状态应用:**  如mysql

* 前后请求有关联与依赖
* 需要持久化的数据
* 有状态应用的多个实例之间有依赖，不能相互替换：无论怎么调度，每个 Pod 都有一个永久不变的 ID。



## StatefulSet的特点

- 稳定的、唯一的网络标识符。		(通过headless服务实现)
- 稳定的、持久的存储。			       (通过PV，PVC，storageclass实现)
- 有序的、优雅的部署和缩放。       
- 有序的、自动的滚动更新。           



## StatefulSet的YAML组成

需要三个组成部分:

1. headless service:                 实现稳定，唯一的网络标识
2. statefulset类型资源:            写法和deployment几乎一致，就是类型不一样
3. volumeClaimTemplate :     指定存储卷





# 二、nginx+statefulset+nfs案例

## 创建StatefulSet应用

参考: https://kubernetes.io/zh/docs/tutorials/stateful-application/basic-stateful-set/

创建statelfulset应用来调用名为managed-nfs-storage的storageclass,以实现动态供给

```powershell
[root@master1 ~]# vim nginx-storageclass-nfs.yml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None								   # 无头服务
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web											# statefulset的名称
spec:
  serviceName: "nginx"								# 服务名与上面的无头服务名要一致
  replicas: 3										# 3个副本
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.15-alpine
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
          
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "managed-nfs-storage"		# 与前面定义的storageclass名称对应
      resources:
        requests:
          storage: 1Gi
```



```powershell
[root@master1 ~]# kubectl apply -f nginx-storageclass-nfs.yml
service/nginx created
statefulset.apps/web created
```

~~~powershell
[root@master1 ~]# kubectl get statefulsets			# 可以简写成sts
NAME   READY   AGE
web    3/3     1m
~~~



## 验证pod,pv,pvc

产生了3个pod

```powershell
[root@master1 ~]# kubectl get pods |grep web
web-0                                     1/1     Running   0          1m15s
web-1                                     1/1     Running   0          1m7s
web-2                                     1/1     Running   0          57s
```

自动产生了3个pv

```powershell
[root@master1 ~] # kubectl get pv
pvc-2436b20d-1be3-4c2e-87a9-5533e5c5e2c6  1Gi  RWO   Delete  Bound  default/www-web-0   managed-nfs-storage       3m
pvc-3114be74-5969-40eb-aeb3-87a3b9ae17bc  1Gi  RWO   Delete  Bound  default/www-web-1   managed-nfs-storage       2m
pvc-43afb71d-1d02-4699-b00c-71679fd75fc3  1Gi  RWO   Delete  ound   default/www-web-2   managed-nfs-storage       2m
```

自动产生了3个PVC

```powershell
[root@master1 ~]# kubectl get pvc |grep web
www-web-0  Bound   pvc-2436b20d-1be3-4c2e-87a9-5533e5c5e2c6   1Gi   RWO  managed-nfs-storage  3m
www-web-1  Bound   pvc-3114be74-5969-40eb-aeb3-87a3b9ae17bc   1Gi   RWO  managed-nfs-storage  2m
www-web-2  Bound   pvc-43afb71d-1d02-4699-b00c-71679fd75fc3   1Gi   RWO  managed-nfs-storage  2m
```

## 验证nfs服务目录

在nfs服务器（这里为hostos)的共享目录中发现自动产生了3个子目录

```powershell
[root@hostos ~]# ls /data/nfs/
default-www-web-0-pvc-2436b20d-1be3-4c2e-87a9-5533e5c5e2c6  
default-www-web-2-pvc-43afb71d-1d02-4699-b00c-71679fd75fc3
default-www-web-1-pvc-3114be74-5969-40eb-aeb3-87a3b9ae17bc  
```

3个子目录默认都为空目录

```powershell
[root@hostos ~]# tree /data/nfs/
/data/nfs/
├── default-www-web-0-pvc-2436b20d-1be3-4c2e-87a9-5533e5c5e2c6
├── default-www-web-1-pvc-3114be74-5969-40eb-aeb3-87a3b9ae17bc
└── default-www-web-2-pvc-43afb71d-1d02-4699-b00c-71679fd75fc3
```

## 验证存储持久性

在3个pod中其中一个创建一个主页文件

```powershell
[root@master1 ~]# kubectl exec -it web-0 -- /bin/sh
/ # echo "haha" >  /usr/share/nginx/html/index.html
/ # exit
```

在nfs服务器上发现文件被创建到了对应的目录中

```powershell
[root@hostos ~]# tree /data/nfs/
/data/nfs/
├── default-www-web-0-pvc-2436b20d-1be3-4c2e-87a9-5533e5c5e2c6
│   └── index.html								# 此目录里多了index.html文件，对应刚才在web-0的pod中的创建
├── default-www-web-1-pvc-3114be74-5969-40eb-aeb3-87a3b9ae17bc
└── default-www-web-2-pvc-43afb71d-1d02-4699-b00c-71679fd75fc3


[root@hostos ~]# cat /data/nfs/default-www-web-0-pvc-2436b20d-1be3-4c2e-87a9-5533e5c5e2c6/index.html
haha											# 文件内的内容也与web-0的pod中创建的一致

```

删除web-0这个pod,再验证

```powershell
[root@master1 ~]# kubectl delete pod web-0
pod "web-0" deleted


[root@master1 ~]# kubectl get pods |grep web			# 因为控制器的原因，会迅速再拉起web-0这个pod
web-0                                     1/1     Running   0          9s	  # 时间上看到是新拉起的pod
web-1                                     1/1     Running   0          37m
web-2                                     1/1     Running   0          37m

[root@master1 ~]# kubectl exec -it web-0 -- cat /usr/share/nginx/html/index.html
haha													# 新拉起的pod仍然是相同的存储数据

[root@hostos ~]# cat /data/nfs/default-www-web-0-pvc-2436b20d-1be3-4c2e-87a9-5533e5c5e2c6/index.html
haha													# nfs服务器上的数据还在
```

**结论: 说明数据可持久化**



## 验证pod唯一名称

回顾域名格式:

service: <service name>.<namespace name>.svc.cluster.local.

pod: <PodName>.<service name>.<namespace name>.svc.cluster.local.



可以看到在`web-0`这个pod中，nslookup查询service的域名，直接解析成了3个pod的域名

~~~powershell
[root@master1 ~]# kubectl exec -it web-0 -- /bin/sh
/ # nslookup  nginx-svc.default.svc.cluster.local.	

Name:      nginx-svc.default.svc.cluster.local.					
Address 1: 10.3.104.48 web-0.nginx-svc.default.svc.cluster.local.
Address 2: 10.3.104.47 web-2.nginx-svc.default.svc.cluster.local
Address 3: 10.3.166.185 web-1.nginx-svc.default.svc.cluster.local
~~~



ping这三个pod的域名都可以ping通

~~~powershell
/ # ping web-0.nginx-svc.default.svc.cluster.local.
PING web-0.nginx-svc.default.svc.cluster.local. (10.3.104.48): 56 data bytes
64 bytes from 10.3.104.48: seq=0 ttl=64 time=0.095 ms
64 bytes from 10.3.104.48: seq=1 ttl=64 time=0.159 ms
......
~~~

~~~powershell
/ # ping web-1.nginx-svc.default.svc.cluster.local.
PING web-1.nginx-svc.default.svc.cluster.local. (10.3.166.185): 56 data bytes
64 bytes from 10.3.166.185: seq=0 ttl=62 time=0.700 ms
64 bytes from 10.3.166.185: seq=1 ttl=62 time=0.493 ms
......
~~~

~~~powershell
/ # ping web-2.nginx-svc.default.svc.cluster.local.
PING web-2.nginx-svc.default.svc.cluster.local. (10.3.104.47): 56 data bytes
64 bytes from 10.3.104.47: seq=0 ttl=63 time=0.101 ms
64 bytes from 10.3.104.47: seq=1 ttl=63 time=0.086 ms
......
~~~

补充: 当pod被删除后，重新拉起来，pod-IP可能会变，但上面的pod域名仍然可以ping通(请自行验证)



## 验证statefulset的伸缩

### 扩容

~~~powershell
[root@master1 ~]# kubectl scale sts web --replicas=4
statefulset.apps/web scaled
~~~



~~~powershell
[root@master1 ~]# kubectl get pods
NAME                                      READY   STATUS    RESTARTS   AGE
nfs-client-provisioner-5b5ddcd6c8-c2gbl   1/1     Running   0          3h3m
web-0                                     1/1     Running   0          7m31s
web-1                                     1/1     Running   0          21m
web-2                                     1/1     Running   0          21m
web-3                                     1/1     Running   0          10s
有序地扩展了一个pod,名称为web-3
~~~



### 裁剪

~~~powershell
[root@master1 ~]# kubectl scale sts web --replicas=1
statefulset.apps/web scaled

~~~



~~~powershell
[root@master1 ~]# kubectl get pods |grep web
web-0                                     1/1     Running       0          31m
web-1                                     1/1     Running       0          45m
web-2                                     1/1     Running       0          22m
web-3                                     0/1     Terminating   0          22m
先裁剪web-3
~~~



~~~powershell
[root@master1 ~]# kubectl get pods |grep web
web-0                                     1/1     Running       0          31m
web-1                                     1/1     Running       0          45m
web-2                                     1/1     Terminating   0          22m
再裁剪web-2
~~~



~~~powershell
[root@master1 ~]# kubectl get pods  |grep web
web-0                                     1/1     Running       0          32m
web-1                                     0/1     Terminating   0          46m
最后裁剪web-1
~~~



~~~powershell
[root@master1 ~]# kubectl get pods |grep web
web-0                                     1/1     Running   0          32m
只留下web-0
~~~



# 三、mysql-statefulset-nfs案例

## 编写statefulset

~~~powershell
[root@master1 ~]# vim statefulset-mysql-nfs.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-svc
spec:
  clusterIP: None					# 无头服务
  ports:
  - port: 3306
    protocol: TCP
    targetPort: 3306
  selector:
    app: mysql
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-svc			# 与上面服务名一致
  replicas: 1						# 副本数为1就OK，这里不做mysql集群
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: c1
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD		# mysql5.7的镜像必须要设置一下mysql的root密码
          value: "123456"
        - name: MYSQL_DATABASE			# 给它建一个库名为daniel，用于验证
          value: daniel
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "managed-nfs-storage"          
      resources:
        requests:
          storage: 3Gi
~~~

## 应用YAML

~~~powershell
[root@master1 ~]# kubectl apply -f statefulset-mysql-nfs.yaml
service/mysql-svc created
statefulset.apps/mysql created
~~~

## 验证资源

~~~powershell
[root@master1 ~]# kubectl get pods  |grep mysql
mysql-0                                   1/1     Running   0          28s
~~~

~~~powershell
[root@master1 ~]# kubectl get sts |grep mysql
mysql   1/1     1m
~~~

~~~powershell
[root@master1 ~]# kubectl get pv |grep mysql
pvc-bc72e7ac-d65b-42ae-853f-14b5b1c9d30c   3Gi   RWO   Delete   Bound    default/mysql-data-mysql-0   managed-nfs-storage          6m

~~~

~~~powershell
[root@master1 ~]# kubectl get pvc |grep mysql
mysql-data-mysql-0   Bound    pvc-bc72e7ac-d65b-42ae-853f-14b5b1c9d30c   3Gi     RWO    managed-nfs-storage   7m

~~~

## 验证mysql

~~~powershell
[root@master1 ~]# kubectl exec -it mysql-0 -- /bin/bash
root@mysql-0:/# mysql -p123456
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.32 MySQL Community Server (GPL)

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| daniel             |					# 这里daneil库就是帮我们创建的
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.01 sec)

mysql> exit

~~~

验证nfs上的数据

~~~powershell
[root@hostos ~]# ls /data/nfs/default-mysql-data-mysql-0-pvc-bc72e7ac-d65b-42ae-853f-14b5b1c9d30c/
auto.cnf    client-cert.pem  ib_buffer_pool  ib_logfile1         private_key.pem  server-key.pem
ca-key.pem  client-key.pem   ibdata1         mysql               public_key.pem   sys
ca.pem      daniel           ib_logfile0     performance_schema  server-cert.pem
~~~



删除mysql-0这个pod，会帮我们再次启动，并且数据还是用原来的数据（请自行验证)



