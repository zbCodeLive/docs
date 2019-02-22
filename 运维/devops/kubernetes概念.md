# Kubernetes使用

### pod概念 

1. pod 状态    
- Running：运行中  
- Pending：写数据到etcd集群中异常，可能是调度node节点异常，可能是镜像下载(pull)慢，启动容器异常  
- Succeeded：容器正常中止并退出，一般是Job类pod  
- Failed：Pod 中的所有容器都已终止了，并且至少有一个容器是因为失败终止。也就是说，容器以非0状态退出或者被系统终止  
- Unknown：出于某种原因，无法获得Pod的状态，通常是由于与Pod主机通信时出错

2. pod镜像拉取策略(ImagePullPolicy)
支持三种ImagePullPolicy
- Always：不管镜像是否存在都会进行一次拉取。
- Never：不管镜像是否存在都不会进行拉取  
- IfNotPresent：只有镜像不存在时，才会进行镜像拉取。  
注意：  
默认为IfNotPresent，但:latest标签的镜像默认为Always。  
拉取镜像时docker会进行校验，如果镜像中的MD5码没有变，则不会拉取镜像数据。  
生产环境中应该尽量避免使用:latest标签，而开发环境中可以借助:latest标签自动拉取最新的镜像。

3. 资源限制  
Kubernetes通过cgroups限制容器的CPU和内存等计算资源，包括requests（请求，调度器保证调度到资源充足的Node上）和limits（上限）等:

- spec.containers[].resources.limits.cpu：CPU上限，可以短暂超过，容器也不会被停止  
- spec.containers[].resources.limits.memory：内存上限，不可以超过；如果超过，容器可能会被停止或调度到其他资源充足的机器上  
- spec.containers[].resources.requests.cpu：CPU请求，可以超过  
- spec.containers[].resources.requests.memory：内存请求，可以超过；但如果超过，容器可能会在Node内存不足时清理    

```
限制cpu与memory示例

apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: db
    image: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "password"
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: wp
    image: wordpress
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

4. 容器重启策略(restartPolicy: Always)：      
- Always: 当容器中止退出后，总是重启容器，默认策略  
- OnFailure：当容器异常退出，(退出状态码为非0)时，才重启容器  
- Never：当容器中止退出，从不重启容器

5. Probe 健康检查  
为了确保容器在部署后确实处在正常运行状态，Kubernetes提供了两种探针（Probe，支持exec、tcp和httpGet方式）来探测容器的状态:  

- livenessProbe：如果检查失败，将容器杀死，根据pod的restartPolicy来操作
- readnessProbe：如果检查失败，Kubernetes会把pod从service endpoints中剔除  
Probe支持三种检查方式  
- httpGet：发起请求，返回200-400范围状态码为成功  
- exec：执行shell命令，返回状态码为0即为成功  
- tcpSocket：发起TCP Socket建立成功

6. 调度约束


7. 故障排查


### pod控制器
1. Deployment：部署无状态服务比如应用等   
部署示例：

#### 创建
使用yaml新建pod:`kubectl create -f pod.yaml`
```
# pod.yaml如下：
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 3 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14
        ports:
        - containerPort: 80
```
使用`kubectl get deployments -o wide`查看deployments创建情况
```
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS   IMAGES       SELECTOR
nginx-deployment   3         0         0            0           2d2h   nginx        nginx:1.15   app=nginx
```
使用`kubectl get pod -o wide`查看pod是否正常运行
```
NAME                                READY   STATUS             RESTARTS   AGE    IP            NODE            NOMINATED NODE
nginx-deployment-5d445489fb-2rv5b   1/1     Running            0          2d2h   172.17.86.3   192.168.2.103   <none>
nginx-deployment-5d445489fb-tjz6d   1/1     Running            1          2d2h   172.17.53.2   192.168.2.102   <none>
```
- NAME 列出群集中的部署名称。  
- DESIRED显示应用程序的所需副本数，您在创建部署时定义这些副本。这是理想的状态。  
- CURRENT 显示当前正在运行的副本数量。  
- UP-TO-DATE 显示已更新以实现所需状态的副本数。  
- AVAILABLE 显示您的用户可以使用的应用程序副本数。  
- AGE 显示应用程序运行的时间。  

请注意每个字段中的值如何与部署规范中的值相对应：

- 根据.spec.replicas字段，所需副本的数量为3 。
- 根据.status.replicas字段，当前副本的数量为3 。
- 根据.status.updatedReplicas字段，最新副本的数量为3 。
- 根据.status.availableReplicas字段，可用副本的数量为3 。

#### 更新
例如需要将nginx的版本更新到1.15.7，只需将pod.yaml中的nginx版本进行更改，执行如下命令(即可将镜像版本更新到1.15.7)
`kubectl apply -f pod.yaml`

也可执行如下命令进行更新：`kubectl set image nginx-deployment nginx=nginx:1.15.7 --record`
"--record"是将命令进入kubernetes中方便查看

查看命令是否执行成功:
```
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-1564180365   3         3         3       6s
nginx-deployment-2035384211   0         0         0       36s
```
原来的副本数变为0个，新副本数更新到3个

kubectl apply -f pod.yaml可实现无缝更新
pod更新时，pod存活的数量会在2-4个范围内

#### 回滚
```
查看版本
kubectl rollout history deployment/nginx-deployment

