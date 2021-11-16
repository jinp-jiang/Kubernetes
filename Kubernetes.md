# Kubernetes学习笔记（CKA）

<!--**node节点CPU、MEM的大小直接影响是否可以顺利部署pod**-->

#### 安装部分：

##### 1.新建kubernetes repo源

vim /etc/yum.repos.d/kubernetes.repo

```
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
```

##### 2.关闭setenforce 服务

```
setenforce 0
```

##### 3.安装相关服务并启动

yum install -y docker kubelet kubeadm kubectl kubernetes-cni

systemctl enable docker && systemctl start docker

systemctl enable kubelet && systemctl start kubelet

##### 4.kubeadm安装部署K8S

【kubeadm初始化时指定相关参数】kubeadm init --apiserver-advertise-address=10.0.3.5 --image-repository registry.aliyuncs.com/google_containers --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16

【kubeadm】kubeadm init   kubeadm reset

###### kuebadn init初始化master的整个过程：

【preflight】 环境检查swap是否关闭，内核大小是否足够，拉取组件镜像

【certs】生成k8s、Etcd证书（启用Https）证书目录/etc/kubernetes/pki

【kubeconfig】生成kubeconfig文件，认证文件

【kubelet-start】生成kubelet配置文件，并启动/var/lib/kubelet/kubeadm-flags.env   /var/lib/kubelet/config.yaml

【control-plane】部署管理节点组件，用镜像启动容器 /etc/kubernetes/manifests

【etcd】部署etcd数据库，用镜像启动容器

【upload-config】【kubelet】【upload-certs】上传配置文件及证书，存储到k8s

【mark-control-plane】为管理节点添加标签，添加的一个污点node-role.kubernetes.io/master 目的不要让容器分配到master

【bootstrap-token】【kubelet-finalize】为其他节点kubelet自动颁发证书使用

【addons】插件部署CoreDNS、kube-proxy

##### kubeadm初始化时报错问题及注意点

指定aipserverIP以及国内可使用的docker可用源

##### 	4.1 swap的报错需要设置

​		  vim /etc/sysconfig/kubelet 

```
KUBELET_EXTRA_ARGS="--fail-swap-on=false"
```

##### 	4.2 遇到需要忽略的报错

​		kubeadm init ----ignore-preflight-errors=Swap 如果mem和cpu不满足master的要求，也会报错

##### 	4.3 初始化时，部分镜像无法下载

​		查看需要使用到的镜像，人工下载（可以不需要人工下载，直接修改镜像源即可）

​		kubeadm config images list

​		根据查询结果，使用docker search 来下载对应版本的images

```
docker pull louwy001/kube-scheduler:v1.21.0
docker pull louwy001/kube-proxy:v1.21.0
docker pull louwy001/pause:3.4.1
docker pull louwy001/etcd:3.4.13-0
docker pull louwy001/coredns:v1.8.0
docker pull louwy001/coredns:v1.8
docker pull louwy001/coredns-coredns:v1.8.0
```

​		下载完成后，使用docker tag [已存在信息] [要求命名]

​		重命名后，使用docker rmi [删除旧信息]

##### 	4.4 初始化完成之后会返回本机master的信息如下

​		kubeadm join 192.168.31.128:6443 --token 74rl2p.1n2m2gxezowqr7zs \
​	--discovery-token-ca-cert-hash sha256:1532efc0f07b5b3d74acaf0729b7d3e461490ddc4b87671abd257f9a0ce0481b 

##### 	4.5 其他节点如需加入需要执行

​		kubeadm join --token <token> <master-ip>
​		
【kubectl加入source源可补全命令】

​	yum -y install bash-completion

​	source <(kubectl completion bash)

​	echo "source <(kubectl completion bash)" >> /etc/profile

##### 5.安装节点网络插件

在Calico官网 上下载calico.yaml文件

wget https://docs.projectcalico.org/manifests/calico.yaml  

kubectl apply -f calico.yaml

【修改IPV4网段】修改IPV4_cidr 10.244.0.0/16

【如出现calico节点ready状态0/1】pods 节点报错信息calico/node is not ready: BIRD is not ready: BGP not established with 10.0.3.10

【解决方法】在官网calico.yaml中，calico节点部分添加对应本机的网卡名

```
name: IP_AUTODETECTION_METHOD
value: "interface=eth0"
```

安装dashboard UI

wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.2.0/aio/deploy/recommended.yaml 

在service板块k8s-app下行添加type：NodePort

kubectl apply -f recommended.yaml

##### 执行该yaml文件可能会遇到的问题

##### 5.1 The connection to the server localhost:8080 was refused - did you specify the right host or port?

该报错需要配置~/.bash_profile环境变量

```
echo “export KUBECONFIG=/etc/kubernetes/admin.conf” >> ~/.bash_profile
source ~/.bash_profile
```

