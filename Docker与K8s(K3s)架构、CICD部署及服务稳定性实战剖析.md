# Docker与K8s\(K3s\)架构、CICD部署及服务稳定性实战剖析

结合你当前**腾讯云K3s集群\+原生Docker混合部署**的生产场景，核心痛点集中在「容器网络通信不一致、部署模式不统一、CICD流程割裂、服务稳定性无统一保障方案」。本文将从架构本质、核心技术点、网络通信原理（解决你的核心网络问题）、CICD落地流程、稳定性保障方案、常用实操命令、全流程模拟实操、核心资源深度对比剖析全方位拆解，适配K3s轻量集群特性，贴合云上生产落地场景。

# 一、核心架构剖析：Docker vs K8s\(K3s\)，彻底理清混合部署冲突根源

## 1\.1 原生Docker架构（单机容器架构）

Docker是**单机容器化引擎**，核心定位是「打包、运行容器」，仅负责单节点的容器生命周期管理，无集群调度能力，这也是你独立Docker服务和K3s集群服务通信异常的根本原因。

### 核心架构组件

- **Docker Client**：命令行交互入口（docker build/run/push），接收用户指令

- **Docker Daemon\(dockerd\)**：后台守护进程，核心服务，管理本机镜像、容器、网络、存储

- **Containerd**：容器运行时（Docker 1\.19\+默认替换旧版shim），负责容器创建、启停、资源隔离，是真正的容器执行载体

- **OCI镜像规范**：统一容器镜像标准，Docker镜像可直接在K3s/K8s中运行，这是混合部署的唯一兼容点

### Docker网络架构（单机隔离，跨节点无法互通）

Docker默认4种网络模式，所有网络均为**单机级别**：bridge（默认网桥）、host、none、container。同一主机Docker容器可通过网桥通信，**跨主机、跨K3s集群节点无法直接互通**，这是你混合部署网络不通的核心症结。

## 1\.2 K8s/K3s集群架构（分布式容器编排架构）

K3s是**轻量版K8s**，完全兼容K8s原生API，剔除了冗余组件（etcd替换为sqlite、精简监控插件），适配边缘、云上轻量集群场景，核心定位是「多节点容器集群调度、运维、治理」，解决Docker单机单点、无法集群管理的问题。

### K3s核心架构组件（控制面\+数据面）

#### 控制面（集群管理核心）

- **kube\-apiserver**：集群唯一入口，所有操作指令、资源读写均通过该组件，认证鉴权

- **kube\-controller\-manager**：资源控制器，维持集群期望状态（如副本数、节点状态）

- **kube\-scheduler**：调度器，将Pod分配到最优集群节点

- **sqlite数据库**：替代原生K8s的etcd，轻量化存储集群所有资源数据

#### 数据面（业务运行核心）

- **kubelet**：每个节点代理，负责当前节点Pod创建、启停、状态上报

- **kube\-proxy**：集群网络核心，实现Pod、Service、节点之间的网络转发、负载均衡

- **Containerd**：K3s默认容器运行时，和新版Docker运行时一致，保证镜像兼容性

- **Flannel**：K3s默认网络插件，提供**跨节点Pod互通的集群网络**

## 1\.3 混合部署网络冲突核心原因（精准解答你的问题）

1\. 网络体系隔离：独立Docker使用单机bridge网络，K3s使用Flannel集群网络，两套网络协议栈、网段完全独立，无默认路由互通；

2\. 服务发现机制不同：Docker无原生服务发现，仅靠IP通信；K3s依靠Service、CoreDNS实现集群内域名服务发现（如 Consul 集群域名、网关服务域名）；

3\. 端口管理冲突：Docker端口映射为单机主机端口，K3s通过Service虚拟端口、NodePort调度，规则不统一；

4\. 流量策略缺失：Docker无网络策略限制，K3s默认集群内Pod互通，但不兼容外部Docker容器流量。

# 二、Docker \& K3s 核心常用技术点（生产必备）

## 2\.1 Docker 核心技术点（适配CICD打包阶段）

- **镜像分层构建**：基于Dockerfile分层构建，缓存复用，大幅提升CICD构建速度，生产多采用多阶段构建减小镜像体积

- **镜像仓库管理**：腾讯云CR、DockerHub私有仓库，CICD流程核心环节：构建\-推送镜像

- **资源配额**：可限制容器CPU、内存，避免单机资源抢占

