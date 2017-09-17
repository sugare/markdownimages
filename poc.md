# POC 事项清单
- DCE 的平台部署
- Portworx 分部署存储部署
- DataLake 的部署
- prometheus 监控的部署
- AutoScale 弹性伸缩的部署
- DCS 的部署（可选项） 
- 相关插件的部署 （可选项）



## DCE 的平台部署步骤如下:
1. 基础环境的准备 （安装包，基础设施环境的准备（存储，网络，计算））
2. 安装部署平台
3. license 申请
4. 基础平台搭建完毕以及基本验证


**提前准备好相关的介质 [ 环境前提：完全离线 ]：**

  - CentOS everthing iso 镜像文件
  - DCE 安装介质（DCE的tar.gz包）
  - docker 的存储驱动方式设置脚本 install01.sh

**安装步骤：**
- 通过fdisk 给docker 分区

- 设置 docker 使用的director-loop 存储

- 安装docker 以及 kubelet

- 安装 DCE 平台

- 激活 DCE 平台，两种方式 （1，在线激活。2，离线激活，地址：https://license.daocloud.io/offline-license）




**1. 创建docker使用的分区**
```
cat >part.txt <<EOF
n
p



wq
EOF
printf "请输入分区:";read volume;
echo "设置完成!"
cat part.txt | fdisk $volume
partprobe $volume
partprobe $volume
partprobe $volume
kpartx $volume
kpartx $volume
```
**2. 使docker存储设置成devicemapper-lvm的存储格式**
```
printf "请确定当前不是即将安装平台控制节点"
printf "安装DCE主控节点脚本!!! 请注意一定要提前分区且不要格式化!!!"
pvcreate /dev/sda3  ### 注意这个盘区是不是刚刚创建的分区
vgcreate docker /dev/sda3 ###  注意这个盘区是不是刚刚创建的分区
lvcreate --wipesignatures y -n thinpool docker -l 95%VG
lvcreate --wipesignatures y -n thinpoolmeta docker -l 1%VG
lvconvert -y --zero n -c 512K --thinpool docker/thinpool --poolmetadata docker/thinpoolmeta
cat >> /etc/lvm/profile/docker-thinpool.profile <<EOF
activation {
  thin_pool_autoextend_threshold=80
  thin_pool_autoextend_percent=20
}
EOF
cat /etc/lvm/profile/docker-thinpool.profile
lvchange --metadataprofile docker-thinpool docker/thinpool
lvs -o+seg_monitor
# 8. Prepare docker to use devicemapper
mkdir  /etc/docker
cat >> /etc/docker/daemon.json <<EOF
{
    "storage-driver": "devicemapper",
    "storage-opts": [
    "dm.thinpooldev=/dev/mapper/docker-thinpool",
    "dm.use_deferred_removal=true",
    "dm.use_deferred_deletion=true"
    ],
 "registry-mirrors": [
        "http://dce.m.daocloud.io"
    ],
    "insecure-registries": [
        "0.0.0.0/0"
    ]
}
EOF
cat /etc/docker/daemon.json
```

**3. 安装 docker-ce 和 kubelet**
```
tar -xf dce-2.7.15.tar
cd dce-2.7.15/
./dce-installer prepare-docker   ### 安装docker 以及 kubelet
systemctl enable kubelet        ### 设置开机自启动kubelet
```
**4. 安装DCE**

# 部署安装 Portworx 分布式存储 
***准备三个机器节点 注意不要将三个etcd 节点部署在控制节点上 尽量部署在计算节点上***

**1. 部署 Portworx 的 etcd kv存储**

部署第一个etcd节点