创建用户

kubectl create serviceaccount dashboard-admin -n kube-system

用户授权

kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin

获取token

kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}'

##### 6.新建一个带nginx服务的pod

首先需要使用docker拉取nginx的镜像

docker pull nginx:latest

然后可以通过命令或yaml文件的方式创建pod（推荐使用yaml） 

【使用deploymnet控制器部署镜像】kubectl create deployment web --image-nginx --replicas=3

【使用service暴露pod】kubectl expose deployment web --port=80 --target-port=80 --type=NodePort   （port为集群内部端口，target-port为镜像内端口）

【使用dry-run输出yaml文件】kubectl expose deployment web --port=80 --target-port=80 --type=NodePort -o yaml --dry-run=client

【Yaml文件的格式可以从kubernetes官网直接搜索】

【使用yaml文件创建deployment】kubectl create deployment web --image=nginx --replicas=3 --dry-run=client -o yaml > nginx-dep.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: web
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: web
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
```

【使用yaml文件创建service】kubectl expose deployment web --port=80 --target-port=80 --type=NodePort --dry-run=client -o yaml > nginx-svc.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web2
  namespace: default
spec:    
  selector:
    app: nginx2       指定关联的Pod标签
  ports:      
    - protocol: TCP   协议
      port: 80        service端口
      targetPort: 80  容器端口
  type: NodePort      服务类型
```

##### 6.2 整体yaml文件创建

vim nginx.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      name: nginx
  template:
    metadata:
      labels:
        name: nginx
    spec:
      containers:
        - name: nginx
          image: docker.io/nginx:latest
          imagePullPolicy: IfNotPresent
	      ports:
            - containerPort: 80
          env:
            - name: TZ
              value: Asia/Shanghai
	      volumeMounts:
            - name: volume-data
              mountPath: "/usr/share/nginx/html"
            - name: nginx-conf
              mountPath: "/etc/nginx/conf.d"
      volumes:
        - name: volume-data
          hostPath:
            path: /usr/local/nginx/html
        - name: nginx-conf
          hostPath:
            path: /usr/local/nginx/conf.d
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-nodeport
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: NodePort
  selector:
    name: nginx
```

##### 部署yaml编写的pod

kubectl apply -f nginx.yaml

##### 允许主节点部署pod

kubectl taint nodes --all node-role.kubernetes.io/master-

##### 如果不允许调度

kubectl taint nodes master1 node-role.kubernetes.io/master=:NoSchedule

##### 污点可选参数

NoSchedule: 一定不能被调度

PreferNoSchedule: 尽量不要调度

NoExecute: 不仅不会调度, 还会驱逐Node上已有的Pod

##### 7.关于kubectl的相关命令

【查看k8s节点信息】kubectl get nodes

【查看pod详情】kubectl get pod [name] -o wide

【删除pod】kubectl delete pod [name]

【查看deployment并删除】kubectl get deploymenets (查看)；kubectl delete deployments

【服务起来以后查看对应映射的端口号】kubectl get svc； 删除 kubectl delete svc [name] 

【查看pod节点详情】kubectl describe pod

【查看pod节点日志】kubectl logs -f pod_name

【查看service关联的pod】kubectl get endpoints （kubectl get ep）

【检查是否创建了副本控制器】 ReplicationController：kubectl get rc

【检查是否创建了副本集】 ：replicasets：kubectl get rs

【命令行升级并记录】kubectl set image deployment web nginx=nginx:1.18 -n namespace --record=true

【增加】--record=true 方便后续查看升级版本的描述，否则描述为NONE

【查看版本升级步骤】kubectl describe deployment web

#####   版本回滚

【查看历史版本】kubectl rollout history deployment web

【通过get rs版本查看】kubectl describe rs rs名称 | grep Image/revision 

【回滚到上一个版本】kubectl rollout undo deployment web

【回滚到指定版本】kubectl rollout undo deployment web --to-revision=2

#####   水平扩容缩容

【直接修改yaml文件】更改replicas或者其他参数，更改后直接apply yaml文件

【命令行修改】kubectl scale deployment web --replicas=10

【实时查看pod创建情况】kubectl get pods -n default -w 

##### 【Pod运行多容器，共享存储及共享网络实践】
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-test
  namespace: ms
spec:
  replicas: 3
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
        image: nginx
        volumeMounts:
        - name: log
          mountPath: /data
      - name: bs
        image: busybox
        command:
        - sleep
        - 24h
        volumeMounts:
        - name: log
          mountPath: /data2
      volumes:
      - name: log
        emptyDir: {}
```

【进入pod中的容器终端】kubectl exec -it <pod名称> -c <container名称> -n namespace -- bash/sh  【查看指定volumes下的路径文件是否是共享】

