---
layout: post
title: "lstio+envoy"
subtitle: ""
date: 2024-07-09
author: "fjw"
header-img: "img/contact-bg.jpg"
tags: [k8s,网络]
---

# Gateway API

**Gateway API** 是 Kubernetes 官方推出的新一代网络 API 规范（从 2019 年开始开发），专门用于管理 Kubernetes 集群内的**流量路由、负载均衡、服务暴露和流量治理**。它被设计成 **Ingress API 的继任者**（next-generation Ingress），但功能更强大、更灵活、更标准化。

简单来说： **Gateway API = Kubernetes 原生的“高级入口 + 内部路由”管理标准**，它取代了传统 Ingress 的局限性，让不同 Ingress Controller（如 Istio、Envoy Gateway、NGINX Gateway Fabric、Traefik、Kong 等）能用同一套统一的 CRD（Custom Resource Definition）来配置流量规则。

## Gateway API 的核心组成部分

Gateway API 采用**角色导向（role-oriented）\**的设计，把配置拆分成几个独立的资源，让\**基础设施团队**和**应用开发团队**可以分开管理：

| 资源类型                                                     | 谁负责配置    | 主要作用                                                     | 示例场景                                                 |
| ------------------------------------------------------------ | ------------- | ------------------------------------------------------------ | -------------------------------------------------------- |
| **GatewayClass**                                             | 集群管理员    | 定义一类 Gateway 的控制器实现（如 "istio" 或 "envoy"）       | 声明“我用 Istio 来管 Gateway”                            |
| **Gateway**                                                  | 集群/平台团队 | 定义流量入口点（监听端口、协议、TLS、外部 IP 等），相当于“负载均衡器实例” | 暴露 80/443 端口的公网入口                               |
| **HTTPRoute** / **GRPCRoute** / **TCPRoute** / **UDPRoute** / **TLSRoute** | 应用开发团队  | 定义路由规则（host、路径、header、method、权重拆分、重定向等） | 把 /api/v1 路由到 service A，把 /api/v2 路由到 service B |
| **ReferenceGrant**                                           | 应用/安全团队 | 跨 namespace 引用资源的安全授权                              | 允许 dev namespace 引用 infra 的 Gateway                 |

- **v1.0**：核心稳定。
- **v1.4**：新增标准通道特性，如更多协议支持、更好的多租户隔离等。
- 当前：Gateway API 已非常成熟，被视为 Kubernetes 网络的未来标准，许多项目（如 Istio、Envoy Gateway）已全面迁移支持。

## Gateway API vs 传统 Ingress（最常见对比）

| 维度             | 传统 Ingress API                              | Gateway API                                              | 谁更好？（2026 年视角） |
| ---------------- | --------------------------------------------- | -------------------------------------------------------- | ----------------------- |
| **协议支持**     | 只 HTTP/HTTPS                                 | HTTP/HTTPS + gRPC + TCP + UDP + TLS passthrough          | Gateway API 完胜        |
| **路由表达能力** | 基本 host/path，高级靠注解（vendor-specific） | 原生支持 header、query、method、权重、镜像、超时、重试等 | Gateway API 更强大      |
| **角色分离**     | 一个 Ingress 资源混杂入口 + 路由规则          | Gateway（入口） vs Route（路由），多租户友好             | Gateway API 更安全      |
| **跨 namespace** | 很难（需 annotation hack）                    | 原生支持（ReferenceGrant）                               | Gateway API 胜出        |
| **标准化**       | 注解碎片化（nginx、traefik、istio 各不同）    | 统一 CRD，所有实现者遵循同一规范                         | Gateway API 未来性强    |
| **可扩展性**     | 有限                                          | 支持实验通道 + 社区扩展（如 Inference Extension）        | Gateway API 更开放      |
| **维护状态**     | ingress-nginx 等主流控制器 2026 年 3 月后 EOL | 活跃发展，Kubernetes 官方 SIG-NETWORK 项目               | Gateway API 推荐迁移    |

## Istio 与 Gateway API 的关系

Istio 从 **1.22**（2024 年）开始就全面支持 Gateway API（GA），到 Istio 1.27/1.28 已经非常稳定：

- Istio 可以用 Gateway API 来配置 **Ingress**（north-south 流量）和 **Mesh**（east-west 内部流量）。
- 推荐方式：用 Gateway + HTTPRoute / GRPCRoute 替换旧的 Gateway + VirtualService。
- Istio Ambient 模式下，Gateway API 支持更好（更简洁）。
- 迁移路径：Istio 官方有工具和文档支持从 Ingress / VirtualService 迁移到 Gateway API。

一句话总结： **Gateway API 是 Kubernetes 为了解决 Ingress 的各种痛点而推出的“下一代入口和路由标准”**，它更通用、更表达力强、更注重角色分离和多协议支持。当前已经是主流选择，如果用 Istio，直接用 Gateway API 配置入口和服务网格是最推荐的做法（比老的 VirtualService 简单多了）。

