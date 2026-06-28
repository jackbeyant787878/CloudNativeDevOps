# IdentityService 完整 CICD\+K3s 部署全流程深度剖析（逐步骤、逐技术点拆解）

本文基于你提供的全套生产资源：**GitHub Actions 流水线、Namespace、PVC、Deployment、Service、Secret 动态创建、TCR 私有镜像拉取、NFS 密钥挂载、双探针健康检测、单副本定时任务适配架构**，进行**逐行、分步、全技术点拆解**。

适配环境：**腾讯云 K3s（轻量 K8s）**

业务场景：**Identity 认证服务 \+ Hangfire 定时任务（禁止多副本并发）**

核心亮点：**无落地明文密钥、幂等部署、镜像版本可追溯、滚动更新零误删、密钥目录权限适配、集群内外网络隔离**

# 整体架构总览（先看懂整套流程）

## 1\.1 流水线整体链路

手动触发 GitHub Actions → 代码检出 → 构建多架构兼容镜像 → 推送腾讯云 TCR 私有仓库 → SCP 传输 K8s 资源清单到 K3s 服务器 → SSH 远程执行部署脚本 → 命名空间初始化 → 镜像凭据创建 → 数据库/Redis 敏感密钥动态注入 → 配置文件、存储挂载 → 动态替换镜像版本 → 滚动更新部署 → 阻塞校验部署成功 → 输出集群状态。

## 1\.2 整套部署用到的核心技术栈

- CI：GitHub Actions 流水线、Buildx 镜像构建、TCR 腾讯云私有镜像仓库

- CD：SCP 资源传输、SSH 远程脚本部署、Kubectl 声明式\+命令式混合运维

- K8s 资源：Namespace、Secret（镜像凭据/业务密钥）、ConfigMap、PVC、Deployment、Service\(NodePort\)

- 稳定性技术：Liveness/Readiness 双探针、滚动更新策略、资源配额、文件权限绑定、幂等创建、失败立即终止机制

- 存储技术：K3s local\-path 存储类、共享密钥 PVC 挂载、跨服务密钥目录统一管理

- 安全技术：密钥不落地、动态临时创建、私有镜像鉴权、环境变量注入敏感配置

# 第一阶段：GitHub Actions CI 镜像构建流程（build\-and\-push 任务）

本阶段核心目标：**编译代码、生成标准镜像、打上双版本标签、推送到腾讯云 TCR，解决镜像版本追溯与私有拉取问题**。

## 1\.1 流水线触发规则深度剖析

```yaml
on:
  workflow_dispatch:   # 仅手动触发，不会在 push 时自动运行
```

- **技术点**：workflow\_dispatch 手动触发事件

- **生产设计意义**：认证服务属于核心基础服务，不允许代码提交自动发布，避免误提交导致线上认证服务故障，严控发布权限，符合运维安全规范。

## 1\.2 核心步骤逐行剖析

### 步骤1：代码检出

uses: actions/checkout@v4

拉取当前仓库完整代码，包含 Dockerfile、所有 K8s YAML 资源清单，为后续构建和部署做文件准备。

### 步骤2：启用 Buildx 构建器

uses: docker/setup\-buildx\-action@v3

- **技术点**：Docker Buildx 跨架构构建

- **作用**：兼容 arm/amd64 架构，适配腾讯云不同架构服务器，保证镜像通用性。

### 步骤3：登录腾讯云 TCR 私有仓库

uses: docker/login\-action@v3

- **核心技术点**：基于 GitHub Secrets 动态读取账号密码，**全程无明文落地、无硬编码**

- **适配地址**：ccr\.ccs\.tencentyun\.com 腾讯云官方容器镜像仓库

- **生产价值**：私有镜像必须鉴权登录，否则无法推送和拉取，是私有化部署的前置核心步骤。

### 步骤4：构建并推送镜像（核心优化点）

```yaml
provenance: false
sbom: false
```

- **关键踩坑修复（核心技术点）**：关闭镜像物料清单、成果证明生成

- **问题根因**：Docker 默认开启该功能会生成多架构元数据，腾讯云 TCR 对新型元数据兼容差，会导致**镜像推送假死、上传成功但无法拉取**

- **版本标签策略**：同时打 latest（默认最新）\+ github\.sha（提交哈希唯一版本）

- **生产意义**：latest 用于快速部署，git sha 用于**版本唯一追溯、精准回滚、故障定位**，解决版本混乱问题。

# 第二阶段：GitHub Actions CD 远程部署流程（deploy 任务）

依赖 build\-and\-push 执行成功后运行，核心：**传输 K8s 清单 → 远程服务器执行 kubectl 部署 → 全流程幂等、失败即终止**

## 2\.1 核心前置机制：set \-e

