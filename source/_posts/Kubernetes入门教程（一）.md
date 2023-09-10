---
title: Kubernetes入门教程（一）
author: Neil Ning
date: 2023-09-10 22:49:18
tags: ['kubernetes', 'minikube']
categories: 学习
cover: bg.jpeg
---
## kubernetes简介
Kubernetes是一个开源的容器编排引擎，已经是主流的应用部署工具，作为开发人员有必要对这个工具有一定的了解，下面对Kubernetes做一个简单的介绍。

Kubernetes集群由若干工作节点(work machine)组成，他们被称为nodes。管理这些工作节点组件被被称为控制面（Control Plane）或者主节点，Control Plane由多个组件构成，每个组件负责不同的功能。下面是Kubernetes集群的组件架构图。
![post1.png](post1.png)
控制面相当于集群中的指挥官，控制整个集群的资源和状态，主要由以下组件构成：
- kube-apiserver，控制面向其他各个组件的发指令的枢纽，也是集群中各个组件相互沟通的枢纽。他也暴露一些API，后面通过kubectl运行命令就是向apiserver发送请求。
- etcd，一致且高可用的键值存储，用作Kubernetes集群数据的数据库。
- kube-scheduler，负责监视、创建Node或者Pod。
- kube-controller-manager，控制节点、任务等。

![post2.png](post2.png)

上图是工作节点的架构图，工作节点是执行具体任务的单元，可以是一个物理机或者虚拟机，工作及诶点主要由以下组件构成：
- kubelet，管理节点上的pod，确保pod或容器的运行状态，向apiserver报告某个节点的状态，也接收来自apiserver指令。
- kube-proxy，节点的网络代理，维护一些网络规则，允许节点内pod和外界的通信。
- container runtime，容器运行时,Kubernetes支持多种容器运行时，最常见的是docker。
- pod，该组件是kubernetes集群中的最小单元，运行着应用容器。

前期学习对以上各个组件的职能有一个大概的了解即可。

## 安装minikube
创建一个较为完整可用的Kubernetes集群，需要三台物理机或虚拟机，一台作为master节点，另外两台为工作节点。这使得在本地调试很不方便，因此官方提供了minikube工具，可以在本地创建一个Kubernetes集群，minikube的安装过程[参考这里](https://minikube.sigs.k8s.io/docs/start/)。安装完之后执行`minikube start`可以启动一个Kubernetes集群。通过`minikube pause`或`minikube stop`命令可以停止该集群，minikube其他的可用命令可以[参考这里](https://minikube.sigs.k8s.io/docs/handbook/controls/)或执行`minikube help`查看帮助手册。

在本地安装Kubernetes集群除了minikube之外还有其他的工具，如kind、kubeadm。可以[参考这里](https://kubernetes.io/docs/tasks/tools/)学习其他工作的安装方式。
## 创建deployment
使用minikube创建好集群之后，我们本地创建一个nodejs应用，并将其构建成docker镜像。创建app.js:
```
const express = require('express');
const app = express();
const port = 4000;

const podname= process.env.HOSTNAME;
let requests = 0;
let startTime;

app.get('/',  (req, res) => {
  res.send(`Hello world, it's ${podname}(v1)\n`);
  console.log(`Running On: ${podname} | Total Requests: ${++requests} | App Uptime: ${(new Date() - startTime)/1000} seconds | Log Time: ${new Date()}`);
});

app.listen(port, () => {
  startTime = new Date();
  console.log(`Example app listening on port ${port} at ${startTime} | Running On: ${podname}\n`)
});
```
应该绑定的端口号为4000，所以在Dockerfile中我们将容器的4000端口暴露出来：
```
FROM node:16

WORKDIR /usr/src/app

COPY . .

RUN npm install

EXPOSE 4000