- **单机日志/数据卷**：volume数据持久化，避免容器删除数据丢失

## 2\.2 K3s\(K8s\) 核心技术点（适配CICD部署、运维阶段）

### 2\.2\.1 核心资源对象（CICD部署核心）

- **Pod**：最小部署单元，一个Pod包含一个或多个业务容器，K3s调度的基本对象

- **Deployment**：无状态服务部署核心，管理Pod副本、滚动更新、回滚，生产90%业务使用

- **Service**：固定服务入口，提供集群内静态IP\+域名，实现Pod负载均衡和服务发现（解决动态PodIP变动问题）

- **Ingress**：集群入口网关，统一管理外部域名、端口转发，替代单机Docker端口暴露

- **ConfigMap/Secret**：配置、密钥统一管理，CICD流程中动态注入配置，无需打包进镜像

- **PV/PVC**：集群持久化存储，适配腾讯云磁盘/NFS，实现容器数据永久保存、跨命名空间共享

### 2\.2\.2 核心能力（稳定性核心）

- **自愈能力**：Pod异常退出、节点故障时，自动重启、重新调度

- **滚动更新**：CICD发布时无停机更新，逐步替换Pod，保证服务不中断

- **资源调度**：自动分配节点资源，避免单点压力过高

- **网络策略**：精准控制Pod、服务之间的通信权限，提升集群安全性

# 三、基于CICD的统一部署方案（解决混合部署混乱问题）

最优生产方案：**统一收口为K3s集群部署**，淘汰独立Docker部署，彻底解决网络通信问题。所有服务通过Docker打包镜像，通过CICD流水线自动部署到K3s集群，实现构建、部署、运维统一。

## 3\.1 标准化CICD全流程（适配腾讯云K3s）

整体流程：代码提交 → 自动拉取代码 → Docker构建镜像 → 推送腾讯云CR仓库 → K3s集群拉取镜像部署 → 健康检查 → 自动回滚（异常时）

### 步骤1：代码托管

使用Gitee/GitLab，配置WebHook，代码Push后触发CICD流水线

### 步骤2：Docker标准化构建（统一镜像规范）

编写通用多阶段Dockerfile，减小镜像体积，提升安全性，所有服务统一构建规则，保证兼容K3s运行时。

### 步骤3：镜像推送私有仓库

对接腾讯云容器镜像服务CR，按「项目\-环境\-版本」规范命名镜像，统一版本管理。

### 步骤4：K3s自动化部署

流水线执行kubectl指令，更新Deployment镜像版本，触发滚动更新，无需手动操作集群。

### 步骤5：部署后健康校验

流水线自动检测Pod状态、服务端口、接口可用性，部署失败则自动回滚上一稳定版本。

## 3\.2 现有混合部署临时兼容方案（过渡期）

若暂时无法下线独立Docker服务，可通过以下方案解决网络通信：

- Docker容器使用**host网络模式**，共享宿主机网络，通过腾讯云主机内网IP与K3s集群Service通信

- K3s开启集群内网访问，通过Service的NodePort端口暴露服务，供Docker容器调用

- 统一内网网段，关闭防火墙拦截，保证宿主机与集群节点内网互通

# 四、服务稳定运行保障方案（生产核心）

## 4\.1 Docker独立服务稳定性保障（过渡期）

- **自动重启**：配置docker run \-\-restart=always，容器异常自动重启

- **资源限制**：\-\-memory、\-\-cpus限制资源，防止容器抢占主机资源

- **日志持久化**：挂载日志目录，开启日志切割，避免磁盘爆满

- **定时健康巡检**：脚本检测服务端口、接口，异常自动重启容器

## 4\.2 K3s集群服务稳定性保障（最终方案）

### 4\.2\.1 基础自愈保障

- **副本冗余**：Deployment配置replicas≥2，单Pod故障不影响服务

- **存活探针\(LivenessProbe\)**：检测容器运行状态，异常自动重启Pod

- **就绪探针\(ReadinessProbe\)**：检测服务是否启动完成，未就绪不接入流量

### 4\.2\.2 发布稳定性保障

- **滚动更新策略**：maxSurge、maxUnavailable控制更新节奏，保证更新期间服务可用

- **版本回滚**：CICD流水线集成自动回滚，部署异常一键恢复

- **灰度发布**：可配合Ingress实现流量灰度，降低发布风险

### 4\.2\.3 资源与监控保障

