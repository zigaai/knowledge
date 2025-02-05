# GitOps CD


### Flux vs Argo CD vs Jenkins-X

1. Flux: 简单小巧，只做一件事，并且把这件事做得很好。
2. Argo CD: 功能与 Flux 类似，但是功能比 Flux 丰富很多，支持多租户多集群多名字空间，可接入 LDAP。有 Web UI。
3. Jenkins-X: 最复杂的一个工具，同时包含了 CI 和 CD。目前不建议使用。

参考：[FluxCD、ArgoCD或Jenkins X，哪个才是适合你的GitOps工具？ ](http://dockone.io/article/10175)


### 存在的问题：资源更新顺序

一个复杂的微服务系统，各微服务之间的依赖关系是很复杂的，这往往要求我们按照依赖链路进行更新。

首先后端的更新策略是：

1. 微服务 API 的每次更新都应该保持向下兼容，保证旧客户端仍然能正常调用此微服务。
1. 客户端升级时可以直接使用新的 API

这导致，微服务更新时，我们**必须在服务端 A 升级完成后，才能升级客户端 B**。因为客户端不向下兼容，直接升级会报错！

后端人员会一次性把最新的微服务全部提交给运维组，运维发布系统必须要自己去根据预先定义好的升级顺序，进行按序升级。

然而目前已有的 kubernetes CD 工具，都没有提供类似的按依赖关系进行部署的功能。因此目前我们是自己实现的这样一个需求。