CMD [ "npm", "run", "start" ]
```
需要注意的是如果直接使用docker build命令构建镜像，构建好的镜像是在本机的docker仓库中的，而下面要实验的Kubernetes集群默认是不会从本地的docker中拉取镜像的。

我们目前使用的是本地的minikube集群，所以我们需要将构建的镜像导入到minikube集群中，这样在k8s集群中部署应用时，才能使用我们构建好的本地镜像。详细过程[参考这里](https://minikube.sigs.k8s.io/docs/handbook/pushing/)，该过程如下。先在当前终端执行：
```
eval $(minikube docker-env)
```
执行完之后，在当前终端执行的任何docker命令都相当于运行在minikube集群当中。然后我们运行docker build构建上面的应用：
```
docker build -t backend-express-demo:v1 .
```
构建成功之后我们就可以将当前镜像发布到k8s集群当中：
```
kubectl create deployment backend-express-demo --image=backend-express-demo:v1 --replicas=2
```
以上命令创建了一个新的deployment，名称为backend-express-demo，使用的镜像是我们刚刚构建好的backend-express-demo:v1镜像，另外指定当前应用的pod数量为2。执行`kubectl get pods`可以看到我们刚刚创建的两个pod。
```
NAME                                    READY   STATUS    RESTARTS   AGE
backend-express-demo-7bb844bff9-gj7j6   1/1     Running   0          16h
backend-express-demo-7bb844bff9-qlvjr   1/1     Running   0          16h
```
当前部署的两个pods在集群内部，外部还不能访问。我们先验证下部署的pod是否运行正常，调试的过程需要使用pod的名称，为了方便起见我们将pod的名称设置为环境变量，运行以下命令：
```
export POD_NAME="$(kubectl get pods -o go-template --template '{{(index .items 0).metadata.name}}{{"\n"}}')"
```
以上命令使用go-template将kubectl get pods的输出结果格式化，然后过滤并获取到第一个pod的名称，go-template的用法[参考这里](https://pkg.go.dev/text/template)。我们刚刚创建的pod处在k8s集群中私有的隔离网络环境中，当前外部的请求需要通过代理才能正常到达应用。新开一个终端执行`kubectl proxy`来开启代理，然后使用curl发送如下请求来查看应用是否正常响应：
```
curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME:8080/proxy/
```
还可以查看pod内的日志：
```
kubectl logs $POD_NAME
```
或者进入镜像内部查看：
```
kubectl exec -ti $POD_NAME -- bash
```
进入镜像内后运行curl命令，可以看到nodejs应用的响应：
```
curl http://localhost:4000
```
## 创建service
上面只是创建了deployment，想要外部请求能够访问刚刚部署的应用，还需要创建service。k8s中有两种类型的service:
- NodePort
- LoadBalancer

### NodePort
我们先创建NodePort类型的service。创建service之前先来查看当前service：
```
kubectl get services
# 或
kubectl get svc
```
输入如下，只有一个默认的service
```
❯ kubectl get services
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
backend-express-demo   NodePort    10.103.220.33   <none>        4000:32690/TCP   19h
```
然后我们通过kubectl expose命令为上面创建的deployment创建service
```
kubectl expose deployment/backend-express-demo --type="NodePort" --port 4000
```
上面创建的service的type为NodePort，端口号为4000，查看创建的service
```
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
backend-express-demo   NodePort    10.103.220.33   <none>        4000:32690/TCP   19h
kubernetes             ClusterIP   10.96.0.1       <none>        443/TCP          42h
```
这里需要指出的是，由于docker在非Linux系统的网络限制，我们不能直接访问minikube中的节点IP，需要使用以下命令创建一个通道(tunnel)来访问backend-express-demo。
```
minikube service backend-express-demo --url
```
输出的结果如下：
```
|-----------|----------------------|-------------|---------------------------|
| NAMESPACE |         NAME         | TARGET PORT |            URL            |
|-----------|----------------------|-------------|---------------------------|
| default   | backend-express-demo |        4000 | http://192.168.49.2:32690 |
|-----------|----------------------|-------------|---------------------------|
🏃  Starting tunnel for service backend-express-demo.
|-----------|----------------------|-------------|------------------------|
| NAMESPACE |         NAME         | TARGET PORT |          URL           |
|-----------|----------------------|-------------|------------------------|
| default   | backend-express-demo |             | http://127.0.0.1:61512 |
|-----------|----------------------|-------------|------------------------|
🎉  正通过默认浏览器打开服务 default/backend-express-demo...
❗  因为你正在使用 darwin 上的 Docker 驱动程序，所以需要打开终端才能运行它。
```
通过在本机浏览器访问`http://127.0.0.1:61512`或者执行`curl http://127.0.0.1:61512`就可以访问我们刚刚部署的nodejs应用。
### LoadBalancer
将service暴露到集群外部网络的标准方式是创建一个LoadBalancer类型的service，创建的方式和上面类似，不过创建service之前要先删除之前的创建的NodePort类型的service：
```
kubectl delete service backend-express-demo
```
由于本地使用的是minikube集群，**所以创建service之前需要在新的终端中运行`minikube tunnel`创建一个通道（tunnel）**，然后执行以下命令创建LoadBalancer类型的service：
```
kubectl expose deployment/backend-express-demo --type=LoadBalancer --port 4000
```
执行完之后运行`kubectl get svc`查看新的service，由于我们是在minikube集群中，且上面运行了`minikube tunnel`创建了一个通道，所以在EXTERNAL-IP列可以看到创建的service被分配了`127.0.0.1`，如果没有运行`minikube tunnel`，EXTERNAL-IP列将会一直显示"pending"。
```
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
backend-express-demo   LoadBalancer   10.103.171.149   127.0.0.1     4000:30504/TCP   2s
kubernetes             ClusterIP      10.96.0.1        <none>        443/TCP          90m
```