回滚到上一版本
kubectl rollout undo deployment/nginx-deployment

回滚到执行版本
kubectl rollout undo deployment/nginx-deployment --to-revision=3
```

#### 扩展部署
扩展nginx副本数为10个
`kubectl scale deployment/nginx-deployment --replicas=10`

2. StatefulSets:部署有状态服务比如rabbitmq集群，redis集群等
```
创建statefulset.yaml(无头服务需将ClusterIP设置为None)

apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: StatefulSet
metadata:
  name: nginx-statefulset
spec:
  serviceName: nginx-stateful
  selector:
    matchLabels:
      app: nginx-sts
  replicas: 3 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx-sts
    spec:
      containers:
      - name: nginx
        image: nginx:1.15
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-stateful
  labels:
    app: nginx-sts
spec:
  ports: 
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx-sts
```

扩容缩容与deployment一致(需要注意的是缩容扩容后新增的pod节点数据问题)

3. DaemonSet  


4. Job  
5. CronJob

6. secret  
7. configmap

- 针对r文件
```
创建configmap(--from-file可传入单个文件，多个文件，文件夹)

使用已有文件创建configmap
mkdir -p /opt/configmap
wget https://k8s.io/docs/tasks/configure-pod-container/configmap/kubectl/game.properties -O /opt/configmap
kubectl create configmap game-config --from-file=/opt/configmap/game.properties


使用yaml创建configmap(导出yaml：kubectl get configmap game-config -o yaml)
game-configmap.yaml如下：
apiVersion: v1
data:
  game.properties: |-
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAS
    secret.code.allowed=true
    secret.code.lives=30
kind: ConfigMap
metadata:
  creationTimestamp: 2018-12-19T03:25:44Z
  name: game-config2
  namespace: default
  resourceVersion: "1282775"
  selfLink: /api/v1/namespaces/default/configmaps/game-config2
  uid: c54eefcb-033d-11e9-9e23-005056b61cef
  
kubectl create -f game-configmap.yaml
```

```
使用configmap

