### Prometheus 

随着heapster项目停止更新并慢慢被metrics-server取代，集群监控这项任务也将最终转移。prometheus的监控理念、数据结构设计其实相当精简，包括其非常灵活的查询语言；但是对于初学者来说，想要在k8s集群中实践搭建一套相对可用的部署却比较麻烦，由此还产生了不少专门的项目（如：prometheus-operator），本文介绍使用helm chart部署集群的prometheus监控。

helm已成为CNCF独立托管项目，预计会更加流行起来

#### kubeasz 集成安装

* 1.修改 clusters/xxxx/config.yml 中配置项 prom_install: "yes"
* 2.安装 ezctl setup xxxx 07

> 注：涉及到镜像需从 quay.io 下载，国内比较慢，可以使用项目中的工具脚本 tools/imgutils

#### 验证安装
```
[root@master1 ~]# kubectl get all -n monitor
NAME                                                         READY   STATUS    RESTARTS   AGE
pod/alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2     Running   0          6h44m
pod/prometheus-grafana-55c5f574d9-9m8xd                      2/2     Running   0          6h45m
pod/prometheus-kube-prometheus-operator-5f6774b747-4444g     1/1     Running   0          6h45m
pod/prometheus-kube-state-metrics-5f89586745-52rz4           1/1     Running   0          6h45m
pod/prometheus-prometheus-kube-prometheus-prometheus-0       2/2     Running   1          6h44m
pod/prometheus-prometheus-node-exporter-2pvml                1/1     Running   0          6h45m
pod/prometheus-prometheus-node-exporter-4sb4q                1/1     Running   0          6h45m
pod/prometheus-prometheus-node-exporter-bgmxc                1/1     Running   0          6h45m
pod/prometheus-prometheus-node-exporter-lrmqw                1/1     Running   0          6h45m
pod/prometheus-prometheus-node-exporter-mzk29                1/1     Running   0          6h45m

NAME                                              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/alertmanager-operated                     ClusterIP   None            <none>        9093/TCP,9094/TCP,9094/UDP   6h44m
service/prometheus-grafana                        NodePort    10.68.85.57     <none>        80:30903/TCP                 6h45m
service/prometheus-kube-prometheus-alertmanager   NodePort    10.68.189.90    <none>        9093:30902/TCP               6h45m
service/prometheus-kube-prometheus-operator       NodePort    10.68.159.23    <none>        443:30900/TCP                6h45m
service/prometheus-kube-prometheus-prometheus     NodePort    10.68.103.65    <none>        9090:30901/TCP               6h45m
service/prometheus-kube-state-metrics             ClusterIP   10.68.236.202   <none>        8080/TCP                     6h45m
service/prometheus-operated                       ClusterIP   None            <none>        9090/TCP                     6h44m
service/prometheus-prometheus-node-exporter       ClusterIP   10.68.23.84     <none>        9100/TCP                     6h45m

NAME                                                 DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/prometheus-prometheus-node-exporter   5         5         5       5            5           <none>          6h45m

NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus-grafana                    1/1     1            1           6h45m
deployment.apps/prometheus-kube-prometheus-operator   1/1     1            1           6h45m
deployment.apps/prometheus-kube-state-metrics         1/1     1            1           6h45m

NAME                                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/prometheus-grafana-55c5f574d9                    1         1         1       6h45m
replicaset.apps/prometheus-kube-prometheus-operator-5f6774b747   1         1         1       6h45m
replicaset.apps/prometheus-kube-state-metrics-5f89586745         1         1         1       6h45m

NAME                                                                    READY   AGE
statefulset.apps/alertmanager-prometheus-kube-prometheus-alertmanager   1/1     6h44m
statefulset.apps/prometheus-prometheus-kube-prometheus-prometheus       1/1     6h44m
```
* 访问prometheus的web界面：http://$NodeIP:30901
* 访问alertmanager的web界面：http://$NodeIP:30902
* 访问grafana的web界面：http://$NodeIP:30903 

> (默认用户密码查看：)

```
用户名： damin
密码： 执行以下命令

# grep "adminPassword" /etc/kubeasz/roles/cluster-addon/templates/prometheus/values.yaml.j2
```
### 管理操作
#### 验证告警
* 修改prom-alertsmanager.yaml文件中邮件告警为有效的配置内容，并使用 helm upgrade更新安装
* 手动临时关闭 master 节点的 kubelet 服务，等待几分钟看是否有告警邮件发送

```
在 master 节点运行
#  systemctl stop kubelet
```
#### [可选] 配置钉钉告警
* 创建钉钉群，获取群机器人 webhook 地址
使用钉钉创建群聊以后可以方便设置群机器人，【群设置】-【群机器人】-【添加】-【自定义】-【添加】，然后按提示操作即可，     
参考 https://open-doc.dingtalk.com/docs/doc.htm?spm=a219a.7629140.0.0.666d4a97eCG7XA&treeId=257&articleId=105735&docType=1

上述配置好群机器人，获得这个机器人对应的Webhook地址，记录下来，后续配置钉钉告警插件要用，格式如下
```
https://oapi.dingtalk.com/robot/send?access_token=xxxxxxxx
```
* 创建钉钉告警插件，参考 http://theo.im/blog/2017/10/16/release-prometheus-alertmanager-webhook-for-dingtalk/
```
  编辑修改文件中 access_token=xxxxxx 为上一步你获得的机器人认证 token
# vi /etc/ansible/manifests/prometheus/dingtalk-webhook.yaml
  运行插件
# kubectl apply -f /etc/ansible/manifests/prometheus/dingtalk-webhook.yaml
```
* 修改 alertsmanager 告警配置后，更新 helm prometheus 部署，成功后如上节测试告警发送
```
修改 alertsmanager 告警配置
# cd /etc/ansible/manifests/prometheus
# vi prom-alertsmanager.yaml
增加 receiver dingtalk，然后在 route 配置使用 receiver: dingtalk
    receivers:
    - name: dingtalk
      webhook_configs:
      - send_resolved: false
        url: http://webhook-dingtalk.monitoring.svc.cluster.local:8060/dingtalk/webhook1/send
 ...
更新 helm prometheus 部署
# helm upgrade --tls monitor -f prom-settings.yaml -f prom-alertsmanager.yaml -f prom-alertrules.yaml prometheus
```