- **资源配额**：配置requests（请求资源）、limits（最大资源），避免资源耗尽

- **日志监控**：集成K3s自带日志组件，集中收集服务日志

- **告警机制**：节点异常、Pod重启、资源过载实时告警

# 五、生产高频实操命令（Docker\+K3s）

## 5\.1 Docker 核心常用命令

```bash
# 1. 构建镜像（CICD核心）
docker build -t 腾讯云CR地址/项目/服务:版本 -f Dockerfile .

# 2. 登录私有仓库
docker login cr.tencentcloud.com

# 3. 推送镜像
docker push 腾讯云CR地址/项目/服务:版本

# 4. 运行容器（稳定模式）
docker run -d --name=service-demo --restart=always --memory=1g --cpus=1 -p 8080:8080 镜像名

# 5. 查看容器日志
docker logs -f 容器名

# 6. 重启/停止容器
docker restart 容器名
docker stop 容器名
```

## 5\.2 K3s\(K8s\) 基础核心常用命令

```bash
# 1. 查看集群节点、Pod状态
kubectl get nodes
kubectl get pods -A -owide

# 2. 查看服务、域名、端口
kubectl get svc
kubectl get ingress

# 3. 滚动更新服务（CICD核心）
kubectl set image deployment/服务名 容器名=镜像地址:新版本

# 4. 查看部署日志、异常排查
kubectl logs -f pod名
kubectl describe pod pod名

# 5. 版本回滚
kubectl rollout undo deployment/服务名

# 6. 重启服务
kubectl rollout restart deployment/服务名

# 7. 查看集群网络信息
kubectl get networkpolicy
```

# 六、K8s核心资源深度剖析（定义\+关键配置\+Docker区别\+生产场景）

本章节为**重点补充深度知识点**，针对PV/PVC、ConfigMap、Secret、Deployment、Service五大高频核心资源，做全方位落地剖析，对比Docker同类能力，补齐生产刚需知识点，解决你日常部署、运维、踩坑核心问题。

## 6\.1 PV\&PVC 集群持久化存储（核心重点）

### 6\.1\.1 基础定义

**PV（PersistentVolume）**：集群级别的持久化存储资源，由集群管理员预先创建，代表底层真实存储设备（NFS、云硬盘、本地磁盘），独立于Pod、生命周期不随容器销毁而消失。

**PVC（PersistentVolumeClaim）**：命名空间级别的存储声明，是业务资源，由普通用户/流水线创建，作用是**向集群申请绑定PV存储资源**，挂载到Pod中使用。

核心设计思想：**存储资源与业务解耦**，管理员管底层存储\(PV\)，业务管存储申请\(PVC\)，标准化、可复用、可管控。

### 6\.1\.2 关键核心配置（生产必配）

#### PV核心配置项

- **storage容量**：定义存储最大可用空间，如1Gi、10Gi，限制业务存储上限

- **accessModes访问模式**：核心三模式
        
\- ReadWriteOnce\(RWO\)：单节点读写，默认模式，大部分云硬盘使用
        
\- ReadOnlyMany\(ROX\)：多节点只读
        
\- **ReadWriteMany\(RWX\)**：多节点多Pod同时读写（你的网关密钥共享核心依赖）
      

- **persistentVolumeReclaimPolicy回收策略**
\- Retain：保留数据（生产首选，误删不丢数据）
        
\- Delete：删除PVC后自动删除PV数据（测试环境用）
        
\- Recycle：清空数据（旧版本废弃）
      

- **storageClassName存储类**：K3s默认local\-path，用于PV与PVC精准匹配，必须一致否则绑定失败

- **底层存储配置**：NFS服务端地址、路径，云硬盘ID等真实存储信息

#### PVC核心配置项

- **namespace**：命名空间隔离，不同命名空间PVC相互独立

- **resources\.requests\.storage**：申请的存储容量，必须≤PV容量

- **accessModes**：申请的访问模式，必须与PV匹配

- **volumeName**：强制绑定指定PV（生产刚需，避免动态随机绑定错乱）

### 6\.1\.3 K8s PV/PVC vs Docker 数据卷 核心区别