## 具体 YAML 示例

以下是一个使用 **Istio + Gateway API** 暴露 gRPC 服务的完整 YAML 示例（基于 Istio 1.22+ 和 Gateway API v1.0+ 的稳定支持，2026 年主流实践）。

### 前提条件

- Istio 已安装（istioctl install --set profile=demo 或包含 ingress gateway）。
- Gateway API CRDs 已安装（Istio 会自动处理，或手动 kubectl apply -k github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.1.x）。
- gRPC 服务已部署，包括：
  - Deployment（gRPC server）
  - Service（ClusterIP 类型，端口 50051）

### 示例：暴露 gRPC 服务（外部访问）

假设你的 gRPC 服务：

- Service 名：grpc-echo
- 端口：50051
- 协议：gRPC（基于 HTTP/2）
- 外部域名：grpc.example.com（需 DNS 解析到 Istio Ingress Gateway 的外部 IP）

#### 1. 创建 Gateway（入口定义，由平台管理员管理）

```yaml
# gateway-grpc.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: grpc-gateway
  namespace: istio-ingress  # 通常放在 istio-ingress 或 istio-system
spec:
  gatewayClassName: istio  # Istio 实现的 GatewayClass
  listeners:
  - name: grpc-listener
    protocol: GRPC          # 或 HTTP2（gRPC 基于 HTTP/2）
    port: 50051             # 外部暴露端口（可改成 443 + TLS）
    allowedRoutes:
      namespaces:
        from: All           # 允许所有 namespace 的 Route 引用
```

#### 2. 创建 GRPCRoute（路由规则，由应用团队管理）

```yaml
# grpcroute.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: grpc-echo-route
  namespace: default  # 你的 gRPC 服务所在 namespace
spec:
  parentRefs:
  - name: grpc-gateway
    namespace: istio-ingress  # Gateway 所在 namespace
  hostnames:
  - "grpc.example.com"          # 外部访问域名
  rules:
  - matches:
    - method:
        service: "echo.EchoService"   # gRPC 服务全名（从 proto 定义）
        method: "SayHello"            # 可选：具体方法匹配
    backendRefs:
    - name: grpc-echo             # 你的 Kubernetes Service 名
      port: 50051
      kind: Service
```

- 如果不需要 method 匹配（所有 gRPC 请求都转发），可以省略 matches 部分。

- 支持权重拆分（traffic split）：

  ```yaml
  backendRefs:
  - name: grpc-echo-v1
    port: 50051
    weight: 80
  - name: grpc-echo-v2
    port: 50051
    weight: 20
  ```

#### 3. （可选）启用 mTLS 或 TLS 终止

如果想用 TLS（推荐生产）：

- 在 Gateway 加 TLS 配置：

  ```yaml
  listeners:
  - name: grpc-tls
    protocol: TLS
    port: 443
    tls:
      mode: Terminate          # 或 Passthrough
      certificateRefs:
      - kind: Secret
        name: grpc-tls-secret  # kubectl create secret tls ...
  ```

- Istio 会自动处理 mTLS（如果 mesh 配置了 STRICT mTLS）。

#### 4. 应用和测试

```shell
# 应用
kubectl apply -f gateway-grpc.yaml
kubectl apply -f grpcroute.yaml

# 查看 Gateway 状态
kubectl get gateway -n istio-ingress grpc-gateway -o yaml

# 查看外部 IP（LoadBalancer 或 NodePort）
kubectl get svc -n istio-system istio-ingressgateway

# 从外部测试（用 grpcurl 或你的 gRPC 客户端）
grpcurl -plaintext -proto your.proto grpc.example.com:50051 echo.EchoService/SayHello
```

### 注意事项

- **GRPCRoute 支持**：Istio 从 1.20+ 开始实验支持 GRPCRoute，到 1.24+ 基本稳定（但部分高级 filter 可能仍需 VirtualService 补充）。如果 GRPCRoute 未完全生效，可 fallback 用 HTTPRoute（gRPC 兼容 HTTP/2）：

  YAML

  ```
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: grpc-httproute
  spec:
    parentRefs:
    - name: grpc-gateway
    hostnames:
    - "grpc.example.com"
    rules:
    - matches:
      - headers:
        - type: RegularExpression
          name: ":method"
          value: "POST"  # gRPC 都是 POST
      backendRefs:
      - name: grpc-echo
        port: 50051
  ```

- **内部 east-west**：GAMMA 模式下，GRPCRoute 可以 parentRefs 到 Service（非 Gateway），实现网格内 gRPC 路由。

- **调试**：用 istioctl proxy-config routes <pod> 查看 Envoy 配置，或 Kiali 查看流量图。