# 新建容器configmap-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-configmap-pod
spec:
  containers:
    - name: test-container
      image: busybox:1.28.4
      command: [ "/bin/sh", "-c", "cat /etc/config/game.properties" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config/
  volumes:
    - name: config-volume
      configMap:
        # Provide the name of the ConfigMap containing the files you want
        # to add to the container
        name: game-config2
  restartPolicy: Never

kubectl create -f configmap-test.yaml
```


### pod 常用命令使用
`kubectl create -f pod.yaml`

将pod作为服务暴露到外部进行访问
```
kubectl expose deployment nginx-deployment --port=8080 --target-port=80 --name=nginx-service --type=NodePort

expose：端口暴露命令
deployment：控制器类型
nginx-deployment：pod name
port：clusterIP的端口
target-port：容器中服务的端口
name：service服务的名称，任意命名但不能重复
type：暴露端口类型，默认ClusterIp,NodePort,LoadBalance,ExternalName

返回结果如下：
[root@CentOS7 pod-yaml]# kubectl get svc -o wide
NAME            TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE     SELECTOR
kubernetes      ClusterIP   10.0.0.1     <none>        443/TCP          3d16h   <none>
nginx-service   NodePort    10.0.0.36    <none>        8080:41787/TCP   14s     app=nginx

使用节点IP:41787可访问nginx欢迎页面(注意不要使用master节点)
```


```
[root@CentOS7 pod-yaml]# kubectl get endpoints
NAME            ENDPOINTS                               AGE
kubernetes      192.168.2.100:6443,192.168.2.101:6443   3d16h
nginx-service   172.17.53.2:80,172.17.53.3:80           8m16s
```



```
查看pod详细信息
kubectl describe pod nginx-deployment-5d445489fb-2cc7c

查看deployment详细信息
kubectl describe deployment nginx-deployment
```

```
使用run命令导出yaml文件
kubectl run nginx --images=nginx:latest --port=80 --replicas=3 --dry-run -o yaml > test-nginx-deployment.yaml
```

```
用get命令导出，然后作为模板进行编辑
kubectl get deployment/nginx -o=yaml --export  > new.yaml
```

```
字段补全，pod可由kind中的字段代替，例如：deployment statefulset daemonset等
kubectl explain pod.spec.affinity.podAffinity
```

```
获取namespace
kubectl get ns
```

pod动态扩容 


kubectl logs
```
输出pod中一个容器的日志。如果pod只包含一个容器则可以省略容器名。

kubectl logs [-f] [-p] POD [-c CONTAINER]
示例
# 返回仅包含一个容器的pod nginx的日志快照
$ kubectl logs nginx

# 返回pod ruby中已经停止的容器web-1的日志快照
$ kubectl logs -p -c ruby web-1

# 持续输出pod ruby中的容器web-1的日志
$ kubectl logs -f -c ruby web-1

# 仅输出pod nginx中最近的20条日志
$ kubectl logs --tail=20 nginx

# 输出pod nginx中最近一小时内产生的所有日志
$ kubectl logs --since=1h nginx
```


### kubernetes rbac权限控制

创建角色
```
新建role.yaml

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

创建集群角色  
(集群角色设置namespace则限制role的命名空间范围，若不设置namespace则默认该权限可访问所有的namespace)
```
新建clusterRole.yaml

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  # "namespace" omitted since ClusterRoles are not namespaced
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

角色绑定
```
新建roleBinding.yaml

# This role binding allows "jane" to read pods in the "default" namespace.
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```

集群角色绑定
```
新建clusterRoleBinding.yaml

# This role binding allows "dave" to read secrets in the "development" namespace.
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-secrets
  namespace: development # This only grants permissions within the "development" namespace.
subjects:
- kind: User
  name: dave # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

查看apiversions,apigroups命令如下:
`kubectl api-versions`
```
(apiGroups为下列返回/前的名称)
[root@CentOS7 ~]# kubectl api-versions
admissionregistration.k8s.io/v1beta1
apiextensions.k8s.io/v1beta1
apiregistration.k8s.io/v1
apiregistration.k8s.io/v1beta1
apps/v1
apps/v1beta1
apps/v1beta2
authentication.k8s.io/v1
authentication.k8s.io/v1beta1
authorization.k8s.io/v1
authorization.k8s.io/v1beta1
autoscaling/v1
autoscaling/v2beta1
autoscaling/v2beta2
batch/v1
batch/v1beta1
certificates.k8s.io/v1beta1
coordination.k8s.io/v1beta1
events.k8s.io/v1beta1
extensions/v1beta1
networking.k8s.io/v1
policy/v1beta1
rbac.authorization.k8s.io/v1
rbac.authorization.k8s.io/v1beta1
scheduling.k8s.io/v1beta1
storage.k8s.io/v1
storage.k8s.io/v1beta1
v1
```

resources：nodes, deployments, statefulsets, daemonsets, comfigmaps, secrets, jobs, cronjobs等

verbs：get, list, watch, create, update, patch, delete, post等

使用命令行创建用户并绑定角色：  
授予cluster-admin ClusterRole到整个集群命名为“root”用户：
```
kubectl create clusterrolebinding root-cluster-admin-binding --clusterrole=cluster-admin --user=root
```

授予system:node ClusterRole到一个名为“kubelet”整个集群用户：
```
kubectl create clusterrolebinding kubelet-node-binding --clusterrole=system:node --user=kubelet
```

将权限授予view ClusterRole整个群集中名称空间“acme”中名为“myapp”的服务帐户：
```
kubectl create clusterrolebinding myapp-view-binding --clusterrole=view --serviceaccount=acme:myapp
```

#### ceph挂载
1. 指定主机挂载
```
# cephfs.yaml
apiVersion: v1
kind: Pod
metadata:
  name: cephfs
spec:
  containers:
  - name: cephfs-rw
    image: nginx
    volumeMounts:
    - mountPath: "/mnt/cephfs"
      name: cephfs
  volumes:
  - name: cephfs
    cephfs:
      monitors:
      - 192.168.1.21:6789
      # - 192.168.1.22:6789
      # - 192.168.1.23:6789
      # by default the path is /, but you can override and mount a specific path of the filesystem by using the path attribute
      # path: /home/cephfs
      user: admin
      secretRef:
        name: ceph-secret
      readOnly: false

```

2. 静态pv与pvc
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-static
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  cephfs:
    monitors: 
      - 192.168.1.21:6789
    path: /
    user: admin
    secretRef:
      name: ceph-secret
    readOnly: false

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-static
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi

---
apiVersion: v1
kind: Pod
metadata:
  name: pv-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    volumeMounts:
      - name: www
        mountPath: /usr/share/nginx/html
  volumes:
    - name: www
      persistentVolumeClaim:
        claimName: pvc-static
```

3. 动态pv与pvc

1). 创建admin-secret
```
ceph auth get client.admin 2>&1 |grep "key = " |awk '{print  $3'} |xargs echo -n > /tmp/secret
kubectl create ns cephfs
kubectl create secret generic ceph-secret-admin --from-file=/tmp/secret --namespace=cephfs
```

2). 启动Cephfs provisioner
```
# clusterrolebinding.yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cephfs-provisioner
subjects:
  - kind: ServiceAccount
    name: cephfs-provisioner
    namespace: cephfs