```
mkdir /var/local/px_etcd
export HostIP="192.168.1.24" 
docker run -d -v /var/local/px_etcd:/data -p 4001:4001 -p 2380:2380 -p 2379:2379 \
--name px_etcd daocloud.io/daocloud/px-etcd:2.3.8 \
-name pwx1 \
-initial-advertise-peer-urls http://${HostIP}:2380 \
-listen-client-urls http://0.0.0.0:2379,http://0.0.0.0:4001 \
-listen-peer-urls http://0.0.0.0:2380 \
-advertise-client-urls http://${HostIP}:2379,http://${HostIP}:4001 \
-initial-cluster-token px-etcd-token \
-initial-cluster pwx1=http://192.168.1.24:2380,pwx2=http://192.168.1.25:2380,pwx3=http://192.168.1.26:2380 \
-initial-cluster-state new
```

部署第一个etcd节点
```
mkdir /var/local/px_etcd
export HostIP="192.168.1.25"
docker run -d -v /var/local/px_etcd:/data -p 4001:4001 -p 2380:2380 -p 2379:2379 \
--name px_etcd daocloud.io/daocloud/px-etcd:2.3.8 \
-name pwx2 \
-initial-advertise-peer-urls http://${HostIP}:2380 \
-listen-client-urls http://0.0.0.0:2379,http://0.0.0.0:4001 \
-listen-peer-urls http://0.0.0.0:2380 \
-advertise-client-urls http://${HostIP}:2379,http://${HostIP}:4001 \
-initial-cluster-token px-etcd-token \
-initial-cluster pwx1=http://192.168.1.24:2380,pwx2=http://192.168.1.25:2380,pwx3=http://192.168.1.26:2380  \
-initial-cluster-state new
```

部署第三个节点
```
mkdir /var/local/px_etcd
# HostIP 修改为当主机的IP
export HostIP="192.168.1.26"
docker run -d -v /var/local/px_etcd:/data -p 4001:4001 -p 2380:2380 -p 2379:2379 \
--name px_etcd daocloud.io/daocloud/px-etcd:2.3.8 \
-name pwx3 \
-initial-advertise-peer-urls http://${HostIP}:2380 \
-listen-client-urls http://0.0.0.0:2379,http://0.0.0.0:4001 \
-listen-peer-urls http://0.0.0.0:2380 \
-advertise-client-urls http://${HostIP}:2379,http://${HostIP}:4001 \
-initial-cluster-token px-etcd-token \
-initial-cluster pwx1=http://192.168.1.24:2380,pwx2=http://192.168.1.25:2380,pwx3=http://192.168.1.26:2380 \
-initial-cluster-state new
```

**2. etcd 集群的验证方式**
```
[root@dce04-px ~]# docker exec -it a65 /etcdctl member list

2d1de3940854d01c: name=pwx3 peerURLs=http://192.168.1.26:2380 clientURLs=http://192.168.1.26:2379,http://192.168.1.26:4001 isLeader=true
5457ac7d0ff05628: name=pwx2 peerURLs=http://192.168.1.25:2380 clientURLs=http://192.168.1.25:2379,http://192.168.1.25:4001 isLeader=false
8e85a21b5b14768a: name=pwx1 peerURLs=http://192.168.1.24:2380 clientURLs=http://192.168.1.24:2379,http://192.168.1.24:4001 isLeader=false 
```

**3. 部署 Portworx的Lighthouse 组件**

注意要安装kernel-devel 然后部署px-enterprise

```
yum install -y kernel-devel kernel-headers
```

在DCE的应用模板点击应用模板导入px-lighthouse-kube_v2.0.zip或者创建应用模板,模板内容如下:

```
apiVersion: v1
kind: Service
metadata:
  name: influxdb
  labels:
    app: px-lighthouse
spec:
  ports:
    - port: 8086
  selector:
    app: px-lighthouse
    tier: db
  clusterIP: None
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: influxdb
  labels:
    app: px-lighthouse
spec:
  template:
    metadata:
      name: influxdb
      labels:
        app: px-lighthouse
        tier: db
    spec:
      containers:
      - name: influxdb
        image: {{ REGISTRY }}/px-influxdb:{{ INFLUXDB_VERSION }}
        env:
          - name: ADMIN_USER
            value: {{ INFLUXDB_ADMIN_USER }}
          - name: INFLUXDB_INIT_PWD
            value: {{ INFLUXDB_INIT_PWD }}
          - name: PRE_CREATE_DB
            value: "px_stats"
---
apiVersion: v1
kind: Service
metadata:
  name: px-lighthouse
  labels:
    app: px-lighthouse
spec:
  ports:
    - port: 80
      nodePort: {{ PUBLISH_PORT }}
  selector:
    app: px-lighthouse
    tier: frontend
  type: NodePort
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: px-lighthouse
  labels:
    app: px-lighthouse
spec:
  template:
    metadata:
      name: px-lighthouse
      labels:
        app: px-lighthouse
        tier: frontend
    spec:
      containers:
      - name: px-lighthouse
        image: {{ REGISTRY }}/px-lighthouse:{{ LIGHTHOUSE_VERSION }}
        args: ["-d", "http://{{ INFLUXDB_ADMIN_USER }}:{{ INFLUXDB_INIT_PWD }}@influxdb:8086", "-k", "{{ ETCDS }}"]

```
**4. 在每个存储节点上部署 Portworx Enterprise 容器**

推荐网卡两块  M 表示管理网络 D 表示数据网络 不需要路由网关的 s 分布式存储 -t px集群加入的auth-token API_SERVER 是px-Lighthouse的暴露的IP:PORT

```
docker run --restart=always --name px-enterprise -d --net=host --privileged=true \
-v /run/docker/plugins:/run/docker/plugins \
-v /var/lib/osd:/var/lib/osd:shared \
-v /dev:/dev \
-v /etc/pwx:/etc/pwx \
-v /opt/pwx/bin:/export_bin:shared \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /var/cores:/var/cores \
-v /usr/src:/usr/src \
-e API_SERVER=http://192.168.1.21:32080 \
daocloud.io/daocloud/px-enterprise -t token-a9c83bd9-9abe-11e7-98b7-62d897e5003b \
-s /dev/sdb  -m ens160 -d ens160  
```

**5. 初始化加入集群，以及查看存储状态信息**

```
/opt/pwx/bin/pxctl status
docker logs -f px容器id
```

**6. 在无存储的节点部署px存储插件**

```
docker run --restart=always --name px-enterprise -d --net=host --privileged=true \
-v /run/docker/plugins:/run/docker/plugins \
-v /var/lib/osd:/var/lib/osd:shared \
-v /dev:/dev \
-v /etc/pwx:/etc/pwx \
-v /opt/pwx/bin:/export_bin:shared \
-v /var/run/docker.sock:/var/run/docker.sock \
-v /var/cores:/var/cores \
-v /usr/src:/usr/src \
-e API_SERVER=http://192.168.1.21:32080 \
daocloud.io/daocloud/px-enterprise -t token-a9c83bd9-9abe-11e7-98b7-62d897e5003b \
-m ens160 -d ens160

```


# 弹性伸缩
**1.第一步 导入heapster 镜像**

**2.第二步 在kube-system 名称空间下 部署heapster**

编排文件如下：

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
    k8s-app: heapster
    kubernetes.io/cluster-service: "true"
    version: v1.3.0
  name: heapster-v1.3.0
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: heapster
      version: v1.3.0
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ""
      labels:
        k8s-app: heapster
        version: v1.3.0
    spec:
      containers:
      - command:
        - /heapster
        - --source=kubernetes.summary_api:''
        image: 192.168.1.21/daocloud/heapster-amd64:v1.3.0  #### 修改写仓库地址
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /healthz
            port: 8082
            scheme: HTTP
          initialDelaySeconds: 180
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 5
        name: heapster
        resources:
          limits:
            cpu: 88m
            memory: 204Mi
          requests:
            cpu: 88m
            memory: 204Mi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      tolerations:
      - key: CriticalAddonsOnly
        operator: Exists
---
apiVersion: v1
kind: Service
metadata:
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: Heapster
  name: heapster
  namespace: kube-system
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8082
  selector:
    k8s-app: heapster
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