|对比维度|Docker Volume（数据卷）|K8s PV/PVC|
|---|---|---|
|作用范围|单机级别，仅当前主机容器可用|集群级别，跨节点、跨命名空间共享|
|生命周期|跟随容器/主机，迁移容器需手动拷贝数据|独立生命周期，Pod重建、节点迁移数据不丢失|
|共享能力|仅单机多容器共享，无法跨节点共享|支持RWX模式，全集群多Pod跨节点读写共享|
|资源管控|无配额、无权限管控、无统一管理|支持容量配额、访问权限、回收策略精细化管控|
|适配场景|单机临时持久化、测试环境|生产集群、微服务共享数据、密钥配置持久化|
|自动化能力|无自动化，手动挂载、手动迁移|支持动态供给、自动绑定、CICD自动化部署|

### 6\.1\.4 生产落地应用场景（完全适配你的业务）

- **微服务密钥共享**：网关\(gateway\)、认证\(identity\)服务共用NFS存储的密钥文件，通过同一PV绑定多命名空间PVC，无需重复部署密钥

- **日志持久化存储**：集群所有服务日志统一挂载共享PV，集中收集、排查问题，Pod重建日志不丢失

- **配置文件统一托管**：静态配置、证书文件通过PV统一存储，全局生效，更新一次所有服务同步生效

- **数据迁移场景**：节点故障、服务迁移时，无需迁移数据，直接在新节点挂载原有PVC即可恢复业务

### 6\.1\.5 生产高频知识点与避坑总结

- PVC状态一直`Pending`：99%原因是**storageClass、accessModes、容量不匹配**，或未指定volumeName导致无可用PV

- 生产禁止使用动态PV随机绑定，必须通过volumeName强制绑定固定PV，避免环境错乱

- RWX模式仅NFS、OSS等网络存储支持，本地磁盘、普通云硬盘不支持多节点读写

- PVC删除后，PV数据保留（Retain策略），需手动清理磁盘文件，避免磁盘爆满

- K3s默认local\-path存储类仅适配本地节点，跨节点共享必须使用NFS网络存储

## 6\.2 ConfigMap 配置管理深度剖析

### 6\.2\.1 基础定义

ConfigMap是K8s**非敏感明文配置资源**，专门用于存储服务配置、环境变量、配置文件，实现**配置与镜像代码解耦**，无需重新构建镜像即可修改业务配置。

### 6\.2\.2 关键核心配置

- **metadata\.name/namespace**：配置名称与命名空间，Pod精准引用

- **data**：核心配置区，存储键值对明文配置，支持多层业务配置映射

- **envFrom\.configMapRef**：全量注入，将所有键值对转为容器环境变量（你的网关服务核心用法）

- **volume挂载**：可将ConfigMap挂载为配置文件，适配需要物理配置文件的业务

### 6\.2\.3 与Docker配置方式区别

- Docker：配置硬编码进Dockerfile/镜像，或启动命令临时传入，修改配置必须重新构建镜像或重启容器，无版本管理、无法统一管控

- K8s ConfigMap：配置独立托管，支持声明式更新、CICD动态替换、多服务复用，修改配置无需改代码镜像

### 6\.2\.4 生产场景与核心坑点

- **核心场景**：服务端口、注册中心地址（Consul）、文件路径、超时时间等非敏感配置统一管理

- **\.NET专属特性**：双下划线`__`映射层级配置，`CONSUL__ADDRESS`对应`Consul:Address`，适配框架配置规范

- **致命坑点**：ConfigMap更新**不会自动热更新Pod**，必须执行`kubectl rollout restart`滚动重启服务生效

- **权限隔离**：仅存储明文，密码、密钥、令牌绝对不能存入ConfigMap

## 6\.3 Secret 密钥资源深度剖析

### 6\.3\.1 基础定义

Secret是K8s**加密敏感资源**，专门用于存储密码、令牌、镜像仓库凭据、数据库密码等敏感信息，替代明文配置，保障生产数据安全。

### 6\.3\.2 核心类型与关键配置

- **kubernetes\.io/dockerconfigjson**：私有镜像仓库专属类型，用于拉取私有镜像（你的腾讯云CR核心用法）

- **Opaque**：通用密钥类型，存储自定义密码、令牌

- **tls**：证书专用类型，存储SSL/TLS证书

- **核心特性**：数据base64编码存储，集群内加密传输，权限精细化管控，仅指定命名空间可访问

### 6\.3\.3 Docker vs K8s Secret 区别

- Docker：密钥通过启动参数、环境变量传入，明文暴露，极易泄露，无安全管控

