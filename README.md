# jzo2oKubernetes 部署项目

家政 O2O 到家服务平台「云岚到家」的 Kubernetes 云原生部署实践，涵盖基础环境与集群搭建、核心中间件容器化、10 个 Spring Boot 微服务全链路交付，以及前后端 VUE3 部署。

## 项目背景

云岚到家业务模块多、角色多、流程长（用户下单 → 派单 → 履约 → 支付 → 营销），单体架构存在代码膨胀、扩展困难、故障扩散等问题。本项目将整套服务拆分为 10 个独立微服务，统一容器化并部署到 Kubernetes 集群，实现故障隔离、独立扩容与自动化交付。

## 技术栈

- **容器运行时**：Docker、Containerd
- **编排**：Kubernetes（kubeadm）、Calico
- **服务发现/配置**：Nacos（外接 MySQL 持久化）
- **存储**：NFS 共享存储 + PV/PVC
- **数据库**：MySQL（主从 + 分库分表 ShardingSphere）
- **缓存**：Redis
- **消息队列**：RabbitMQ
- **搜索引擎**：ElasticSearch
- **数据同步**：Canal（监听 MySQL binlog）
- **任务调度**：XXL-JOB
- **流量治理**：Sentinel（限流/熔断）
- **分布式事务**：Seata
- **前端**：Vue3 + Nginx

## 集群规划

3 台节点（1 master + 2 worker），节点规划如下：

| 节点 | 角色 | 职责 |
|---|---|---|
| master01.itcast.cn | control-plane | 集群管理、镜像构建 |
| worker01.itcast.cn | worker | 应用负载 |
| worker02.itcast.cn | worker | 应用负载 |

## 仓库目录结构

```
jzo2o/
├── README.md                    # 本文件
├── 排障笔记.md                   # 典型故障排查记录（面试复盘重点）
├── jz.yaml                      # 全部 61 个 K8s 资源合集（按部署顺序编号）
└── split/                       # 按组件拆分的独立 YAML（20 个文件）
    ├── mysql.yaml               # Secret + ConfigMap + PV + PVC + StatefulSet + Service
    ├── nacos.yaml               # ConfigMap + PV + PVC + Deployment + Service
    ├── rabbitmq.yaml            # ConfigMap + PV + PVC + Deployment + Service
    ├── es.yaml                  # ConfigMap + PV + PVC + Deployment + Service
    ├── canal.yaml               # ConfigMap + PV + PVC + Service+Deployment
    ├── redis.yaml               # ConfigMap + PV + PVC + Service+Deployment
    ├── xxl-job.yaml             # ConfigMap + Deployment + Service
    ├── sentinel-dashboard.yaml  # Deployment + Service
    ├── jzo2o-gateway.yaml       # 网关 + 占位 Service
    ├── jzo2o-foundations.yaml   # 基础服务
    ├── jzo2o-publics.yaml       # 公共服务
    ├── jzo2o-customer.yaml      # 客户服务
    ├── jzo2o-orders-manager.yaml# 订单管理
    ├── jzo2o-orders-seize.yaml  # 抢单
    ├── jzo2o-orders-dispatch.yaml # 派单
    ├── jzo2o-orders-history.yaml  # 订单历史
    ├── jzo2o-trade.yaml         # 交易
    ├── jzo2o-market.yaml        # 营销
    ├── project-xzb-pc-admin-vue3-java.yaml  # 管理端前端
    └── project-xzb-pc-vue3-java.yaml        # 机构端前端
```

## 部署顺序（依赖关系）

按依赖顺序分阶段上线，顺序错乱会导致服务启动报错：

```
第一阶段：基础组件
  NFS → MySQL → Nacos → RabbitMQ → ElasticSearch → Canal → Redis

第二阶段：登录阶段（微服务前置准备 + 基础服务）
  微服务镜像构建/导入 → Nacos 配置导入 → XXL-JOB / Sentinel
  → foundations → publics → customer → gateway

第三阶段：订单链路
  orders-manager → orders-seize → orders-dispatch → orders-history → trade

第四阶段：营销闭环
  market（依赖 Seata + Canal）

第五阶段：前端
  admin-vue3（管理端）→ vue3（机构端）
```

## 核心亮点

1. **组件容器化**：基于 NFS + PV/PVC 为 MySQL（开 binlog）、Nacos、RabbitMQ、ES、Canal、Redis 提供持久化，ConfigMap 挂载配置、Secret 管理密码。
2. **微服务镜像交付**：为 10 个 Spring Boot 微服务编写统一 Dockerfile 模板，脚本批量构建、`docker save` 导出分发导入 Containerd。
3. **profile 正确切换**：通过 `JAVA_OPTS=-Dspring.profiles.active=prod` + 环境变量注入 Nacos 地址，解决 jar 内置 dev 配置导致注册失败的问题。
4. **全链路打通**：按依赖分三批上线，Service 服务发现 + Nginx 反向代理，打通登录/下单/派单/支付核心链路，10 个微服务全部注册就绪。
5. **流量治理与调度**：部署 XXL-JOB 分布式调度 + Sentinel 限流熔断，支撑任务调度与流量控制。
6. **排障沉淀**：见《排障笔记.md》，覆盖集群、中间件、微服务、订单/营销各阶段的典型故障。

## 常见踩坑速查

| 场景 | 一句话原因 | 详见排障笔记 |
|---|---|---|
| 节点 Not Ready | Calico 未就绪 | §1.1 |
| Nacos 登录找不到用户 | 种子用户没导入 | §2.1 |
| Canal 连 MySQL 失败 | 认证插件不兼容 | §2.4 |
| 微服务注册不进集群 | 没切 prod / 连错 Nacos | §3.1 |
| 前端 Nginx 启动失败 | 网关 Service 不存在 | §3.2 |
| Pod 卡 ImagePullBackOff | 镜像没导入全 | §3.3 |
| 启动报 driver class | Nacos 业务配置为空 | §3.4 |

## 安全提醒

> ⚠️ 本仓库 YAML 中 Secret 含明文密码（MySQL、Redis、Nacos 等），若公开到 GitHub 请务必替换为占位符，避免密钥泄露。

## 如何部署

1. 按 `split/` 目录顺序逐个 `kubectl apply -f` 应用，或直接 `kubectl apply -f jz.yaml` 一次性部署（需注意依赖顺序）。
2. 每个组件部署后 `kubectl get pods -n jzo2o` 确认 Ready 再继续下一阶段。
3. 遇到问题先查《排障笔记.md》对应章节。