```bash
set -e
```

- **技术点**：Shell 严格报错模式

- **生产价值**：流水线任意一步命令失败（密钥创建失败、镜像拉取失败、资源创建失败），**立即终止流水线，不继续执行后续部署**，避免半部署状态导致集群资源错乱。

## 2\.2 文件传输机制

通过 scp\-action 将本地 k8s/ 目录所有 YAML 清单传输到 K3s 服务器 /tmp/auth\-deploy/ 临时目录，不占用业务目录，部署后可自动清理，保证服务器目录整洁。

# 第三阶段：K3s 集群逐行部署命令 \& 技术点深度剖析（核心）

## 3\.1 步骤1：创建独立业务命名空间 identity

```bash
kubectl apply -f auth-namespace.yaml
```

对应资源：独立 identity 命名空间，专门存放认证服务所有资源

- **技术点1：资源隔离**：将认证服务与网关、业务服务完全隔离，避免资源名称冲突、配置污染、权限越界

- **技术点2：幂等执行**：apply 命令可重复执行，已存在则不改动，适配流水线多次重跑

## 3\.2 步骤2：动态创建 TCR 私有镜像拉取凭据

```bash
kubectl delete secret tcr-secret -n identity --ignore-not-found
kubectl create secret docker-registry tcr-secret -n identity ...
```

- **技术点1：镜像凭据类型**：docker\-registry 专属 Secret 类型，专门用于容器镜像仓库鉴权

- **技术点2：平滑更新**：先删除再创建，避免旧凭据失效导致镜像拉取失败

- **技术点3：\-\-ignore\-not\-found**：首次部署无旧密钥不报错，保证流水线幂等

- **生产作用**：让 K3s 集群有权限拉取你腾讯云 TCR 的私有镜像，彻底解决 ImagePullBackOff 错误

## 3\.3 步骤3：动态创建业务敏感密钥 Secret（核心安全设计）

```bash
kubectl create secret generic auth-secrets -n identity \
--from-literal=xxx=xxx --dry-run=client -o yaml | kubectl apply -f -
```

存储内容：SQLServer 数据库连接字符串、Hangfire 任务数据库、Redis 配置（全部核心敏感数据）

- **核心高阶技术点：dry\-run 流水线无落地创建**

- 不直接 create，而是先空跑生成 YAML 再 apply，**彻底避免密钥重复创建报错**，同时保证敏感数据不落地、不留日志

- **统一安全规范（生产强制标准）**：所有敏感信息**禁止硬编码、禁止写入YAML清单、禁止服务器明文落地**。数据库连接串、Redis密钥、TCR镜像仓库账号密码、服务鉴权密钥等核心隐私数据，全部在「GitHub Actions secrets and variables」中统一创建、集中管理，流水线动态读取、临时注入，全程无留存、可统一轮换、权限可控。

- **使用方式**：Deployment 通过 secretRef 引用，以环境变量注入容器

## 3\.4 步骤4：应用配置、镜像密钥与存储资源

```bash
kubectl apply -f auth-config.yaml
kubectl apply -f auth-keys-pvc.yaml
```

### 3\.4\.1 核心配置资源：auth\-config 完整解析（业务核心配置）

本次新增 **auth\-config\.txt** 为认证服务专属 ConfigMap，存储所有非敏感业务参数、JWT 令牌策略、密钥生命周期、Consul服务注册配置，是服务启动的核心配置依赖，完整YAML如下：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: auth-config
  namespace: identity
data:
  ASPNETCORE_ENVIRONMENT: "Production"
  ASPNETCORE_URLS: "http://+:5008"
  OPENIDDICTSETTINGS__ACCESSTOKENLIFETIMEMINUTES: "60"
  OPENIDDICTSETTINGS__REFRESHTOKENLIFETIMEDAYS: "14"
  OPENIDDICT__DISABLEENCRYPTION: "true"
  KEYSETTINGS__BASEDIRECTORY: "/app/keys"
  KEYSETTINGS__SIGNINGKEYLIFETIMEDAYS: "90"
  KEYSETTINGS__ENCRYPTIONKEYLIFETIMEDAYS: "180"
  OPENIDDICTSETTINGS__ISSUER: "http://auth-svc.identity.svc.cluster.local:5008"
  CONSULCONFIG__SERVICENAME: "IdentityServiceCenter"
  CONSULCONFIG__SERVICEIP: "auth-svc.identity.svc.cluster.local"
  CONSULCONFIG__REGISTRYADDRESS: "http://consul-server.consul.svc.cluster.local:8500"