#####   Pod管理命令

【命令创建pod】kubectl run my-pod --image==nginx:1.17

【查看Pod节点详情】kubectl describe pod my-pod

【查看Pod中容器日志】kubectl logs my-pod -c nginx -f

##### Pod对象重启策略restartPolicy: Always

• Always：当容器终止退出后，总是重启容器，默认策略。
• OnFailure：当容器异常退出（退出状态码非0）时，才重启容器。
• Never：当容器终止退出，从不重启容器。

##### Pod健康检查

• livenessProbe（存活检查）：如果检查失败，将杀死容器，根据Pod的restartPolicy来操作。
• readinessProbe（就绪检查）：如果检查失败，Kubernetes会把Pod从service endpoints中剔除。 
• startupProbe（启动检查）：检查成功才由存活检查接手，用于保护慢启动容器

##### 【exec】1.检查容器中文件是否创建，如果没有被检测到pod重启

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/test.sock; sleep 12h
  livenessProbe:
    exec:
      command:
      - cat
      - /tmp/test.sock
    initialDelaySeconds: 5  容器启动后5s开始检查/tmp/test.sock 是否创建
    periodSeconds: 5        以后每次检查间隔5s检查
```

##### 【tcpSocket】 2.探测端口

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: probe-test
  name: probe-test
spec:
  containers:
  - image: nginx
    name: probe-test
    ports:
    - containerPort: 8080
    livenessProbe:
      tcpSocket:
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
    readinessProbe:
      tcpSocket:
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
```

##### 【httpGet】3.发送HTTP请求，返回200-400范围状态码为成功

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: test-readiness
  name: test-readiness
spec:
  replicas: 3
  selector:
    matchLabels:
      app: test-readiness
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: test-readiness
    spec:
      containers:
      - image: nginx
        name: nginx
        livenessProbe:
          httpGet:
            port: 80
            path: /index.html
          initialDelaySeconds: 5
          periodSeconds: 5
        readinessProbe:
          httpGet:
            port: 80
            path: /index.html
          initialDelaySeconds: 5
          periodSeconds: 5
```

#####  【readinessProbe例子deployment】如果检查失败，Kubernetes会把Pod从service endpoints中剔除

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: test-readiness
  name: test-readiness
spec:
  replicas: 3
  selector:
    matchLabels:
      app: test-readiness
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: test-readiness
    spec:
      containers:
      - image: busybox
        name: busybox
        args:
        - /bin/sh
        - -c
        - touch /tmp/test.sock; sleep 24h
      livenessProbe:
        exec:
          command:
          - cat
          - /tmp/test.sock
        initialDelaySeconds: 5
        periodSeconds: 5
      readinessProbe:
        exec:
          command:
          - cat
          - /tmp/test.sock
        initialDelaySeconds: 5
        periodSeconds: 5
```

##### 【startupProbe】检查成功才由存活检查接手，用于保护慢启动容器

##### 【pod引用环境变量env】

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-envars-fieldref
spec:
  containers:
    - name: nginx
      image: nginx
    - name: test-container-bs
      image: busybox
      command:
      - sleep
      - 24h
      env:
        - name: MY_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: ABC
          value: "12345"
```

##### 【Init Container初始化工作】deployment例子

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: init-test
  name: init-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: init-test
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: init-test
    spec:
      initContainers:
      - name: bs
        image: busybox
        command:
        - wget
        - "-O"
        - "/opt/index.html"
        - http://www.aliangedu.cn
        volumeMounts:
        - name: wwwroot
          mountPath: "/opt"
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: wwwroot
          mountPath: /usr/share/nginx/html/
      volumes:
      - name: wwwroot
        emptyDir: {}
```

##### 【静态pod特点】

• Pod由特定节点上的kubelet管理
• 不能使用控制器
• Pod名称标识当前节点名称

##### 【静态Pod配置路径】

vim /var/lib/kubelet/config.yaml 查看kubelet配置文件
...
staticPodPath: /etc/kubernetes/manifests 直接将yaml文件放置该路径下即可，kubelet会自动创建

【查看已运行服务yaml】kubectl get service web2 -o yaml

【查看标签关联的pod】kubectl get pods -l app=nginx

【查看某个Pod的标签】kubectl get pods --show-labels

【service定义与创建】官网下载service.yaml模板，添加关联Pod标签

【service的三种类型】

ClusterIP：默认类型，只在集群内部使用，外部无法访问，ClusterIP基于四层转发，pod及node都可访问，但无法ping通

NodePort：对外暴露的端口，端口范围30000~32767.同时会新建集群内部IP，并支持外网访问节点IP：NodePort

Loadbalancer：与NodePort相似，但暴露相对来说安全，不直接暴露任何node节点上的公网地址

