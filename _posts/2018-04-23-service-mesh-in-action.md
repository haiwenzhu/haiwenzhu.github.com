---
layout: post
title: "Servic Mesh实践"
categories:
  - Microservice
---

## 前言
随着云计算和云服务的不断发展和普及，越来越多的应用和服务被部署在了云端，部署方式的改变也反过来驱动了服务实现上的变化，这种变化的一个体现就是微服务在最近几年大行其道。
当然，任何一种技术都不是silver bullet，微服务也不是，虽然和monolithic application比起来，微服务在开发、部署上任然有其自身的优势，但同时也引入了分布式系统的复杂性，这些复杂性包括服务的发现、服务间通信、服务监控等。针对微服务带来的这些服务性问题，Buoyant公司最早提出了[Service Mesh](https://buoyant.io/2017/04/25/whats-a-service-mesh-and-why-do-i-need-one/)的概念，Service Mesh可以理解为是构建在服务下的一个基础设施层，其主要职责是负责服务间的通信，如果把服务类比于物理网络节点的话，Service Mesh可以类比为TCP协议。
下面这张图([图片来源](http://philcalcado.com/2017/08/03/pattern_service_mesh.html))中绿色模块代表微服务，蓝色模块代表Service Mesh，可以看到，每一个service都会同时部署一个Service Mesh的代理，服务只需要和它本地的代理通信，各个服务代理共同构建起了Service Mesh的网络。
![title](http://philcalcado.com/img/service-mesh/mesh1.png)
目前，已经有很多基于Serice Mesh概念的实现，包括Linkerd、Istio、Nginx Mesh、Envy等。本文会尝试基于Linkerd实现微服务的部署，希望通过具体的实例加深对Service Mesh的了解。

## 服务实现
首先，用go语言实现两个简单的服务：ping和pong。
ping服务源码：
```go
func main() {
    http.HandleFunc("/", func (w http.ResponseWriter, r *http.Request) {
        log.Println("Handle request")
        resp, err := http.Get("http://localhost:8002/pong")
        if err != nil {
            fmt.Fprintln(w, "I'm ping, talk to pong falied")
            return
        }
        defer resp.Body.Close()
        body, err := ioutil.ReadAll(resp.Body)
        if err != nil {
            fmt.Fprintln(w, "I'm ping, talk to pong with return empty")
        } else {
            fmt.Fprintf(w, "I'm ping, talk to pong with response: %s", body)
        }
    })
    http.HandleFunc("/ping", func (w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "ping")
    })
    http.ListenAndServe(":8001", nil)
}
```
pong服务和ping服务类似，提供http://localhost:8002/和http://localhost:8002/pong两个接口，源码就不贴出来了。

## Sidecar部署
上面的两个服务，相互间的通信都是直接通过ip和端口，而通常微服务都是通过服务发现获取到服务的最终地址的。
为了避免服务间的直接连接，在Service Mesh价格中，每个服务都会同时部署一个代理，服务和外部的通信都通过代理完成，这种模式也被称作[Sidecar pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/sidecar)。下面我们会分别为ping和pong分别部署各自的Sidecar。
为了部署方便，服务都采用docker部署，共享宿主机网络，部署命令：
> docker run --rm --name linkerd-ping --network="host" -v `pwd`/config.yaml:/config.yaml buoyantio/linkerd:1.3.7 /config.yaml

config.yaml文件内容如下：
```yaml
admin:
  port: 9990
  ip: 0.0.0.0
routers:
- protocol: http
  label: outging
  dtab: |
    /svc/pong => /$/inet/127.1/8002;
  servers:
  - port: 4001
    ip: 0.0.0.0
```
我们部署了一个sidecar服务，服务端口为4001，服务会将所有对pong的请求解析到127.1:8002上，为了让ping服务使用这个代理，我们需要对ping服务做两个调整，一是为服务设置一个http代理，让服务的对外请求都请求到部署的代理上，而是把服务的ip和端口换成服务的名字。
```
os.Setenv("http_proxy", "http://localhost:4001")
resp, err := http.Get("http://pong/pong")
```
pong的sidecar部署和ping类似，不再重复。
这里顺便简单介绍一下Linkerd的route过程，详细的文档请看[这里](https://linkerd.io/advanced/routing/)。下面这张图是Linkerd的routing的过程：
![title](https://linkerd.io/images/routing1.png)
从请求到解析到最终的服务会经理3个阶段：
1. Identification，Identification阶段用于将请求转换成名字，Linkerd支持多种[Identifiers](https://linkerd.io/config/head/linkerd/index.html#http-1-1-identifiers)，默认使用http的host header，如上面的例子，`http://pong/pong`会转换成/svc/pong。
2. Bidding，bindding阶段是根据我们配置的[dtab](https://linkerd.io/advanced/dtabs/)路由规则，对名字做转换，在上面的例子中，Bidding后会得到`/$/inet/127.1/8002`。
3. Resolution，第二步的bidding或将名字转换成以`/$`或`/#`开头的client name，再通过resolution讲client name转换成服务的ip和端口，在上面的例子中，我们最终拿到的服务ip和端口是127.0.0.1:8002。

## Sidecar服务编排
上面我们的服务和服务的sidecar是分开部署的，但sidecar是伴随每一个服务而存在的，可以使用docker compose把两个服务编排在一起。
首先，需要把我们的ping和pong两个服务docker化，Dockerfile内容如下：
```
FROM golang:1.10

WORKDIR /go/src/app
COPY ./main.go .

CMD ["go","run","main.go"]
```
执行下面的命令build镜像:

> docker build -t my/ping .

然后，生成docker-composer.yaml配置文件，内容如下：
```yaml
version: "3.2"
services:
  sidecar:
    image: buoyantio/linkerd:1.3.7
    container_name: my-ping-sidecar
    network_mode: "host"
    volumes:
      - type: bind
        source: ./config.yaml
        target: /config.yaml
    command: ["/config.yaml"]
  serv:
    image: bugzhu/ping
    container_name: my-ping
    network_mode: "host"
    volumes:
      - type: bind
        source: .
        target: /go/src/app
```
执行`docker-compose up`启动服务即可。

## Service Discovery服务部署
到目前位置，我们已经完成了sidecar的部署，实现了服务的解析。但是从上面的配置文件可以看到，服务的解析是直接写在配置文件里的，这显然无法满足微服务动态服务部署、服务上下线、负载均衡等要求。因此，我们需要引入专门的服务用于服务的注册和发现。在Linkerd中，[namers](https://linkerd.io/config/head/linkerd/index.html#namers-and-service-discovery)配置用于服务发现相关服务的配置，目前Linkerd支持的service discovery服务包括本地文件、zookeeper、consul，k8s等。我们选择zookeeper作为我们的service discovery服务。
首先，部署zookeeper服务：
> docker run --name my-zookeeper --network="host" -d zookeeper:3.3.6

修改Linkerd配置增加namer配置：
```yaml
namers:
- kind: io.l5d.serversets
  zkAddrs:
  - host: localhost
    port: 2181
routers:
- protocol: http
  label: outging
  dtab: |
    /svc => /#/io.l5d.serversets/discovery;
  servers:
  - port: 4001
    ip: 0.0.0.0
```
当请求http://pong/pong时，Linkerd会根据我们配置的route规则最终解析到`/#/io.l5d.serversets/discovery/pong`，然后通过namers配置的Service Discovery服务解析到最终的ip和端口。
顺便说一下，使用serversets时，保存在zookeeper中的信息需要符合serversets格式，比如上面的离职，最终的服务节点都保存在zookeeper的/discovery/pong/member_xxxx路径下，如`/discovery/pong/member_0000000000`、`/discovery/pong/member_0000000002`等，每一个路径代表一个服务节点，每个节点保存的信息如下：
```json
{"status": "ALIVE", "additionalEndpoints": {}, "serviceEndpoint": {"host": "127.0.0.1", "port": 8002}}
```
## Service Mesh部署
到目前位置，我们已经实现的Serivce Discovery服务的配置，可以通过Serice Discovery服务实现服务的动态部署。但是有一个问题，在文章开头的那副关于Service Mesh的示意图里，Server Mesh网络是通过sidecar服务互相通信的，服务本身只和sidecar交互。但是在我们的实例里，ping服务请求pong服务最终解析到的是pong的真实ip和地址，整个过程为ping请求ping's sidecar，ping's sidecar请求pong，和pong的sidecar没有关系。预期的架构应该是像下图的左边部分，但现状是下图的右边部分。
![title](https://photos.app.goo.gl/MEF3Kz3dSdmXvTuC3)

为了实现左边的架构，我们不仅需要让服务发出的请求走sidecar，同时也要让服务接收的请求走sidecar。事实上，Linkerd也是支持这种[部署方式](https://linkerd.io/advanced/deployment/)的。
修改pong的Linkerd的配置如下：
```yaml
namers:
- kind: io.l5d.serversets
  zkAddrs:
  - host: localhost
    port: 2181
routers:
- protocol: http
  label: outging
  dtab: |
    /svc => /#/io.l5d.serversets/discovery;
  servers:
  - port: 4003
    ip: 0.0.0.0
- protocol: http
  label: incoming
  dtab: |
    /svc/* => /$/inet/127.1/8002;
  servers:
  - port: 4004
    ip: 0.0.0.0
```
pong的sidecar开启了4003和4004两个端口，其中4003作为服务的代理，4004作为服务的反向代理，之前在service discovery中注册的pong的服务地址为127.0.0.1:8002，8002端口需要改成4004，4004再将请求转发给8002。

## Namerd服务部署
到目前位置，我们已经完成了一个简单的基于Service Mesh的微服务部署，但还是有很多问题，首先是我们的route规则都是写在Linkerd配置里的，其次是每个微服务都直连了service discovery服务，而这两部分在各个微服务之前是可以共享的。
Linkerd提供了Namerd服务，Namerd为我们提供了route规则的管理，同时也支持serverice discovery服务，实现了这两部分的集中管理。
首先，启动Namerd服务：
> docker run --rm --name namerd --network="host" -v `pwd`/config.yaml:/config.yaml buoyantio/namerd:1.3.7 /config.yaml
config.yaml内容如下：
```yaml
admin:
  port: 9991
storage:
  kind: io.l5d.zk
  zkAddrs:
  - host: 127.0.0.1
    port: 2181
  sessionTimeoutMs: 100000000
namers:
- kind: io.l5d.serversets
  zkAddrs:
  - host: 127.0.0.1
    port: 2181
interfaces:
- kind: io.l5d.thriftNameInterpreter
  port: 4100
  ip: 0.0.0.0
- kind: io.l5d.httpController
  port: 4180
  ip: 0.0.0.0
```
集成Namerd之后的Linkerd配置如下：
```yaml
admin:
  port: 9990
  ip: 0.0.0.0
routers:
- protocol: http
  label: outging
  interpreter:
    kind: io.l5d.namerd
    dst: /$/inet/localhost/4100
    namespace: default
    transformers:
    - kind: io.l5d.port
      port: 4004
  servers:
  - port: 4001
    ip: 0.0.0.0
- protocol: http
  label: incoming
  dtab: |
    /svc/* => /$/inet/127.1/8001;
  servers:
  - port: 4002
    ip: 0.0.0.0
```
上面的配置里，iterpreter配置了Namerd服务的地址，其中有一个transformers的配置需要特殊说明一下，还记得上一部分我们为了实现服务接收的请求也走sidecar，把微服务的端口注册成了对应的sidecar的代理端口。transformers可以实现在发送请求之前，把端口替换成sidecar的代理端口，这样我们注册在service discovery里的服务端口不在需要替换成sidecar的端口了。

## 后记
基于Linker，我们实现了一个简单的Service Mesh架构的微服务部署，当然，还有很多微服务基础的功能缺失，包括服务的注册，服务的监控等。Linkerd支持服务相关数据收集和上报，但是，在Service Register这块功能目前是缺失的。
其实，在Service Mesh的实现上，[Istio](https://istio.io/)的功能更丰富一点，由于Istio还在快速的迭代中，部署起来也相对复杂，应此选择了Linkerd来进行Service Mesh的尝试。也推荐感兴趣的同学可以关注一下Istio。
为了部署方便，本文的所有基于docker部署的服务都使用了host网络模式，但在实际部署中很少采用这种模式。

## 参考连接
1. [Pattern: Service Mesh](http://philcalcado.com/2017/08/03/pattern_service_mesh.html)
2. [What’s a service mesh? And why do I need one?](https://buoyant.io/2017/04/25/whats-a-service-mesh-and-why-do-i-need-one/)
3. [Service Mesh：下一代微服务](http://dockone.io/article/2801)
4. [Linkerd](https://linkerd.io/docs/index.html)
5. [Istio](https://istio.io/docs/)