- K8s Secret：集中加密托管，权限隔离、动态注入、不落地明文，适配CICD密钥动态注入

### 6\.3\.4 生产应用场景

- 私有镜像仓库登录凭据（解决ImagePullBackOff镜像拉取失败）

- 数据库账号密码、Redis密钥、接口鉴权令牌

- CICD流水线动态密钥注入，全程无硬编码、无明文泄露

## 6\.4 Deployment 部署资源深度剖析

### 6\.4\.1 基础定义

Deployment是K8s**无状态服务核心部署资源**，生产99%无状态微服务（网关、业务接口、认证服务）均使用该资源，负责管理Pod副本、滚动更新、版本回滚、自愈恢复。

### 6\.4\.2 关键核心生产配置

- **replicas副本数**：定义服务运行Pod数量，生产推荐≥2实现高可用

- **strategy滚动更新策略**
\- maxSurge：最大超量Pod数
        
\- maxUnavailable：最大不可用Pod数
        
\- 单副本场景：maxSurge=0、maxUnavailable=1（解决端口冲突）
        
\- 多副本场景：maxSurge=1、maxUnavailable=0（零停机更新）
      

- **livenessProbe/readinessProbe双探针**：保障服务稳定性，剔除异常Pod、重启卡死容器

- **imagePullSecrets**：绑定私有仓库密钥，授权拉取私有镜像

- **resources资源限制**：防止服务抢占集群资源，保障集群稳定性

### 6\.4\.3 Docker容器运行 vs K8s Deployment 区别

- Docker：单容器运行，无副本、无自愈、无滚动更新，挂了需手动重启，版本更新直接停机重建

- Deployment：集群级自动化运维，自动容错、无停机更新、一键回滚、负载均衡，适配生产高可用

### 6\.4\.4 生产核心价值

- 解决服务单点故障，实现集群高可用

- 标准化CICD发布流程，无停机迭代

- 自动自愈、异常重启，大幅降低运维成本

## 6\.5 Service 服务发现资源深度剖析

### 6\.5\.1 基础定义

Service是K8s集群**固定服务入口**，屏蔽Pod动态IP变化，提供稳定集群域名\+虚拟IP，实现微服务之间稳定通信与负载均衡。

### 6\.5\.2 核心类型与适配场景

- **ClusterIP（默认）**：仅集群内访问，用于微服务内部互相调用（Consul、网关内部通信）

- **NodePort**：节点端口暴露，内网所有节点可访问，轻量集群生产首选（你的网关30040端口用法）

- **LoadBalancer**：云厂商负载均衡，外网统一入口，需付费

- **ExternalName**：域名转发，对接外部服务

### 6\.5\.3 核心关键配置

- **selector标签匹配**：精准关联Deployment管理的Pod，实现动态绑定

- **port/targetPort/nodePort**：三层端口映射，区分集群端口、容器端口、节点暴露端口

- **sessionAffinity**：会话保持，适配需要固定Pod访问的业务

### 6\.5\.4 Docker服务通信 vs K8s Service 核心区别

- Docker：容器IP动态变化，服务通信依赖固定IP/端口，重启后IP变更导致通信断裂，无负载均衡

- K8s Service：域名永久固定，屏蔽PodIP变动，内置负载均衡，集群内服务通信稳定可靠

### 6\.5\.5 集群域名解析报错终极解释（对应你的报错）

`xxx.svc.cluster.local` 是K8s CoreDNS生成的**纯集群内部域名**：

- 仅集群节点、集群Pod可解析访问

- 本地电脑、外网浏览器、非集群主机无法解析，必然报网页解析失败

- 该域名仅用于**微服务内部调用**，不对外提供访问，外网必须通过NodePort/Ingress访问

# 七、K3s生产落地全套命令、YAML与深度技术剖析（适配网关/微服务\+CICD）

本章节为你线上生产落地的**完整可复用实战体系**，包含命名空间、私有镜像密钥、配置注入、滚动更新、健康探针、NFS共享存储、临时数据同步、CICD自动化部署全流程，针对腾讯云K3s轻量集群特性做适配优化，解决微服务多命名空间隔离、配置统一管理、密钥共享、部署稳定性、镜像拉取失败等生产高频问题。

## 7\.1 命名空间 Namespace 资源隔离实战

### 核心命令

```bash
# 命令式快速创建
kubectl create namespace gateway
# 声明式幂等创建（CICD推荐，可重复执行无报错）
kubectl apply -f gateway-namespace.yaml
```