# Service Mesh

**服务网格（Service Mesh）** 是现代微服务架构中一种**专用的基础设施层**，专门负责处理**服务之间（service-to-service）的通信**。它把原本散落在每个微服务代码里的网络逻辑（如路由、安全、观测等）抽离出来，统一放到这个“网格”层来管理，让开发者可以更专注业务逻辑，而不是纠结于分布式系统的复杂性。

简单一句话定义（来自 William Morgan，Linkerd 创始人，2017 年经典表述，至今仍是权威）：

> 服务网格（Service Mesh）是一个**基础设施层**，用于处理服务间通信。云原生应用有着复杂的服务拓扑，服务网格保证请求在这些拓扑中可靠地穿梭。在实际应用中，服务网格通常是由一系列轻量级的**网络代理**组成的，它们与应用程序一起部署，但**对应用程序透明**。

### 服务网格的核心组成部分

服务网格通常分为两个平面：

1. **数据平面（Data Plane）**
   - 真正处理流量的部分，由一个个**轻量级代理**（proxy）组成。
   - 最常见的是 **sidecar 模式**：每个微服务 Pod 旁边注入一个代理容器（如 Envoy），拦截所有进出流量。
   - 2024-2026 年新趋势：**sidecar-less / Ambient 模式**（Istio Ambient、Cilium Service Mesh），用节点级代理（如 ztunnel）或 eBPF 代替每个 Pod 的 sidecar，大幅降低资源开销。
   - 代理干的活：
     - 负载均衡、路由、重试、超时、熔断、流量镜像、故障注入
     - mTLS（自动双向 TLS 加密 + 身份验证）
     - 生成指标、日志、分布式追踪
2. **控制平面（Control Plane）**
   - 大脑，负责配置和管理所有代理。
   - 常见实现：Istio 的 istiod、Linkerd 的 control plane、Cilium 的 Hubble 等。
   - 通过 API（如 CRD 或 Gateway API）接收用户策略，然后动态推送给数据平面（xDS 协议）。

### 服务网格

微服务多了之后，服务间调用会遇到这些问题：

- **网络可靠性**：重试、超时、熔断谁来管？
- **安全**：服务间要不要加密？怎么做零信任？证书怎么发？
- **可观测性**：调用链路哪里慢？哪里出错了？指标、日志、追踪怎么统一？
- **流量控制**：金丝雀发布、A/B 测试、灰度、限流、权重路由怎么实现？
- **开发负担**：每个服务都重复写这些逻辑，代码臃肿、语言不统一。

服务网格把这些“横切关注点”统一抽到基础设施层，应用代码零改动（透明代理），就能享受以上能力。

### 主流服务网格对比

| 服务网格          | 数据平面实现                  | 主要特点                               | 资源开销               | 社区活跃度   | 典型场景                     |
| ----------------- | ----------------------------- | -------------------------------------- | ---------------------- | ------------ | ---------------------------- |
| **Istio**         | Envoy（sidecar 或 Ambient）   | 功能最全、L7 最强、Gateway API 支持好  | 中～高（Ambient 低）   | 最高         | 大型企业、复杂路由、安全需求 |
| **Linkerd**       | linkerd-proxy                 | 极简、性能高、开箱即用                 | 最低                   | 高           | 中小型团队、追求简单高效     |
| **Cilium**        | eBPF + Envoy（可选）          | sidecar-less、内核级高性能、网络策略强 | 最低                   | 快速增长     | 性能敏感、eBPF 爱好者        |
| **Istio Ambient** | ztunnel（L4）+ waypoint（L7） | Istio 的 sidecar-less 模式             | 比传统 Istio 低 50-80% | Istio 子项目 | Istio 用户想省资源           |

2026 年趋势：sidecar-less / Ambient / eBPF 模式越来越主流，因为传统 sidecar 模式在 Pod 多、资源紧张的环境下开销太大（每个 Pod 多一个代理容器）。

### 总结：一句话理解

**服务网格 = “微服务之间的智能交通系统”**

- 它不改变你的业务代码
- 但让所有服务间的“道路”变得可控、安全、可观测、高可靠
- 你只需要关注“去哪”（业务逻辑），网格负责“怎么走”（网络治理）

# 介绍

**Istio** 和 **Envoy** 是微服务架构中 **服务网格（Service Mesh）** 的核心组件，它们密切合作，但角色完全不同。下面用通俗的话解释它们各自“做了什么”，以及它们的关系（基于 Istio 最新架构，2026 年视角）。

### Envoy 做了什么？

Envoy 是 Istio 的**数据平面（Data Plane）**核心，也是一个独立的高性能代理（proxy），最初由 Lyft 开发，用 C++ 写的。

Envoy 的主要工作：