然后我们就可以使用EXTERNAL-IP来访问应用，如可以在浏览器中打开`http://127.0.0.1:4000/`。
> 有关minikube通道的相关知识，可以[参考这里](https://minikube.sigs.k8s.io/docs/handbook/accessing/)。
## 扩缩容
上面创建deployment时，指定的pods数量是2。k8s可以根据请求量的大小动态的扩容或者缩容。演示扩缩容之前我们现在新的终端中执行如下命令，动态的观察pod数量以及pod的创建和销毁的过程：
```
kubectl get pods -w
```
回到之前的终端执行以下命令可以扩容：
```
kubectl scale deployments/backend-express-demo --replicas=4
```
执行完之后查看pod数量的变化，可以看到类似如下的输出：
```
NAME                                    READY   STATUS              RESTARTS   AGE
backend-express-demo-7bb844bff9-9rmd9   1/1     Running             0          20m
backend-express-demo-7bb844bff9-njjgq   1/1     Running             0          20m
backend-express-demo-7bb844bff9-675rw   0/1     Pending             0          0s
backend-express-demo-7bb844bff9-675rw   0/1     Pending             0          0s
backend-express-demo-7bb844bff9-njscq   0/1     Pending             0          0s
backend-express-demo-7bb844bff9-675rw   0/1     ContainerCreating   0          0s
backend-express-demo-7bb844bff9-njscq   0/1     Pending             0          0s
backend-express-demo-7bb844bff9-njscq   0/1     ContainerCreating   0          0s
backend-express-demo-7bb844bff9-675rw   1/1     Running             0          1s
backend-express-demo-7bb844bff9-njscq   1/1     Running             0          1s
```
k8s新创建了两个新的pod，如果想缩容，我们直接将pod数量改小即可：
```
kubectl scale deployments/backend-express-demo --replicas=2
```
可以看到一段时间后pod被销毁。
## 更新和回滚应用
如果我们的应用修改了之后要重新发布，k8s会滚动式的更新应用，即逐步替换老的pod。如果我们有至少两个pod，就能做到发布时服务不中断。在演示应用更新之前，先来修改应用代码：
```
app.get('/',  (req, res) => {
  res.send(`Hello world, it's ${podname}(v2)\n`);
  console.log(`Running On: ${podname} | Total Requests: ${++requests} | App Uptime: ${(new Date() - startTime)/1000} seconds | Log Time: ${new Date()}`);
});
```
然后将修改后的代码打包成v2版本的镜像：
```
docker build -t backend-express-demo:v2 .
```
镜像构建成功之后，我们将上面创建的deployment更新为新的镜像：
```
kubectl set image deployments/backend-express-demo backend-express-demo=backend-express-demo:v2
```
可以观察到更新应用时先创建了新的v2版本镜像的pod，然后逐步删除老的pod，且由于我们有两个pods，所以服务不会中断。
如果已经发到线上的应用要回滚到上一个版本，命令如下：
```
kubectl rollout undo deployments/backend-express-demo
```
所有演示完毕后可以清理刚刚创建的deployment和service
```
kubectl delete deployments/backend-express-demo services/backend-express-demo
```
## Kubernetes Dashboard
k8s提供了dashboard来方便的管理k8s中的各种资源，由于我们本地使用了minikube，可以使用minikube的插件来开启此dashboard。先安装dashboard所必须的插件。
```
minikube addons enable metrics-server
```
插件安装成功后，开启dashboard。
```
minikube dashboard
```
几分钟之后，浏览器会自动打开dashboard，在dashboard中可以更直观的看到上面创建deplyment、service资源、扩缩容、更新和回滚的过程。
![post3.png](post3.png)
## 使用yaml配置文件
上面演示k8s的功能使用的是命令行的方式，k8s当然也只是使用配置文件的方式来管理应用。我们来演示如何使用配置文件。创建deployment.yaml：
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-express-demo
  labels: 
    app: backend-express-demo
    role: backend
    env: test
spec:
  selector:
    matchLabels:
      app: backend-express-demo
  replicas: 2    
  template:
    metadata:
      labels:
        app: backend-express-demo
        role: backend
        env: test
    spec:
      containers:
      - name: backend-express-demo
        image: backend-express-demo:v2
        ports:
        - containerPort: 4000
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
```
然后执行
```
kubectl apply -f deployment.yaml
```
创建deployment之后，为了正常的访问新创建的LoadBlancer类型的service，我们需要保持上面的`minikube tunnel`运行。创建service.yaml：
```
apiVersion: v1
kind: Service
metadata:
  name: backend-express-demo
  labels:
    app: backend-express-demo
    role: backend
    env: test
spec:
  type: LoadBalancer
  ports:
  - port: 4000
  selector:
    app: backend-express-demo
    role: backend
    env: test
```
然后执行
```
kubectl apply -f service.yaml
```
创建成功之后，可以在浏览器中访问`http://127.0.0.1:4000/`来查看应用的输出。
我们还可以将上面两个文件写在同一个配置文件中，仅靠一条命令就可以同时创建deployment和service，两部分配置通过`---`分隔。例如如下deployment-service.yaml文件：
```
...
      containers:
      - name: backend-express-demo
        image: backend-express-demo:v2
        ports:
        - containerPort: 4000
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: backend-express-demo
  labels:
    app: backend-express-demo
    role: backend
    env: test
...
```
然后执行如下命令：
```
kubectl apply -f depoyment-service.yaml
```