```

#### 深度技术剖析 \& 生产场景

- **环境与端口绑定**：指定生产环境、服务监听所有网卡5008端口，与Deployment容器端口、Service映射端口完全统一，保证端口一致性无冲突。

- **JWT令牌生命周期管控**：精准配置AccessToken60分钟有效期、RefreshToken14天刷新周期、签名密钥90天轮换、加密密钥180天轮换，标准化生产安全策略，避免密钥长期不更新引发安全风险。

- **密钥目录统一绑定**：固定密钥读取路径 `/app/keys`，与PVC挂载路径完全匹配，实现密钥文件统一读取，适配全局共享密钥存储架构。

- **集群内部域名核心配置（解决报错疑惑）**：
          
配置中 `auth-svc.identity.svc.cluster.local:5008`、`consul-server.consul.svc.cluster.local:8500` 均为**K3s集群CoreDNS内部域名**。
✅ 正常访问场景：集群内所有Pod、节点可正常解析、调用
          
❌ 报错根因：本地浏览器、外网机器无法解析集群内部域名，因此出现「网页解析失败」，**并非服务故障，属于正常集群网络隔离机制**

- **Consul服务注册自动化**：预设Consul注册地址、服务名、服务内网域名，服务启动后自动注册到Consul注册中心，实现网关与认证服务的自动发现与调用。

- **\.NET配置适配规则**：沿用双下划线 `__` 层级解析规则，完美适配\.NET配置体系，无需硬编码配置文件，全环境变量注入。

### 3\.4\.2 镜像密钥资源：tcr\-secret 完整解析（私有镜像拉取核心）

本次新增 **tcr\-secret\.txt** 为K3s标准私有镜像仓库凭据资源，与流水线动态创建的Secret完全对应，是集群拉取腾讯云TCR私有镜像的核心权限凭证，完整YAML如下：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tcr-secret
  namespace: identity
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: PLACEHOLDER_BASE64
```

#### 深度技术剖析 \& 生产场景

- **标准类型约束**：固定类型 `kubernetes.io/dockerconfigjson`，为K8s专属镜像仓库认证类型，不可替换为通用Secret，否则容器运行时无法识别鉴权信息。

- **全链路安全闭环**：**\.dockerconfigjson** 加密内容、TCR登录账号密码均取自 GitHub Secrets，流水线实时加密生成，不落地任何明文配置；所有敏感变量统一在 GitHub 后台集中维护，支持一键更新、密钥轮换，无需修改业务代码与部署清单。

- **命名空间隔离**：精准绑定identity命名空间，仅认证服务可引用，其他服务无法复用，最小权限保证镜像凭据安全。

- **与流水线联动逻辑**：流水线采用「删除重建」机制更新该Secret，保证TCR账号密码变更后，集群凭据实时同步，杜绝镜像拉取权限失效问题。

- **资源引用方式**：Deployment中通过 `imagePullSecrets.name` 引用，全局生效于当前Pod所有容器，自动携带凭据拉取私有镜像。

### 3\.4\.3 存储资源：PVC挂载能力说明

- **ConfigMap与存储联动**：ConfigMap指定密钥读取路径，PVC实现目录持久化挂载，二者配合实现「配置定义路径\+存储落地文件」的完整密钥读取链路。

- **生产价值**：配置、密钥、存储完全解耦，支持单独更新配置、轮换密钥、扩容存储，无需重构服务部署逻辑。

```bash
kubectl apply -f auth-config.yaml
kubectl apply -f auth-keys-pvc.yaml
```

- **ConfigMap（auth\-config）**：存储认证服务非敏感业务配置、JWT令牌策略、Consul注册参数、密钥生命周期，以环境变量注入容器，配置与镜像完全解耦，外网无法解析的集群域名属于正常网络隔离机制，不影响集群内服务调用。

- **Secret（tcr\-secret）**：标准TCR私有镜像鉴权凭据，Base64加密存储，命名空间隔离，为服务镜像拉取提供权限支撑。

- **PVC 存储挂载**：基于 K3s 默认 local\-path 存储类，申请 1Gi 存储空间，用于挂载密钥证书目录 /app/keys，适配ConfigMap密钥读取路径

## 3\.5 步骤5：动态替换镜像版本 \+ 部署资源

```bash
sed -i "s|image:.*|image: 镜像地址:gitsha|g" auth-deployment.yaml
kubectl apply -f auth-deployment.yaml
kubectl apply -f auth-service.yaml
```

- **技术点**：流水线动态替换占位镜像，每次部署使用最新 Git Commit 哈希版本

- **价值**：每一次部署版本唯一，可精准回滚、故障溯源，杜绝 latest 版本覆盖无法追溯的问题

## 3\.6 步骤6：滚动更新阻塞校验（流水线成败关键）

```bash
kubectl rollout status deployment/auth-deployment -n identity --timeout=3m
```

- **核心技术点**：阻塞式等待 Pod 启动就绪、滚动更新完成