【ingress yaml文件，可官网找文档】

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wildcard-host
spec:
  rules:
  - host: "foo.bar.com"
    http:
      paths:
      - path: /
        backend:
          service:
            name: service1
            port:
              number: 80
```

##### 【部署完整的项目java-demo】

【安装java运行环境】yum -y install java-1.8.0-openjdk maven

【配置国内阿里云的maven仓库】vim /etc/maven/setting.xml  将百度到的maven配置添加到mirrors中

【拉取git上的代码】git clone git地址

【使用maven对解压后的包进行编译】mvn clean package -Dmaven.test.skip=true

【编写dockerfile将war包打入镜像】

```dockerfile
From xxx/image:tag
LABEL maintainer www.ctns.com
RUN rm -rf /usr/local/tomcat/webapps/*
ADD target/*.war /usr/local/tomcat/webapps/Root.war
```

【使用dockerfile构建新的镜像】docker build -t java-demo:1.0 .

【将新构建的镜像推送】docker push image:tag

【编写k8s的yaml文件】引用新推送的镜像，创建deployment，并使用service（4层转发iptables）或者ingress（http/https）对外暴露端口即可

#####   K8S监控部分

【配置metrics日志监控】官网下载https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

【需要更改跳过证书认证】vim components.yaml 更改image部分为国内，并添加- --kubelet-insecure-tls

​			kubectl apply -f components.yaml

##### pod资源调度

指定pod只用node节点资源

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: request-pod
  name: request-pod
spec:
  containers:
  - image: nginx
    name: request-pod
    resources:
      requests:
        memory: 100M  所给内存大小，不得大于limits
        cpu: 0.5m     1核等于1000m
      limits:
        memory: 500M
        cpu: 1m
```

【查看node节点占用资源】kubectl describe node worknode1

【添加节点标签】kubectl label node worknode1 disk=ssd

【查看节点污点】kubectl describe nodes | grep Taint

【nodeSelector节点选择器】Pod可按照node节点标签（key:vaule）进行选择

【nodeAffinity更细致对标签做罗辑上的判断】详情可参照官网例子

##### required（必须满足）：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd            
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

##### preferred（尝试满足）：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd          
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

【Taint污点设置】避免pod调度到worknode1

NoSchedule：一定不能被调度

PreferNoSchedule：尽量不被调度，非必要配置容忍

NoExecute：不仅不会被调度，还会驱逐Node上已有的Pod

kubectl taint node work-node1 key=vaule:NoSchedule

【Tolerations污点容忍】详情可参照官网例子

```yaml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
```

```yaml
tolerations:
- key: "key1"
  operator: "Exists"
  effect: "NoSchedule"
```

【nodeName】该调度不经过调度器，直接指定node节点名nodeName: worknode1

【DaemonSet】在每个节点上运行；新加入的node也同样自动运行一个pod；应用于监控agent、网络插件、日志agent



#####   相关课后作业

#####   【查看yaml相关名词拼写可使用explain】kubectl explain deployment/pods.spec.template

【新建一个命名空间，创建一个deployment并暴露service】kubectl create ns aliang-cka;kubectl create deployment web --image=nginx -n aliang-cka;kubctl expose deployment web --port=80 --target-port=80 --type=NodePort -n aliang-cka

【列出指定命名空间下指定标签pod】kubectl get pod -l  k8s-app=kube-dns -n kube-system

【查看POD日志并输出到/opt/web】kubectl logs podname -n namespace | grep Error > /var/web

【查看制定label使用cpu最高的pod，记录到/opt/cpu】kubectl top pods -l app=web -n namespace --sort-by='cpu' | awk 'NR==2{print $1}'

【创建一个deployment副本数为3，然后滚动更新镜像版本并记录更新，最后再回滚到上一个版本】

kubectl create deployment nginx --image=nginx:1.16 --replicas=3

kubectl set image deployment nginx --nginx=nginx:1.17 --record=true

kubectl rollout history deployment nginx

kubectl rollout undo deployment  nginx --to-revision=3

【给web deployment扩容副本数3】kubetcl scale deployment web --replicas=3 

【把deployment输出到json文件】kubectl create deployment web --image=nginx -o json --dry-run=client > test.json 

【生成一个deployment yaml文件保存到/opt/deploy.yaml】kubectl create deployment web --image=nginx -o yaml --dry-run=client > /opt/deploy.yaml 

更改选择器标签及pod标签为app_env_stage=dev

【创建一个pod，分配到指定标签node上】kubectl label node worknode1 disk=ssd

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: web
  name: web
spec:
  containers:
  - image: nginx
    name: web
  nodeSelector:
    disk: "ssd"
```

【确保在每个节点上运行一个POD】

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: daemonset-test
  name: daemonset-test
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
      tolerations:
        - operator: "Exists"
          effect: "NoSchedule"
```