roleRef:
  kind: ClusterRole
  name: cephfs-provisioner
  apiGroup: rbac.authorization.k8s.io
  
# clusterrole.yaml
metadata:
  name: cephfs-provisioner
  namespace: cephfs
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
  - apiGroups: [""]
    resources: ["services"]
    resourceNames: ["kube-dns","coredns"]
    verbs: ["list", "get"]

# deployment.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: cephfs-provisioner
  namespace: cephfs
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: cephfs-provisioner
    spec:
      containers:
      - name: cephfs-provisioner
        image: "quay.io/external_storage/cephfs-provisioner:latest"
        env:
        - name: PROVISIONER_NAME
          value: ceph.com/cephfs
        command:
        - "/usr/local/bin/cephfs-provisioner"
        args:
        - "-id=cephfs-provisioner-1"
      serviceAccount: cephfs-provisioner
      
# rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cephfs-provisioner
  namespace: cephfs
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: cephfs-provisioner
subjects:
- kind: ServiceAccount
  name: cephfs-provisioner
  
# role.yaml
kind: Role
metadata:
  name: cephfs-provisioner
  namespace: cephfs
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["create", "get", "delete"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
    
# serviceaccount.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cephfs-provisioner
  namespace: cephfs
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["create", "get", "delete"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]

# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cephfs-provisioner
  namespace: cephfs

```

3). 创建一个storageclass
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: cephfs
  namespace: cephfs
provisioner: ceph.com/cephfs
parameters:
    monitors: 192.168.1.21:6789
    adminId: admin
    adminSecretName: ceph-secret-admin
    adminSecretNamespace: "cephfs"
```
检查cephfs storageclass是否创建成功:
```
[root@centos7 cephfs]# kubectl get storageclass -n cephfs
NAME     PROVISIONER       AGE
cephfs   ceph.com/cephfs   38m

```

4). 创建PVC使用cephfs storageClass动态分配PV
```
# claim.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: claim1
  namespace: cephfs
spec:
  storageClassName: cephfs
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```
检查动态分配是否成功
```
[root@centos7 cephfs]# kubectl get storageclass -n cephfs
NAME     PROVISIONER       AGE
cephfs   ceph.com/cephfs   38m
[root@centos7 cephfs]# kubectl get pvc -n cephfs
NAME     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
claim1   Bound    pvc-be9c2296-0f39-11e9-87de-005056b661e1   1Gi        RWX            cephfs         36m
```

5). 接下来我们创建一个Pod绑定该PVC,来验证是否成功的将cephfs绑定的Pod中
```
# test-rbac.yaml
apiVersion: v1
kind: Pod
metadata:
  name: rbac-pod
  namespace: cephfs
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    volumeMounts:
      - name: www
        mountPath: /usr/share/nginx/html
  volumes:
    - name: www
      persistentVolumeClaim:
        claimName: claim1
```