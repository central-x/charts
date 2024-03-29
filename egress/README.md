# Kubernetes Egress
## 概述
&emsp;&emsp;在生产环境中，为了保证应用在一个安全的环境下运行，减少数据泄露、服务器被攻击的风险，通常会通过网络策略将生产服务器集群与外部服务器进行网络隔离，只开放指定的服务器与外部服务器进行通讯。网络拓扑图如下：

![](./assets/topology.svg)

&emsp;&emsp;在上面这种网络拓扑结构下，我们可以通过将 Pod 调度到 `Node 5` 节点下，就可以让需要与外部通信通信的服务正常工作了。但是这样设计的话，所有需要和外部通信的服务都得要部署到这个节点上了，极容易造成资源不平衡。

&emsp;&emsp;为了解决这个问题，可以在 `Node 5` 这个节点上部署一个 Nginx 进行外部服务的反向代理，这样内部的服务就可以不用集中调度到 `Node 5` 下了。

&emsp;&emsp;基于以上理论，为了方便将外部业务系统快速封装成内部服务（Service），因此封装了 Egress。Egress 可以根据用户配置的路由信息，自动生成对应代理信息和 Kubernetes Service，后续集群内部的服务就可以通过 `http://<service-name>` 的方式访问外部网络。

## 容器部署规范
&emsp;&emsp;本部署包遵循以下部署规范：

- 分散部署：当应用支持 `HPA` 时，多个实例会尽量分散部署到不同的节点上，以保证应用的可用性。
- 隔离部署：应用只调度与对应命名空间的节点（`node.kubernetes.io/namespace=<namespace>`）上，保证各环境（生产环境、预发布环境等）节点隔离。
- 支持探针：支持启动探针（`startupProbe`）与存活探针（`livenessProbe`），保证应用一直处于可用状态。
- 支持维护状态：当维护状态（`maintenance`）属性值为 true 时，即进入维护状态。当进入维护状态时，`HPA` 将会失效，所有的 `Deployment` 和 `StatefulSet` 的 `replicas` 将被设置 0。

## 组件依赖
&emsp;&emsp;本系统不依赖外部组件。

## 部署
### 修改集群节点信息
&emsp;&emsp;由于只有指定点节才可以与外部业务系统通信，因此需要为这个节点添加标签，然后 Egress 通过节点选择器固定到这个节点上。推荐为这个节点添加 `cluster.k8s/egress: enabled` 标签：

```bash
# 为 node5.cluster.k8s 节点添加标签
$ kubectl label node node5.cluster.k8s cluster.k8s/egress=enabled
node/node5.cluster.k8s labed
```

### 创建配置文件
&emsp;&emsp;创建配置文件 `egress.yaml`，将 `values.yaml` 中的内容复制到该文件中。删除其余无用的配置，保留需要修改的内容，如下：

```yaml
# 维护状态
# 当进入维护状态时，hpa 将会失效，所有的 Deployment 和 StatefulSet 的 replicas 将被设置 0
maintenance: false

# 资源限制
resources:
  limits:
    cpu: 500m
    memory: 128Mi

# 自动横向拓展
autoscaling:
  # 是否启用横向拓展。如果不启用，则默认启动 minReplicas 个副本
  enabled: true
  # 最小副本数量
  minReplicas: 2
  # 最大副本数量
  maxReplicas: 5
  # CPU 超过 resource.limits.cpu 里面限制的 60% 就触发横向拓展
  targetCPUPercentage: 60

# 代理路由
routers:
  - name: dashboard               # 服务名（必填）
    scheme: https                 # 以指定协议访问服务（必填）
    host: dashboard.cluster.k8s   # 以指定的 Host 名访问服务（选填）
    targets:                      # 服务地址列表（必填）
      - 10.10.20.21:443           # 服务的真实 IP 地址、端口。如果有多个，将会自动做负载均衡
      - 10.10.20.21:443
    options:                      # 代理选项
      connectTimeout: 60          # 连接超时时间，不填时默认 10s
      sendTimeout: 60             # 发送数据超时时间，不填时默认 10s
      readTimeout: 60             # 读取数据超时时间，不填时默认 10s
      clientMaxBodySize: 1024m    # 最大请求体大小，不填时默认 16m
  - name: nexus
    scheme: http
    host: mirror.cluster.k8s
    targets:
      - 10.10.20.0
```

### 部署应用
&emsp;&emsp;使用 helm 执行以下命令，完成部署：

```bash
# 使用 egress.yaml 指定的参数部署
$ helm install <release-name> oci://registry-1.docker.io/centralx/helm-egress -f egress.yaml
```

## 使用
&emsp;&emsp;Egress 完成部署后会根据上面的路由配置中的 `routers.name` 生成 Kubernetes Service，因此现在就可以在集群内部通过 `http://dashboard` 或 `http://nexus` 的方式调用外部系统的接口了。

```bash
# 进入集群的 pod
$ kubectl exec -it <pod> -- bash

# 在 pod 里模仿应用访问外部系统
$ curl http://nexus
```