### 完整YAML配置（gateway\-namespace\.yaml）

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gateway
```

### 深度技术剖析与应用场景

- **核心作用**：实现集群资源逻辑隔离，将网关、认证、业务服务拆分独立命名空间，避免资源名称冲突、配置混杂、权限混乱。

- **命令差异**：`kubectl create`为一次性创建，重复执行报错；`kubectl apply`为幂等操作，CICD流水线必须使用，保证多次构建部署无异常。

- **生产场景**：微服务架构拆分 `gateway`网关、`identity`认证、`business`业务命名空间，实现分模块运维、权限管控、资源配额隔离。

## 7\.2 私有镜像仓库 Secret 凭据配置（解决镜像拉取失败）

### 核心命令（CICD自动化专用）

```bash
# 先删除旧密钥（忽略不存在报错，适配首次部署）
kubectl delete secret tcr-secret -n gateway --ignore-not-found
# 动态创建腾讯云CR镜像仓库凭据
kubectl create secret docker-registry tcr-secret -n gateway \
  --docker-server=ccr.ccs.tencentyun.com \
  --docker-username="${{ secrets.TENCENT_REGISTRY_USERNAME }}" \
  --docker-password="${{ secrets.TENCENT_REGISTRY_PASSWORD }}"
```

### 等效声明式YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tcr-secret
  namespace: gateway
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64编码的仓库认证信息>
```

### 深度技术剖析与应用场景

- **专属类型作用**：`kubernetes.io/dockerconfigjson`是K8s标准私有镜像仓库凭据类型，专门用于存储Docker Registry登录信息，区别于普通密钥Secret，可被容器运行时自动识别。

- **CICD适配关键点**：通过GitHub Secrets动态注入账号密码，避免硬编码泄露；密码使用双引号包裹，防止特殊字符（@、\#、\!）解析异常。

- **生产避坑**：`--ignore-not-found`解决首次部署无旧密钥导致的流水线报错，保证流水线幂等可重跑。

- **引用方式**：Deployment中配置 `imagePullSecrets`，集群拉取腾讯云私有镜像自动携带凭据，彻底解决 `ImagePullBackOff` 异常。

## 7\.3 ConfigMap 配置热注入（无侵入管理业务配置）

### 核心命令

```bash
kubectl apply -f gateway-config.yaml
```

### 完整YAML配置（gateway\-config\.yaml）

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gateway-config
  namespace: gateway
data:
  ASPNETCORE_URLS: "http://+:5040"
  CONSUL__ADDRESS: "http://consul-server.consul.svc.cluster.local:8500"
  CONSUL__SERVICE__NAME: "ApiGateway"
  KEYSETTINGS__BASEDIRECTORY: "/app/keys"
```

### 深度技术剖析与应用场景

- **核心定位**：存储**非敏感明文配置**（地址、端口、服务名、路径），替代传统打包进镜像的方式，实现配置与镜像解耦。

- **\.NET专属适配规则**：框架支持双下划线 `__` 替代层级配置，`CONSUL__ADDRESS` 对应配置文件中 `Consul:Address`，无需修改代码即可注入集群服务发现地址。

- **集群域名解析原理**：`consul-server.consul.svc.cluster.local` 是K3s CoreDNS自动生成的集群内部域名，仅集群内Pod可解析访问，外网无法直接访问（对应你之前网页解析失败的核心原因）。

- **生产特性与坑点**：ConfigMap更新后**不会自动重启Pod**，新配置不生效，需执行`kubectl rollout restart` 手动滚动重启服务。

- **注入方式**：Deployment中通过 `envFrom.configMapRef` 全局注入所有键值对，无需逐个配置环境变量。

## 7\.4 Deployment 滚动更新与探针、存储深度配置（生产高可用核心）

### CICD动态镜像替换命令

```bash
# 动态替换镜像版本为GitHub提交哈希，保证版本唯一可追溯
sed -i "s|image:.*|image: ${{ secrets.TENCENT_REGISTRY_GATEWAY }}:${{ github.sha }}|g" gateway-deployment.yaml
kubectl apply -f gateway-deployment.yaml
```

### 核心YAML配置（gateway\-deployment\.yaml）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gateway-deployment
  namespace: gateway
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
  selector:
    matchLabels:
      app: gateway-svc
  template:
    metadata:
      labels:
        app: gateway-svc
    spec:
      imagePullSecrets:
      - name: tcr-secret
      containers:
      - name: gateway-api
        image: PLACEHOLDER_IMAGE
        envFrom:
        - configMapRef:
            name: gateway-config
        volumeMounts:
        - name: keys-volume
          mountPath: /app/keys
        livenessProbe:
          httpGet:
            path: /gateway/health
            port: 5040
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /gateway/health
            port: 5040
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: keys-volume
        persistentVolumeClaim:
          claimName: shared-keys-pvc
```

