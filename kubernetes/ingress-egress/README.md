# Ingress - 入口网关

>网关如何暴露到集群外部：[kubernetes 如何将服务暴露到外部](../kubernetes%20如何将服务暴露到外部.md)

## 基础概念

网关 Gateway，就是指一个流量的集中式出入口，网关有两种类型：

1. Ingress Gateway: 入网网关，所有请求都应该从这里进入集群。
2. Egress Gateway: 出口网关，顾名思义，它用于控制所有从集群出去的流量。

对于一个独立的网络，我们通常对入口的管控要求比较严格，而对出口的要求不高，Kubernetes 原生也没有提供 Egress Gateway 这项功能。
我想到的几个原因如下：

1. 技术问题：目前已有的方案里，只有通过服务网格，才能实现 Egress Gateway, 比如 Istio.
2. 成本问题：前面讲了添加 Egress Gateway 需要 Istio，这会带来性能下降。
3. 繁琐程度：Egress Gateway 要求声明好所有需要访问的外部服务，比较繁琐。
4. 懒？：很多人觉得内网已经足够安全了，只要在入口安全性做好了，内网是安全的，出口就不需要管控。

## 一、Ingress / API Gateway

Ingress 是 Kubernetes 中目前最成熟的入口网关资源，但是它的设计比较有局限性，许多能力都得通过 annotation 提供支持。

而 API Gateway 是一个通用的概念，指功能更丰富的入口网关。
具体而言，API Gateway 就是比普通的网关多干了一些以前我们在应用内部实现的事：身份认证，权限控制，基于来源的流量控制，日志服务等等。

>目前官方开发了新一代的网关资源：Gateway API，目前已进入阶段，正在如火如荼地落地中。

Ingress 资源对应的控制器被称作 Ingress Controller，它负责使集群状态符合 Ingress 资源的描述。
但是现代 Kubernetes 集群中，我们常用的网关会更符合 API Gateway 的定义，功能要比 Ingress 这个官方资源的定义强很多，通常都使用 CRD 来实现。

这里着重介绍几个比较流行的社区实现，它们大都提供了一些更高级的特性，比如 JWT 身份验证、来源限流、或者直接提供了插件系统。