## 自动扩缩容（HPA）
上面演示手动扩缩容，k8s当然也支持自动扩缩容（HorizontalPodAutoscaler），实现自动扩缩容的方式也很简单。

在演示HPA功能之前，需要先在集群中安装并配置[Metrics Server](https://github.com/kubernetes-sigs/metrics-server#readme)，Metrics Server通过kubelets收集集群中各个节点的资源使用情况，然后调用[Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/)将数据上报给master节点，并通过[APIService](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/)创建新的资源满足h pa的设置要求。由于我们演示的环境是minikube，可以启用minikube的metrics-server插件简单的一键安装和配置Metrics Server，上文演示minikube中的k8s dashboard时已经启用过该插件，所以我们可以直接执行自动扩缩容。

先执行`kubectl get hpa`来查看当前没有创建自动扩缩容，然后执行：
```
kubectl autoscale deployment backend-express-demo --cpu-percent=5 --min=2 --max=10
```
该指令设置最小的pod数量为2，最大数量为10，为了演示将CPU的使用率阈值设置为5%，即当CPU使用率超过5%时会执行自动扩容，最终保证所有pod平均的CPU使用率为5%左右。再次执行`kubectl get hpa`查看hpa，输出的结果如下：
```
NAME                   REFERENCE                         TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
backend-express-demo   Deployment/backend-express-demo   0%/5%    2         10        2          80s
```
由于当前应用没有任何的请求，所以TARGETS列的CPU使用率还是0%，为了演示自动扩缩容的过程，我们模拟一些请求来增加CPU使用率。首先我们在新的窗口中执行如下命令，动态的观察hpa：
```
kubectl get hpa backend-express-demo --watch
```
然后我们使用[busybox](https://busybox.net/)镜像来模拟请求：
```
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://backend-express-demo:4000; done"
```
请求一段时间后，查看hpa资源的变化，可以看到CPU的使用率达到5%后pod数量（REPLICAS列）逐步增加。
```
NAME                   REFERENCE                         TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
backend-express-demo   Deployment/backend-express-demo   0%/5%     2         10        2          15s
backend-express-demo   Deployment/backend-express-demo   4%/5%     2         10        2          30s
backend-express-demo   Deployment/backend-express-demo   11%/5%    2         10        2          90s
backend-express-demo   Deployment/backend-express-demo   11%/5%    2         10        4          105s
backend-express-demo   Deployment/backend-express-demo   11%/5%    2         10        5          2m
backend-express-demo   Deployment/backend-express-demo   5%/5%     2         10        5          2m30s
```
除了通过命令行的方式创建HPA，还可以使用yaml文件的方式创建，上面的HPA改成yaml文件后如下：
```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: backend-express-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-express-demo
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 5
```
通过执行以下命令可以创建HPA：
```
kubectl create -f ./hpa.yaml
```
上面的例子中我们使用的指标称为资源指标（resource metrics）并且CPU的使用率为5%，还有可以使用绝对值和其他类型的资源指标。具体可以[参考这里](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)。
## 探针（Probes）
k8s使用探针机制来检查pod内应用健康状态，比如nodejs应用进程已经由于异常已经停止运行，此时需要一种机制来检查应用是否可用，当探针检查失败时，可以销毁该pod并创新创建一个新的pod来保持应用始终健康可用。k8s的探针主要分为liveness探针、startup探针和readiness探针。刚刚介绍的是liveness探针。

### liveness
先来演示如何使用liveness探针。首先在应用内创建一个新的路由，修改app.js：
```
// 添加http liveness探针
app.get('/healthz',  (req, res) => {
  res.status(200);
  console.log(`health check, ${podname} is ok`);
  res.send(`${podname} is ok`);
});

app.listen(port, () => {
  startTime = new Date();
  console.log(`Example app listening on port ${port} at ${startTime} | Running On: ${podname}\n`);
  // 10s后结束进程，模拟应用失败关闭
  setTimeout(() => {
    process.exit(1);
  }, 10000);
});
```
打包镜像
```
docker build -t backend-express-demo:v3 .
```
更新应用
```
kubectl set image deployments/backend-express-demo backend-express-demo=backend-express-demo:v3
```
然后修改depoyment-service.yaml文件如下：
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-express-demo
  labels: 
    app: backend-express-demo
    role: backend
    env: test
spec:
  selector:
    matchLabels:
      app: backend-express-demo
  replicas: 2    
  template:
    metadata:
      labels:
        app: backend-express-demo
        role: backend
        env: test
    spec:
      containers:
      - name: backend-express-demo
        image: backend-express-demo:v2
        ports:
        - containerPort: 4000
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
        livenessProbe:
          httpGet:
            path: /healthz
            port: 4000
            httpHeaders:
            - name: Custom-Header
              value: k8s-liveness-probes
          initialDelaySeconds: 3
          periodSeconds: 3
```
增加`livenessProbe`配置项。修改yaml文件之后可以重新应用该变更：
```
kubectl apply -f depoyment-service.yaml
```
然后可以进入k8s dashboard中查看events事件。k8s的liveness探针检测失败之后，pod被杀死，然后创建新的pod来启动应用。
![post4.png](post4.png)

上面的例子使用的http类型的探针，本质上就是向容器发送也定的http get请求，当请求的返回状态码>=200并<400时，检测结果成功，否则检测失败。k8s还有其他类型的探针如command、tcp、gRPC，具体使用方法[参考这里](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)。

### startup探针
有些应用启动和初始化过程较长，此时liveness探针检测可能会处于失败状态，如果k8s删除了pod并创建新的pod，应用又会重新启动。为了避免这种死锁的状态，k8s提供了startup探针。
```
...

livenessProbe:
  httpGet:
    path: /healthz
    port: 4000
    httpHeaders:
    - name: Custom-Header
      value: k8s-liveness-probes
    initialDelaySeconds: 3
    periodSeconds: 3        
startupProbe:
  httpGet:
    path: /healthz
    port: 4000
  failureThreshold: 30
  periodSeconds: 10  
  
...
```
以上配置指示应用会有10 * 30=300s的时间来完成启动，只有当startup探针的检测结果成功，liveness探针才会开始工作。如果检测失败，300秒之后容器会被杀死，并启动新的pod。

### readiness
有时候应用在运行过程中由于某种原因可能会临时处于不可用状态，如应用加载大的数据或者配置文件或者依赖其他外部的服务。这种情况下，可以使用readiness探针使应用保持运行状态，且不接受外部流量请求。readiness探针的使用方式和liveness完全相同。需要注意的是readiness运行在应用的整个生命周期，而不是启动时，这区别于startup探针。另外liveness探针不会等待readiness探针的执行结果，它只会等待startup探针的执行结果。有关探针的其他详细用法[参考这里](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)。

以上代码演示地址[点击这里](https://github.com/neilning-xc/kubernetes-demo)。

## 参考资料
- https://kubernetes.io/docs/home/
- https://minikube.sigs.k8s.io/docs/start/
- https://okigiveup.net/tutorials/a-tutorial-introduction-to-kubernetes/