### 深度技术剖析与应用场景

- **单副本滚动更新适配**：单副本场景下 `maxSurge:0、maxUnavailable:1` 是最优配置，禁止额外扩容Pod，允许短暂停服更新，彻底解决单副本端口占用冲突问题。

- **双探针高可用机制**：存活探针异常直接重启容器，解决服务卡死、假死问题；就绪探针异常将Pod从Service流量池摘除，避免转发流量到未就绪服务。

- **统一健康检查规范**：与Consul服务发现共用 `/gateway/health` 接口，保证集群探针检测、服务注册健康状态一致性。

- **跨命名空间共享存储**：通过PVC挂载NFS共享目录，实现gateway、identity等多命名空间服务共用密钥文件，无需重复拷贝配置。

- **版本追溯方案**：使用Git Commit哈希作为镜像标签，每一次构建版本唯一，支持精准回滚、问题溯源。

## 7\.5 NodePort Service 服务暴露与集群网络

### 核心命令

```bash
kubectl apply -f gateway-service.yaml
```

### 完整YAML配置（gateway\-service\.yaml）

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gateway-svc
  namespace: gateway
spec:
  selector:
    app: gateway-svc
  ports:
    - port: 5040
      targetPort: 5040
      nodePort: 30040
  type: NodePort
```

### 深度技术剖析与应用场景

- **端口映射规则**：`port`为集群内部Service虚拟端口，供集群内其他服务调用；`targetPort`为容器业务端口；`nodePort`为节点外网/内网暴露端口。

- **NodePort生产价值**：腾讯云K3s轻量集群首选，无需负载均衡费用，每个节点统一开放30040端口，内网所有节点均可访问网关服务。

- **服务发现原理**：通过labels标签精准匹配Deployment Pod，kube\-proxy自动实现负载均衡，屏蔽Pod动态IP变化。

- **迭代方案**：生产后期可平滑迁移为Ingress统一域名入口，替代多NodePort端口混乱问题。

## 7\.6 CICD部署校验：滚动更新状态阻塞检查

### 核心命令

```bash
kubectl rollout status deployment/gateway-deployment -n gateway --timeout=3m
```

### 深度技术剖析与应用场景

- **CICD核心作用**：阻塞式等待滚动更新完成，超时3分钟自动退出，部署失败则流水线返回非零码，自动终止流程并告警。

- **解决的问题**：避免流水线执行完成但服务实际未启动、探针未就绪，实现**真正意义上的部署成功校验**。

- **生产适配**：适配网络波动、镜像拉取慢、容器启动超时等异常场景，提升CICD流水线稳定性。

## 7\.7 NFS\+PV/PVC 跨命名空间共享存储（生产密钥共享核心方案）

### 核心命令

```bash
# 创建共享PV
kubectl apply -f shared-keys-pv.yaml
# 多命名空间绑定同一PV
kubectl apply -f shared-keys-pvc-gateway.yaml
kubectl apply -f shared-keys-pvc-identity.yaml
```

### 核心YAML配置

shared\-keys\-pv\.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: shared-keys-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-path
  nfs:
    path: /data/shared-keys
    server: 10.0.0.15
```

shared\-keys\-pvc\-gateway\.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-keys-pvc
  namespace: gateway
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  volumeName: shared-keys-pv
```

### 深度技术剖析与应用场景

- **ReadWriteMany核心价值**：支持多节点、多命名空间、多Pod同时挂载读写，完美适配密钥文件全局共享场景，普通存储仅支持单Pod挂载。

- **强制绑定PV**：通过 `volumeName` 固定绑定指定PV，避免集群动态供给存储错乱，保证多命名空间PVC指向同一NFS目录。

- **Retain回收策略**：PVC删除后PV数据保留，防止误删操作导致密钥文件丢失，生产数据安全必备。

> （注：部分内容可能由 AI 生成）