- **生产作用**：不等更新完成流水线不结束，启动失败、探针异常、镜像拉取失败直接让流水线报错，避免虚假部署成功

# 第四阶段：Deployment 部署清单深度技术剖析（核心生产设计）

## 4\.1 单副本架构设计（业务专属优化）

```yaml
replicas: 1
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 0
    maxUnavailable: 1
```

- **业务场景适配**：服务内置 Hangfire 定时任务，**绝对禁止多副本并发执行**，否则会导致定时任务重复执行、数据错乱

- **滚动更新策略原理**：
        
maxSurge: 0 不允许临时扩容新 Pod
        
maxUnavailable: 1 允许旧 Pod 先销毁再启动新 Pod
      

- **生产取舍**：为保证任务唯一性，牺牲瞬时高可用，是业务驱动的最优架构选择

## 4\.2 安全权限配置 fsGroup

```yaml
securityContext:
  fsGroup: 1654
```

- **关键技术点**：解决容器挂载目录权限不足问题

- **问题场景**：NFS/本地存储挂载到容器后，容器内部用户无读写权限，导致密钥读取失败、服务启动报错

- **作用**：K3s 自动将挂载目录文件权限递归修改为 1654 组，保证容器正常读写 /app/keys 密钥目录

## 4\.3 镜像拉取策略

```yaml
imagePullPolicy: Always
```

每次重启/更新都强制拉取最新镜像，适配你 GitSHA 动态版本机制，避免节点缓存旧镜像导致部署不生效。

## 4\.4 资源配额限制（防雪崩机制）

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

- **requests**：集群调度预留资源，保证服务基础运行资源，避免被节点抢占

- **limits**：封顶资源，防止认证服务内存泄漏、CPU 打满导致整个 K3s 集群节点雪崩

## 4\.5 双探针高可用机制

- **存活探针 LivenessProbe**：15s 检测一次，服务卡死、假死、无响应自动重启 Pod，实现自愈

- **就绪探针 ReadinessProbe**：10s 检测一次，服务未初始化完成时，**不接入流量**，避免启动中报错

- **统一健康接口**：基于 /health 标准端点，适配集群检测与服务注册

# 第五阶段：Service 网络暴露技术剖析

```yaml
type: NodePort
ports:
  - port: 5008
    targetPort: 5008
    nodePort: 30008
```

- **port**：集群内部 Service 虚拟端口，供网关、集群内微服务内部调用

- **targetPort**：容器真实业务端口

- **nodePort**：固定节点外网端口 30008，所有集群节点均可通过该端口访问认证服务

- **适配 K3s**：轻量集群首选 NodePort，无需负载均衡费用，部署简单、访问稳定

# 第六阶段：集群域名解析报错专项答疑 \+ 整套 CICD \+ K3s 部署核心总结

## 6\.1 集群域名网页解析失败报错终极解答（对应实测报错）

报错链接：`http://auth-svc.identity.svc.cluster.local:5008`、`http://consul-server.consul.svc.cluster.local:8500`

**报错根因（非服务故障）**：此类 `xxx.svc.cluster.local` 后缀域名是 **K3s集群内部CoreDNS专属解析域名**，仅集群节点、集群内Pod可解析访问，用于微服务内部相互调用，**不支持外网、本地浏览器直接访问**，因此网页端会提示解析失败，属于K8s标准网络隔离机制。

**正确访问方式**：

- 内网服务调用：网关、集群内微服务直接使用该内部域名通信（稳定、高效）

- 外网/本地测试：通过服务 NodePort 地址 `服务器IP:30008` 访问认证服务

- Consul访问：登录集群节点/进入Pod，通过内部域名访问8500端口，无需外网暴露

## 6\.2 整套 CICD \+ K3s 部署核心总结（生产核心优势）

1. **企业级安全闭环**：严格遵循敏感信息管控规范，数据库连接串、Redis密钥、镜像仓库凭据等所有隐私数据，统一在 GitHub Actions secrets and variables 配置管理，流水线动态注入、零明文落地、无硬编码、支持统一密钥轮换，完全贴合生产安全合规标准。

2. **版本可追溯**：GitSHA 唯一镜像版本，解决迭代混乱、无法回滚问题

3. **业务架构适配**：针对 Hangfire 定时任务定制单副本滚动更新策略，杜绝任务重复执行

4. **部署高可靠**：set\-e 严格报错、rollout 阻塞校验、双探针自愈，杜绝虚假部署成功

5. **权限与存储适配**：fsGroup 解决挂载权限问题，共享 PVC 统一密钥管理

6. **腾讯云生态适配**：修复 TCR 镜像推送兼容问题，完美适配 K3s 轻量集群特性

> （注：部分内容可能由 AI 生成）