- **拦截和处理所有网络流量**：它像一个“智能门卫”，负责微服务之间（east-west 流量）和进出网格（north-south 流量）的所有进出请求。
- **具体干的活**（L3/L4 + L7 全栈）：
  - **负载均衡**：轮询、随机、最少连接、基于权重的分发等。
  - **路由与流量管理**：根据 header、路径、权重做路由、A/B 测试、金丝雀发布、流量镜像、故障注入（delay、abort）。
  - **熔断、超时、重试**：自动处理下游服务不稳定。
  - **mTLS（双向 TLS）**：自动加密服务间通信 + 身份验证（零信任安全）。
  - **可观测性**：生成详细的访问日志（access log）、指标（metrics，如 Prometheus 格式）、分布式追踪（tracing，如 Jaeger/Zipkin）。
  - **协议支持**：HTTP/1.1、HTTP/2、gRPC、TCP、TLS 等，几乎所有现代微服务协议。
  - **扩展性**：支持 Wasm（WebAssembly）插件自定义过滤器，Envoy 自己就能做很多高级功能。

一句话：**Envoy 是干活的“代理工人”**，它真正处理每一个字节的网络流量。

### Istio 做了什么？

Istio 是**服务网格的完整解决方案**，它把 Envoy 包装成一个易用的系统，提供**控制平面（Control Plane）** 来统一管理多个 Envoy。

Istio 的主要工作：

- **简化 Envoy 的配置**：Envoy 原生配置超级复杂（成千上万行 YAML），Istio 用高级 CRD（如 VirtualService、DestinationRule、Gateway、AuthorizationPolicy）抽象出来，让你用几行 YAML 就能实现高级路由、安全策略。
- **动态服务发现**：从 Kubernetes API 实时拉取 Service/Endpoints/Pod 信息，通过 xDS 协议推送给所有 Envoy（或 ztunnel），让 Envoy 知道“谁在哪、谁健康”。
- **统一管理**：一个 istiod（控制平面）管理整个网格的配置、安全、证书（自动签发 mTLS 证书）。
- **两种模式支持**（2026 年主流）：
  - **Sidecar 模式**：每个 Pod 注入一个 Envoy sidecar（经典方式）。
  - **Ambient 模式**：节点级 ztunnel（L4）+ 可选 waypoint（L7 Envoy），省资源、无需每个 Pod 加 sidecar。
- **额外能力**：Kiali 可视化仪表盘、Prometheus/Grafana 集成、Jaeger 追踪、外部流量管理（Ingress/Egress Gateway）等。

一句话：**Istio 是“老板”**，它指挥 Envoy 干活，提供用户友好的 API 和统一治理。

### Istio 和 Envoy 的关系总结（对比表）

| 方面           | Envoy（独立）                       | Istio（+ Envoy）                                | 谁更重要？（在 Istio 生态中） |
| -------------- | ----------------------------------- | ----------------------------------------------- | ----------------------------- |
| **角色**       | 数据平面代理（干活的）              | 完整服务网格（控制平面 + 数据平面）             | Istio 依赖 Envoy              |
| **配置复杂度** | 高（原生 xDS/YAML 很繁琐）          | 低（用 CRD 抽象，简单几行实现复杂功能）         | Istio 简化了 Envoy            |
| **功能范围**   | 强大（路由、mTLS、观测等）          | 更完整（+ 多集群、Ambient 模式、Kiali 等）      | Istio 扩展了 Envoy            |
| **部署方式**   | 可以单独用（edge proxy、L7 网关等） | 必须用 Envoy 作为数据平面（sidecar 或 ambient） | Istio 内置 Envoy              |
| **资源开销**   | Sidecar 模式高，Ambient 低          | Ambient 模式大幅降低（ztunnel 共享）            | Istio Ambient 优化了 Envoy    |
| **可扩展**     | Wasm 插件、ext-proc 等              | 通过 EnvoyFilter/Wasm + Istio CRD 扩展          | 两者结合最强                  |

**一句话概括**： **Envoy 是 Istio 的“心脏和肌肉”**（真正处理流量的引擎），**Istio 是 Envoy 的“大脑和指挥部”**（让 Envoy 好用、统一管理）。 Istio 离开了 Envoy 就没法干活，但 Envoy 可以独立用（比如做 API 网关、L7 负载均衡），Istio 让 Envoy 在 Kubernetes 微服务网格里发挥最大价值。

# 安装

\# 1. 下载最新版 Istio

```shell
curl -L https://istio.io/downloadIstio | sh - 
cd istio-*
export PATH=$PWD/bin:$PATH 
which istioctl   # 确认可用
# 直接安装 default/demo 模式
istioctl install --set profile=demo -y
# 或者更省资源用 default
istioctl install -y
# 开启自动注入（绝大多数人要的）
kubectl label namespace default istio-injection=enabled --overwrite
# 以后新建 Deployment 就会自动加 sidecar 了

```
