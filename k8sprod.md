# Kubernetes 生产环境部署模式

……和反模式。

我们将探索有帮助的技术，以提高 Kubernetes 部署的弹性和高可用性，
还会看看使用 Docker 和 Kubernetes 时要避免的一些常见错误。

## 安装

首先，按照 [安装说明](README.md#installation) 进行安装

### 反模式：混合了构建环境和运行时环境

来看这个 dockerfile

```Dockerfile
FROM ubuntu:14.04

RUN apt-get update
RUN apt-get install gcc
RUN gcc hello.c -o /hello
```

它编译并运行一个简单的 helloworld 程序：

```bash
$ cd prod/build
$ docker build -t prod .
$ docker run prod
Hello World
```

最终的 Dockerfile 有几个问题：

**大小**

```bash
$ docker images | grep prod
prod                                          latest              b2c197180350        14 minutes ago      293.7 MB
```

几乎是用 300MB 的镜像来部署几 KB 的 c 程序！
引入了包管理器、C 编译器和运行此程序不需要的大量的工具。


这为我们带来第二个问题：

**安全**

我们分发了整个构建工具链。除此之外，还发布了镜像的源代码：

```bash
$ docker run --entrypoint=cat prod /build/hello.c
#include<stdio.h>

int main()
{
    printf("Hello World\n");
    return 0;
}
```

**拆分构建环境和运行环境**

我们使用 “buildbox” 模式来构建包含构建环境的映像，
并且用更小的运行环境来运行程序


```bash
$ cd prod/build-fix
$ docker build -f build.dockerfile -t buildbox .
```

**注意：** 使用 `-f` 标志来指定我们要用的 dockerfile。

现在我们有一个包含构建环境的 “buildbox” 镜像。
可以用它来编译 C 程序：

```bash
$ docker run -v $(pwd):/build  buildbox gcc /build/hello.c -o /build/hello
```

这次没有使用 `docker build` 命令，而是直接挂载源代码并运行编译器。

**注意：** Docker 将很快支持这种模式，通过在构建过程中引入 [构建阶段（build stages）](https://github.com/docker/docker/pull/32063)。

**更新:** [现在 CE（Docker CE Edge）中提供了多阶段构建](https://docs.docker.com/engine/userguide/eng-image/multistage-build/).

现在可以使用更简单（也更小）的 dockerfile 来运行我们的映像：

```Dockerfile
FROM quay.io/gravitational/debian-tall:0.0.1

ADD hello /hello
ENTRYPOINT ["/hello"]
```

```bash
$ docker build -f run.dockerfile -t prod:v2 .
$ docker run prod:v2
Hello World
$ docker images | grep prod
prod                                          v2                  ef93cea87a7c        17 seconds ago       11.05 MB
prod                                          latest              b2c197180350        45 minutes ago       293.7 MB
```

**注意：** 请注意，您应该计划好或者在映像中提供所需的“共享库”，或者在“静态构建”的二进制文件中其包含所需的库。

### 反模式: 僵尸进程和孤儿进程

**注意：** 此示例演示只适用于 Linux

**孤儿进程**

很容易留下孤立的进程在后台运行。来看看前一个例子中建立的镜像：

```bash
docker run busybox sleep 10000
```

现在，打开一个单独的终端并找到该进程

```bash
ps uax | grep sleep
sasha    14171  0.0  0.0 139736 17744 pts/18   Sl+  13:25   0:00 docker run busybox sleep 10000
root     14221  0.1  0.0   1188     4 ?        Ss   13:25   0:00 sleep 10000
```

可见，实际上有两个进程：`docker run` 和在容器中运行的 `sleep 1000`。

我们把 kill 信号发送到 `docker run` 进程（就像 CI / CD 任务对长时间运行的进程一样）：

```bash
kill 14171
```

`docker run` 进程没有退出，`sleep` 进程仍在运行！

```bash
ps uax | grep sleep
root     14221  0.0  0.0   1188     4 ?        Ss   13:25   0:00 sleep 10000
```


Yelp的工程师对为什么出现这种情况有一个很好的答案，看 [这里](https://github.com/Yelp/dumb-init):

> Linux 内核对以 PID 1 运行的进程作了特殊的信号处理。
> 当普通的 Linux 系统上进程接收到一个信号时，内核将首先检查该进程所注册的自定义信号处理程序，否则会执行默认行为（例如，SIGTERM 时杀死进程）。

> 然而，如果接收信号的过程是PID 1，它得到内核的特殊对待；如果没有注册信号处理程序，内核不会执行默认行为，而是不做任何操作。换一种说法，如果进程没有明确处理这些信号，发送 SIGTERM 不会有任何效果。

为了解决这个问题（还有其它问题），需要一个简单的 init 系统，它具有适当的信号处理程序。
幸运的是，`Yelp` 的工程师们构建了简单而轻量级的 init 系统，`dumb-init`

```bash
docker run quay.io/gravitational/debian-tall /usr/bin/dumb-init /bin/sh -c "sleep 10000"
```

现在，您可以简单的使用 SIGTERM 停止 `docker run` 进程，它会妥善处理关闭

### 反模式： 直接使用 Pods

[Kubernetes Pod](https://kubernetes.io/docs/user-guide/pods/#what-is-a-pod) 是一个构建块，本身不持久。

不要直接在生产中使用 Pods。它们不会被重新调度，保留其数据或保证任何持久性。

取而代之的是，可以使用 `Deployment` 与复制因子 1，这将保证 pods 被重新调度，并将在被逐出或节点丢失后得以幸免。


### 反模式： 使用后台进程

```bash
$ cd prod/background
$ docker build -t $(minikube ip):5000/background:0.0.1 .
$ docker push $(minikube ip):5000/background:0.0.1
$ kubectl create -f crash.yaml
$ kubectl get pods
NAME      READY     STATUS    RESTARTS   AGE
crash     1/1       Running   0          5s
```

容器似乎正在运行，但是让我们检查一下我们的服务器进程是否在运行：

```bash
$ kubectl exec -ti crash /bin/bash
root@crash:/# 
root@crash:/# 
root@crash:/# ps uax
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0  21748  1596 ?        Ss   00:17   0:00 /bin/bash /start.sh
root         6  0.0  0.0   5916   612 ?        S    00:17   0:00 sleep 100000
root         7  0.0  0.0  21924  2044 ?        Ss   00:18   0:00 /bin/bash
root        11  0.0  0.0  19180  1296 ?        R+   00:18   0:00 ps uax
root@crash:/# 
```

**使用探针（Probes）**

我们犯了错误，HTTP服务器没有运行，但没有特征表明这一点，因为父进程仍在运行。

第一个直接的修复方法是使用正确的 init 系统并监视 web 服务的状态。
但是，让我们以此开始引入使用存活探测器：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fix
  namespace: default
spec:
  containers:
  - command: ['/start.sh']
    image: localhost:5000/background:0.0.1
    name: server
    imagePullPolicy: Always
    livenessProbe:
      httpGet:
        path: /
        port: 5000
      timeoutSeconds: 1
```

```bash
$ kubectl create -f fix.yaml
```

存活性探测器的探测结果为失败，容器将重新启动。

```bash
$ kubectl get pods
NAME      READY     STATUS    RESTARTS   AGE
crash     1/1       Running   0          11m
fix       1/1       Running   1          1m
```

### 生产模式： 日志

设置日志输出到标准输出：

```bash
$ kubectl create -f logs/logs.yaml
$ kubectl logs logs
hello, world!
```

Kubernetes 和 Docker 都有插件系统，以确保发送到 stdout 和 stderr 的日志被收集，转发和轮转。

**注意：** 这是 [The Twelve Factor App](https://12factor.net/logs) 的模式之一，Kubernetes 内置支持！

### 生产模式： 不可变容器

每当你对容器的文件系统写数据，它会采用 [写时拷贝策略](https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/#container-and-layers)。

新的存储层被存储驱动程序（devicemapper，overlayfs 或其它）创建。
如果使用很活跃，它会为存储驱动程序带来很大的负载，特别是在使用 Devicemapper 或 BTRFS 的情况下。

确保您的容器仅将重要数据写入卷。可以使用 `tmpfs` 保存小的（因为 tmpfs 的所有内容存储在内存中）临时文件：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: busybox
    name: test-container
    volumeMounts:
    - mountPath: /tmp
      name: tempdir
  volumes:
  - name: tempdir
    emptyDir: {}
```

### 反模式： 使用 `latest` 标签（tag）

在生产中不要使用 `latest` 标签。 它造成歧义，因为搞不清楚真正的版本的应用程序是什么。

可以为开发目的而使用 `latest` 标签，确保将 `imagePullPolicy` 设置为 `Always`，确保 Kubernetes 在创建 pod 时总是拉取最新版本：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: always
  namespace: default
spec:
  containers:
  - command: ['/bin/sh', '-c', "echo hello, world!"]
    image: busybox:latest
    name: server
    imagePullPolicy: Always
```

### 生产模式： Pod 就绪状态

想象一种情况，容器需要较长时间才能启动。为模拟这个情况，写一个简单的脚本：

```bash
#!/bin/bash

echo "Starting up"
sleep 30
echo "Started up successfully"
python -m http.serve 5000
```

推送镜像、启动服务和部署：

```yaml
$ cd prod/delay
$ docker build -t $(minikube ip):5000/delay:0.0.1 .
$ docker push $(minikube ip):5000/delay:0.0.1
$ kubectl create -f service.yaml
$ kubectl create -f deployment.yaml
```

进入集群内的 curl 容器，确保一切工作正常：

```
kubectl run -i -t --rm cli --image=tutum/curl --restart=Never
curl http://delay:5000
<!DOCTYPE html>
...
```

当在前 30 秒内尝试访问它时，会出现 `拒绝连接错误`。

更新模拟部署环境：

```bash
$ docker build -t $(minikube ip):5000/delay:0.0.2 .
$ docker push $(minikube ip):5000/delay:0.0.2
$ kubectl replace -f deployment-update.yaml
```

在下一个窗口中，试试看是否有任何服务中断时间：

```bash
curl http://delay:5000
curl: (7) Failed to connect to delay port 5000: Connection refused
```

我们发生了一次服务停机，尽管已经在滚动更新策略里设置了 `maxUnavailable：0`！
发生这种情况是因为 Kubernetes 不知道启动延时和服务的准备就绪状态。

我们通过使用就绪探测来解决这个问题：

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 5000
  timeoutSeconds: 1
  periodSeconds: 5
```

准备探测器指示 pod 容器的准备就绪状态，Kubernetes 在进行部署时会参考这一项：

```bash
$ kubectl replace -f deployment-fix.yaml
```

这次我们没有服务停机。

### 反模式：解绑迅速失败的任务

Kubernetes 提供有用的新工具来安排容器执行一次性任务：[jobs](https://kubernetes.io/docs/concepts/jobs/run-to-completion-finite-workloads/)

但是，有一个问题：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: bad
spec:
  template:
    metadata:
      name: bad
    spec:
      restartPolicy: Never
      containers:
      - name: box
        image: busybox
        command: ["/bin/sh", "-c", "exit 1"]
```


```bash
$ cd prod/jobs
$ kubectl create -f job.yaml
```

你会看到任务无休止的重试，创建了成百上千的容器：

```bash
$ kubectl describe jobs 
Name:		bad
Namespace:	default
Image(s):	busybox
Selector:	controller-uid=18a6678e-11d1-11e7-8169-525400c83acf
Parallelism:	1
Completions:	1
Start Time:	Sat, 25 Mar 2017 20:05:41 -0700
Labels:		controller-uid=18a6678e-11d1-11e7-8169-525400c83acf
		job-name=bad
Pods Statuses:	1 Running / 0 Succeeded / 24 Failed
No volumes.
Events:
  FirstSeen	LastSeen	Count	From			SubObjectPath	Type		Reason			Message
  ---------	--------	-----	----			-------------	--------	------			-------
  1m		1m		1	{job-controller }			Normal		SuccessfulCreate	Created pod: bad-fws8g
  1m		1m		1	{job-controller }			Normal		SuccessfulCreate	Created pod: bad-321pk
  1m		1m		1	{job-controller }			Normal		SuccessfulCreate	Created pod: bad-2pxq1
  1m		1m		1	{job-controller }			Normal		SuccessfulCreate	Created pod: bad-kl2tj
  1m		1m		1	{job-controller }			Normal		SuccessfulCreate	Created pod: bad-wfw8q
  1m		1m		1	{job-controller }			Normal		SuccessfulCreate	Created pod: bad-lz0hq
  1m		1m		1	{job-controller }			Normal		SuccessfulCreate	Created pod: bad-0dck0
  1m		1m		1	{job-controller }			Normal		SuccessfulCreate	Created pod: bad-0lm8k
  1m		1m		1	{job-controller }			Normal		SuccessfulCreate	Created pod: bad-q6ctf
  1m		1s		16	{job-controller }			Normal		SuccessfulCreate	(events with common reason combined)

```

这可不是预期的结果。随着时间的推移，节点和 docker 的负载将相当大，特别是如果作业非常快的失败。

我们先来清理快速失败的工作：

```bash
$ kubectl delete jobs/bad
```

现在让我们使用 `activeDeadlineSeconds` 来限制重试次数：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: bound
spec:
  activeDeadlineSeconds: 10
  template:
    metadata:
      name: bound
    spec:
      restartPolicy: Never
      containers:
      - name: box
        image: busybox
        command: ["/bin/sh", "-c", "exit 1"]
```

```bash
$ kubectl create -f bound.yaml
```

现在你会看到，10秒钟后，这个工作失败了：

```bash
  11s		11s		1	{job-controller }			Normal		DeadlineExceeded	Job was active longer than specified deadline
```


**注意：** 有时，永远重试是有道理的。在这种情况下，请确保设置恰当的 pod 重启策略，以防止意外的集群内 DDOS。


### 生产模式： 断路器

这个例子中，Web 应用是一个虚构的电子邮件服务器。
要渲染页面，我们的前端必须向后端发起两个请求：

* 与天气服务通话以获得当前天气
* 从数据库获取当前邮件

如果天气服务宕机，用户仍然会想查看电子邮件，所以天气服务是边缘服务，而邮件服务是关键服务。

这是用python编写的前端、天气和邮件服务：

**天气**

```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return '''Pleasanton, CA
Saturday 8:00 PM
Partly Cloudy
12 C
Precipitation: 9%
Humidity: 74%
Wind: 14 km/h
'''

if __name__ == "__main__":
    app.run(host='0.0.0.0')
```

**邮件**

```python
from flask import Flask,jsonify
app = Flask(__name__)

@app.route("/")
def hello():
    return jsonify([
        {"from": "<bob@example.com>", "subject": "lunch at noon tomorrow"},
        {"from": "<alice@example.com>", "subject": "compiler docs"}])

if __name__ == "__main__":
    app.run(host='0.0.0.0')
```

**前端**

```python
from flask import Flask
import requests
from datetime import datetime
app = Flask(__name__)

@app.route("/")
def hello():
    weather = "weather unavailable"
    try:
        print "requesting weather..."
        start = datetime.now()
        r = requests.get('http://weather')
        print "got weather in %s ..." % (datetime.now() - start)
        if r.status_code == requests.codes.ok:
            weather = r.text
    except:
        print "weather unavailable"

    print "requesting mail..."
    r = requests.get('http://mail')
    mail = r.json()
    print "got mail in %s ..." % (datetime.now() - start)

    out = []
    for letter in mail:
        out.append("<li>From: %s Subject: %s</li>" % (letter['from'], letter['subject']))
    

    return '''<html>
<body>
  <h3>Weather</h3>
  <p>%s</p>
  <h3>Email</h3>
  <p>
    <ul>
      %s
    </ul>
  </p>
</body>
''' % (weather, '<br/>'.join(out))

if __name__ == "__main__":
    app.run(host='0.0.0.0')
```

我们来创建部署和服务：

```bash
$ cd prod/cbreaker
$ docker build -t $(minikube ip):5000/mail:0.0.1 .
$ docker push $(minikube ip):5000/mail:0.0.1
$ kubectl apply -f service.yaml
deployment "frontend" configured
deployment "weather" configured
deployment "mail" configured
service "frontend" configured
service "mail" configured
service "weather" configured
```

检查一切运行顺利：

```bash
$ kubectl run -i -t --rm cli --image=tutum/curl --restart=Never
$ curl http://frontend
<html>
<body>
  <h3>Weather</h3>
  <p>Pleasanton, CA
Saturday 8:00 PM
Partly Cloudy
12 C
Precipitation: 9%
Humidity: 74%
Wind: 14 km/h
</p>
  <h3>Email</h3>
  <p>
    <ul>
      <li>From: <bob@example.com> Subject: lunch at noon tomorrow</li><br/><li>From: <alice@example.com> Subject: compiler docs</li>
    </ul>
  </p>
</body>
```

引入会崩溃的气象服务：

```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    raise Exception("I am out of service")

if __name__ == "__main__":
    app.run(host='0.0.0.0')
```

构建并重新部署：

```bash
$ docker build -t $(minikube ip):5000/weather-crash:0.0.1 -f weather-crash.dockerfile .
$ docker push $(minikube ip):5000/weather-crash:0.0.1
$ kubectl apply -f weather-crash.yaml 
deployment "weather" configured
```

确保它崩溃：

```bash
$ kubectl run -i -t --rm cli --image=tutum/curl --restart=Never
$ curl http://weather
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>500 Internal Server Error</title>
<h1>Internal Server Error</h1>
<p>The server encountered an internal error and was unable to complete your request.  Either the server is overloaded or there is an error in the application.</p>
```

而我们的前端应该是正常的：

```bash
$ kubectl run -i -t --rm cli --image=tutum/curl --restart=Never
curl http://frontend
<html>
<body>
  <h3>Weather</h3>
  <p>weather unavailable</p>
  <h3>Email</h3>
  <p>
    <ul>
      <li>From: <bob@example.com> Subject: lunch at noon tomorrow</li><br/><li>From: <alice@example.com> Subject: compiler docs</li>
    </ul>
  </p>
</body>
root@cli:/# curl http://frontend                    
<html>
<body>
  <h3>Weather</h3>
  <p>weather unavailable</p>
  <h3>Email</h3>
  <p>
    <ul>
      <li>From: <bob@example.com> Subject: lunch at noon tomorrow</li><br/><li>From: <alice@example.com> Subject: compiler docs</li>
    </ul>
  </p>
</body>
```

一切都按预期工作！
仍有一种问题，我们刚刚观察了服务迅速崩溃的情况，让我们再看看如果天气服务很慢，会发生什么。
这在生产中更经常地发生，例如，由于网络过载或数据库过载。

为了模拟这种失败，我们引入人为的延迟：

```python
from flask import Flask
import time

app = Flask(__name__)

@app.route("/")
def hello():
    time.sleep(30)
    raise Exception("System overloaded")

if __name__ == "__main__":
    app.run(host='0.0.0.0')
```

构建并重新部署：

```bash
$ docker build -t $(minikube ip):5000/weather-crash-slow:0.0.1 -f weather-crash-slow.dockerfile .
$ docker push $(minikube ip):5000/weather-crash-slow:0.0.1
$ kubectl apply -f weather-crash-slow.yaml 
deployment "weather" configured
```

正如预期，天气服务超时了：

```bash
curl http://weather 
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>500 Internal Server Error</title>
<h1>Internal Server Error</h1>
<p>The server encountered an internal error and was unable to complete your request.  Either the server is overloaded or there is an error in the application.</p>
```

问题在于，每个前端的请求也需要 10 秒钟

```bash
curl http://frontend
```

这是一种更常见的服务中断：用户沮丧的离开，因为服务不可用。
为解决这个问题，我们将介绍一种包含[断路器（circuit breaker）](http://vulcand.github.io/proxy.html#circuit-breakers)的特殊代理。

![standby](http://vulcand.github.io/_images/CircuitStandby.png)

断路器是一种特殊的中间件，旨在在服务降级的情况下提供故障转移操作。
有助于防止级联故障：一个服务的故障导致另一个服务的故障。
断路器监控请求统计信息，并检查特定错误的统计信息。


![tripped](http://vulcand.github.io/_images/CircuitTripped.png)

这是用 python 写的简单断路器：

```python
from flask import Flask
import requests
from datetime import datetime, timedelta
from threading import Lock
import logging, sys


app = Flask(__name__)

circuit_tripped_until = datetime.now()
mutex = Lock()

def trip():
    global circuit_tripped_until
    mutex.acquire()
    try:
        circuit_tripped_until = datetime.now() + timedelta(0,30)
        app.logger.info("circuit tripped until %s" %(circuit_tripped_until))
    finally:
        mutex.release()

def is_tripped():
    global circuit_tripped_until    
    mutex.acquire()
    try:
        return datetime.now() < circuit_tripped_until
    finally:
        mutex.release()
    

@app.route("/")
def hello():
    weather = "weather unavailable"
    try:
        if is_tripped():
            return "circuit breaker: service unavailable (tripped)"

        r = requests.get('http://localhost:5000', timeout=1)
        app.logger.info("requesting weather...")
        start = datetime.now()
        app.logger.info("got weather in %s ..." % (datetime.now() - start))
        if r.status_code == requests.codes.ok:
            return r.text
        else:
            trip()
            return "circuit breaker: service unavailable (tripping 1)"
    except:
        app.logger.info("exception: %s", sys.exc_info()[0])
        trip()
        return "circuit breaker: service unavailable (tripping 2)"

if __name__ == "__main__":
    app.logger.addHandler(logging.StreamHandler(sys.stdout))
    app.logger.setLevel(logging.DEBUG)
    app.run(host='0.0.0.0', port=6000)
```

我们来构建和重新部署断路器：

```bash
$ docker build -t $(minikube ip):5000/cbreaker:0.0.1 -f cbreaker.dockerfile .
$ docker push $(minikube ip):5000/cbreaker:0.0.1
$ kubectl apply -f weather-cbreaker.yaml 
deployment "weather" configured
$  kubectl apply -f weather-service.yaml
service "weather" configured
```


断路器将检测到服务中断，边缘的天气服务不会使邮件服务失效：

```bash
curl http://frontend
<html>
<body>
  <h3>Weather</h3>
  <p>circuit breaker: service unavailable (tripped)</p>
  <h3>Email</h3>
  <p>
    <ul>
      <li>From: <bob@example.com> Subject: lunch at noon tomorrow</li><br/><li>From: <alice@example.com> Subject: compiler docs</li>
    </ul>
  </p>
</body>
```

**注意：** 有一些生产级别的代理器，原生支持断路器模式——如 [Vulcand](http://vulcand.github.io/)、[Nginx plus](https://www.nginx.com/products/) 或 [Envoy](https://lyft.github.io/envoy/)。


### 生产模式： 用于速率和连接控制的跨斗（Sidecar）

上一个例子中，我们使用了一种跨斗模式：代理请求到本地的特殊 Pod，为服务添加了额外的逻辑，如错误检测，TLS 终止等功能。

下面是一个为 Nginx 代理添加速率和连接限制的跨斗的例子：

```bash
$ cd prod/sidecar
$ docker build -t $(minikube ip):5000/sidecar:0.0.1 -f sidecar.dockerfile .
$ docker push $(minikube ip):5000/sidecar:0.0.1
$ docker build -t $(minikube ip):5000/service:0.0.1 -f service.dockerfile .
$ docker push $(minikube ip):5000/service:0.0.1
$ kubectl apply -f sidecar.yaml
deployment "sidecar" configured
```

尝试更快地访问服务（每秒多于一个请求），您将看到速率限制起作用

```bash
$ kubectl run -i -t --rm cli --image=tutum/curl --restart=Never
curl http://sidecar
```


[Istio](https://istio.io/docs/concepts/what-is-istio/overview.html#architecture) 是体现这种设计的平台的例子。