**3. 安装插件 autosacle-k**

**4. 压力测试ab -c 100 -n 10000 http://2048 观察DCE应用pod 是否已经扩展**

# POC-容器化部署DataLake dce-k (dce version >2.6.1)

**1.安装准备**

a. 准备介质

镜像用途 | 镜像
---|---
datalake服务端 | daocloud.io/daocloud/datalake_server:2.8.0-rc12
logspout | daocloud.io/daocloud/logspout
datalake插件 | dce K模式： daocloud.io/daocloud/dce_datalake_plugin:2.8.0-rc12-k

b.dce环境，并使用dce管理员账户登陆进行安装操作

**2.部署datalake server**

进入dce的“交付中心”－》“应用模版”，点击“创建应用模块”，复制下面的应用模版,保存为“datalake”

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: {{ APP_NAME }}-datalake
  labels:
    app: {{ APP_NAME }}-datalake
spec:
  template:
    metadata:
      name: {{ APP_NAME }}-datalake
      labels:
        app: {{ APP_NAME }}-datalake
    spec:
      containers:
      - name: {{ APP_NAME }}-datalake
        image: daocloud.io/daocloud/datalake_server:2.8.0-rc12
        ports:
        - containerPort: 8000
          name: datalake-0
        - containerPort: 8089
          name: datalake-1
        - containerPort: 5514
          name: datalake-2
        - containerPort: 5515
          name: datalake-3
        env:
          - name: DCE_URL
            value: https://admin:admin@192.168.4.119
          - name: SPLUNK_START_ARGS
            value: --accept-license --answer-yes
          - name: SPLUNK_USER
            value: root                    
---
apiVersion: v1
kind: Service
metadata:
  name: {{ APP_NAME }}-datalake
spec:
  type: NodePort
  ports:
    - targetPort: '8000'
      protocol: TCP
      port: 8000
      name: datalake-0
    - targetPort: '8089'
      protocol: TCP
      port: 8089
      name: datalake-1
    - targetPort: '5514'
      protocol: UDP
      port: 5514
      name: datalake-2
    - targetPort: '5515'
      protocol: UDP
      port: 5515
      name: datalake-3
  selector:
    app: {{ APP_NAME }}-datalake