>可以在 [API Gateway - CNCF Cloud Native Landscape](https://landscape.cncf.io/card-mode?category=api-gateway&grouping=category) 中找到几乎所有流行的云原生 API 网关。

这样网关大都基于 Nginx/Openresty 或者 Envoy 实现，只有少部分网关是使用 Go/Java 等语言完全重新造的轮子。
其中 Nginx/Openresty 凭借其极高的性能与可拓展性占据一席之地，而 Envoy 则凭借其丰富的功能与面向云原生的设计占据了云原生领域的半壁江山。

>Nginx 在很长一段时间内都是 L7 负载均衡领域的王者，但是随着云计算的发展，企业对 API 网关的需求越来越强烈，开源 Nginx 本身渐渐无法满足需求。
最大的问题是 Nginx 的配置比较复杂，而且不够动态，无法满足 Kubernetes 网关以及其他场景的需要。而新一代 API 网关则瞄准了这一需求，提供基于 API 的动态配置方式，内置网关常用的所有能力，赢得了许多用户的喜爱。

基于 Envoy 实现的 API 网关：

>感觉其中 emissary 是比较好的候选项，支持跟 Istio 集成，底层也是 Envoy，功能看起来也很强大。

- Istio IngressGateway
  - Istio 服务网格的原生网关，基于 Envoy 开发
- https://github.com/emissary-ingress/emissary
  - 官方支持与 Istio 集成
    - 需要注入一个 Envoy Sidecar，但是好像流量不会过 Sidecar，Sidecar 仅用于发现与提供一些 Istio mTLS 证书等支持
- https://github.com/solo-io/gloo
  - gloo 公司是 Istio 社区的核心贡献者之一，很活跃。
  - CRD 跟 istio 很像，不过多一些 GraphQL/OpenAPI/Knative 支持
  - 仅企业版才支持用做 Mesh Gateway
- https://github.com/projectcontour/contour
  - 官方暂未提供 Istio 支持，相关 Issue 已经沉寂两年了

基于 Ngninx/Openresty 实现的 API 网关：

- APISIX：国产项目，完全开源，借鉴了 Kong 的方案，但是官方宣称性能更好，架构更优。
  - APISIX 及其插件是完全开源的，而且功能非常丰富，作为网关用感觉很合适，可以考虑跟 Istio 服务网格结合使用。
- Kong：国外老牌项目，部分关键插件是收费的。
  - Kong 跟 APISIX 都是完全基于 Nginx/Openresty 实现的，但是 Kong 更偏商业化一些，许多插件都是收费的...
  - 根据我的简单压测，APISIX 的性能要略优于 Kong，尤其是 Kong 的部分插件性能比较差，这个好像都比较知名了，很多人都清楚 Kong 的插件性能问题...


其他 API 网关：

- https://github.com/traefik/traefik
  - 非常知名的使用 Go 实现的网关，很多公司都在用

其他非著名 API 网关（应用场景都比较特殊，除非特别契合需求，否则请谨慎选用）：

- https://github.com/megaease/easegress
  - 著名的左耳朵耗子陈皓及其他大佬创建的新项目，主要目的是提供强大的流量编排能力。
  - 其 CRD 结构跟 argo workflow 类似，可以将网关的功能拆分成一个个 filter，然后当成 pipeline 来编排，感觉确实比较有意思。
  - 相关文章 [MegaEase流量网关Easegress介绍](https://cloud.tencent.com/developer/article/1868083)
- https://github.com/alibaba/Sentinel
  - 通过侵入式的 SDK 实现流量治理（流量路由、熔断降级、限流等）
  - 最开始是一个仅支持 Java 的流量治理组件，后来陆续推出了 C++/Go/Rust SDK，以及 Envoy 支持
- https://github.com/TykTechnologies/tyk
  - 2014 年开始的项目，纯 Go 写的，优势有
    - 支持直接导入 OpenAPI 定义
    - 支持 GraphQL
- https://github.com/luraproject/lura: 这个项目是一个 go library，可使用它构建自己的 API 网关
  - 它的主要功能是做聚合网关，将多个 API 返回的数据聚合到一起，去除不必要的信息，再返回给客户端
- https://github.com/apioak/apioak
  - 基于 Nginx 跟 lua，活跃度太低了，不适合选用


## 二、Ingress / API Gateway 与服务网格集成 {#api-gateway-plus-service-mesh}

>这里仅讨论如何将第三方网关跟服务网格集成，Istio 自带的 Ingress 就略过不讨论了，想研究 Istio IngressGateway 请移步 [/kubernetes/service_mesh/istio/Ingress.md](/kubernetes/service_mesh/istio/Ingress.md).

将 Ingress / API Gateway 与 Istio 等服务网格集成的最佳方法，就是把 Ingress 也当成一个普通服务来处理，注入一个 Sidecar Proxy，[如何为服务网格选择入口网关？ - 赵化冰](https://zhaohuabing.com/post/2019-03-29-how-to-choose-ingress-for-service-mesh/) 里给的架构图如下：

![](./_img/api-gateway-plus-istio.png)

虽然此方案在 Gateway 旁边加了一个 Sidecar Proxy，但是它使任何 API Gateway 都能直接无痛接入 Istio 服务网格，哪怕这个网关是一个原始的基于配置文件的 Nginx 容器！都能直接利用上服务网格的 L7 负载均衡、流量分配、故障注入等等能力。

如果你们以前使用的是将 Nginx 部署在虚拟机上的方案，那么改造的第一步，就应该是将 Nginx 放在 Kubernetes 集群里部署并扩缩容，并且注入 Sidecar Proxy 以接入 Istio 服务网格，而外部则可通过云服务商的 L4 负载均衡、或者自建 keepalived(vrrp+ipvs) 对外提供高可用的静态公网 IP。

架构图如下：

```mermaid
flowchart LR
    C[Client] -->|Traffic| L4[L4 Proxy] --> G(Nginx Gateway)
    G --> P[Sidecar Proxy]
    P --> P_A[Sidecar Proxy] <--> A[Service A]
    P_A <---> P_D[Sidecar Proxy] <--> D[Service D]
    P --> P_B[Sidecar Proxy] --> B[Service B]
    subgraph Kubernetes
        subgraph Nginx Pod
        G
        P
        end
        subgraph Pod A
        A
        P_A
        end
        subgraph Pod D
        D
        P_D
        end
        subgraph Pod B
        B
        P_B
        end
    end
```


## 三、最终的选型 - APISIX

我厂换网关的主要目的，一是解决 nginx reload 问题（所有客户端连接会全部断掉重连、代价太高了）、二是 json/yaml 格式的网关配置可以方便做些自动化配置变更之类的事。

我调研的网关候选项主要是 envoy 系跟 nginx 系，也就是前面两节列出的所有网关。

istio ingressgateway 功能有限，有些旧 nginx 已有的功能不太好迁移到这边来，而且它的拓展相比 nginx 系也更复杂，写 lua 的话各种依赖都得自己解决（比 openresty 差太多），写 wasm 的话一个是吃内存、性能不好，二是 wasm 插件编写应该也不简单。

nginx 这边 nginx-ingress-controller 貌似主要通过 ingress + 各种 annotations 完成工作，感觉限制也比较多。

再排除掉一堆不敢上生产环境踩坑的不知名项目，最后就剩 apisix 跟 kong。

Kong 被淘汰的主要原因是，它的一些插件性能确实不如 APISIX，而且很多插件都收费。

所以最后选择了 APISIX，它的主要优势：

1. 所有插件都是开源的，官方的插件非常全，基本覆盖了所有常见需求，性能跟 Kong 持平或者更好
   1. APISIX 的收费模式跟 Kong 不一样，它主要是靠企业级多租户 dashboard、企业级支持等功能来赚钱，插件全部开源
2. 遇到问题去 github 提 issue/discussion 都能得到很认真的回复,社区响应也很快，沟通很顺畅
   1. 社区帮我解决了很多困惑跟问题，省了我自己不少时间，让网关改造项目得以顺利上线，必须点赞，比如 [apisix/discussions/7773](https://github.com/apache/apisix/discussions/7773) 跟 [apisix/discussions/7840](https://github.com/apache/apisix/discussions/7840)。
      1. 反例是很多大社区帮助解决问题的官方人员都很少，很多问题经常得靠自己。
   2. 他们的人在 twtter 也很活跃，偶尔发个网关相关的帖子还能引来 APISIX 员工参与讨论
3. APISIX 的 [@Zexuan Luo](https://twitter.com/spacewanderlzx) 经常在推特上分享 APISIX 的底层技术与网关相关技术内容，也算一个了解 APISIX 网关底层原理与发展方向的渠道。