```
**请需要修改其中的环境变量：“ DCE_URL: https://admin:admin@192.168.4.108 “，“192.168.4.108” 为该dce的访问地址，其中“admin:admin” 为该dce超级管理员用户和密码**


![datalake](https://raw.githubusercontent.com/Mayershi/markdownimages/master/datalake.png)

```
如上图，应用一共有4个端口：
8000：DataLake 服务地址的端口，访问方式是https。 （如图中ip是：192.168.4.83，对应的主机端口是30096，则访问：https://192.168.4.83:30096）
8089，DataLake 服务 API 的端口，访问方式是https。（如图中ip是：192.168.4.83，对应的主机端口是31836，则访问：https://192.168.4.83:31836）
5514，datalake syslog 数据输入端口，可以接收syslog协议的数据，协议是udp。（如图中ip是：192.168.4.83，对应的主机端口是32624，则访问：syslog+udp://192.168.4.83:32624）
5515，datalake lospout 数据输入端口，可以接收syslog协议的数据，协议是udp 。 （如图中ip是：192.168.4.83，对应的主机端口是31015，则访问：syslog+udp://192.168.4.83:31015）
启动成功后，访问DataLake 服务地址(即上面8000端口对应的访问地址“https://192.168.4.83:31015”)  
```

1. 点击“第一次登陆”，并根据界面提示的用户名／密码登陆，
2. 重新设置datalake密码为：dangerous （或别的）

![splunk.png](https://raw.githubusercontent.com/Mayershi/markdownimages/master/splunk.png)

**3.部署datalake 日志收集**

进入dce的“交付中心”－》“应用模版”，点击“创建应用模块”，复制下面的应用模版,保存为“datalakelogspout”
 
```
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: {{ APP_NAME }}-logspout
  labels:
    app: {{ APP_NAME }}-logspout
spec:
  template:
    metadata:
      name: {{ APP_NAME }}-logspout
      labels:
        app: {{ APP_NAME }}-logspout
        io.daocloud.dce.compose.mode: Global
    spec:
      containers:
      - name: {{ APP_NAME }}-logspout
        image: daocloud.io/daocloud/logspout
        ports:
          - containerPort: 3306
        command: ["/bin/logspout"]
        args: [ syslog+udp://datalake_server_ip:5515 ]
        volumeMounts:
        - name: docker
          mountPath: /var/run/docker.sock
        resources:
          limits:
            cpu: "0.5"
            memory: "1073741824"
      volumes:
      - name: docker
        hostPath:
          path: /var/run/docker.sock
---
apiVersion: v1
kind: Service
metadata:
  name: {{ APP_NAME }}-logspout
  labels:
    app: {{ APP_NAME }}-logspout
spec:
  type: NodePort
  ports:
  - port: 3306
  selector:
    app: {{ APP_NAME }}-logspout
```

```
修改上面的“syslog+udp://datalake_server_ip:5515”，“datalake_server_ip:5515”换成上面“datalake logspout数据输入端口“对应的地址(如上图实例即为：syslog+udp://192.168.4.83:31015)
点击部署，（dce目前版本不支持：DaemonSet，因此部署会一直卡在最后一步，你不用管它，等待2分钟左右，点击dce左侧菜单的“容器”，搜索“logspout”,如果容器状态为运行，即表示安装成功。注：也可以点击容器，进入容器界面，点击日志进行查看日志信息）
```

**4.部署DataLake插件**

```bash
在dce “模块中心”－“模块商店”内 “DataLake”模块,点击“安装”，根据提示进行安装，等待安装成功
然后点击“模块中心”－“模块管理”，找到“DataLake”模块，查看插件状态，如是“已启用”，表示DataLake插件安装成功，（更新版本可以点击“datalake”右边的“更新”，然后选择对应版本进行更新）
安装完成后，刷新浏览器，可以看到dce左侧菜单 “DataLake ”-->"设置"，如下图，进入DataLake插件的设置界面，
```

![datalake-plugin](https://raw.githubusercontent.com/Mayershi/markdownimages/master/datalake-plugin.png)


```
需要通过dce管理员进行以下配置：
DataLake 服务地址：为步骤一的“DataLake 服务地址”
DataLake 服务 API 地址：为步骤一的“DataLake 服务 API 地址”
DataLake 用户名：为步骤一的用户名：admin
DataLake 密码：为步骤一设置的密码
点击“保存”，保存设置，即可进行DataLake插件功能的正常使用
点击dce，左侧菜单 “DataLake ”，即可看到“datalake”的“看板”界面，如下图，表示安装成功
```
![view.png](C:/Users/mayer_shi/Desktop/部署文档/view.png "")
![view.png](https://raw.githubusercontent.com/Mayershi/markdownimages/master/view.png)

# 监控解决方案 k8s-dce 安装 kube-prometheus

**准备Prometheus 部署包：**

[kube-prometheus_one.tar.gz](http://dwiki.daocloud.io/download/attachments/12017626/kube-prometheus_one.tar.gz?version=1&modificationDate=1501814319016&api=v2 "")

部署过程：

**1. 下载部署包到k8s-dce master 节点，并解压安装包，会得到 kube-prometheus_one 文件夹和deploy.sh 部署脚本 。**

```
tar -xf  kube-prometheus_one.tar.gz
```

**2. 执行 deploy.sh 脚本部署。**

```
bash  deploy.sh   deploy
```
等待部署完成。

**3. 打开dce页面，选择 monitoring 租户，等待应用部署完成后即可打开 grafana 页面。**

![grafana](https://raw.githubusercontent.com/Mayershi/markdownimages/master/grafana.png)