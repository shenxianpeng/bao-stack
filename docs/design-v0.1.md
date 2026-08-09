# bao-stack v0.1 设计文档

> 工作代号 bao-stack(最终命名待定,见 §12)。
> 定位一句话:**OpenBao 的开箱即用生产发行版** —— bare-metal / VM 优先、一条命令拉起 HA 集群 + 备份 + 监控 + IaC,对标 Pigsty 之于 PostgreSQL。
>
> 文档约定:标注 **[已验证]** 的内容有公开来源支撑;标注 **[假设]** 的内容是设计判断,需在实验室阶段或社区中核实。

---

## 1. 定位与设计原则

**产品定位。** OpenBao 内核能打,但生产化运维层是公认空白:官方 Helm chart 明言"不负责监控、备份、升级",HA 集群的 init/unseal/join 全靠手动,且现有 8 家商业公司卖的都是人天服务而非可交付工件 **[已验证]**。bao-stack 填补的就是这个"工件"位置:不做内核、不改内核字节,把部署、高可用、备份恢复、监控告警、证书管理、升级做成幂等的声明式交付物。

**六条设计原则:**

1. **Bare-metal / VM 优先,不依赖 Docker 与 Kubernetes。** K8s 路线已有官方 Helm chart 在演进,且主权自托管客户中相当比例不上 K8s——这正是 Pigsty "no Docker no K8s" 定位验证过的市场 **[已验证:Pigsty 官方定位]**。
2. **声明式 + 幂等。** 单一 YAML inventory 描述期望状态,Ansible playbook 负责收敛;重复执行安全。
3. **离线可安装(air-gap ready)。** 主权与合规场景的硬需求:本地软件仓库先行,安装过程零外网依赖。**[假设:air-gap 是欧洲主权客户的普遍要求,需在社区访谈中确认优先级]**
4. **上游优先。** 凡是通用能力(备份流程、监控指标、声明式初始化)优先贡献进 OpenBao 上游;发行版只保留"编排与集成"这一层。上游 2025-2026 路线图第一大类就是 Operator Experience(备份/恢复、监控改进、break-glass、profiles)**[已验证]**,与本项目天然分工。
5. **不碰密钥内容。** 发行版管理的是 OpenBao 进程与集群生命周期,永不接触、记录或代管用户的 secret 数据;备份产物默认加密。
6. **每个组件可独立采用。** 模块化设计(见 §4),用户可以只用监控大盘、只用备份 playbook,降低采用门槛——这也是 Pigsty 的模块化经验 **[已验证:Pigsty README]**。

## 2. 目标用户与场景

| 画像 | 驱动力 | 依据 |
|---|---|---|
| 欧洲主权自托管组织 | 数字主权、数据驻留;OpenBao 商业线索约 75% 来自美国以外 | [已验证:TechTarget 报道 ControlPlane 说法] |
| Vault BSL / IBM 收购难民 | 许可与厂商锁定顾虑,API 兼容迁移路径存在 | [已验证] |
| CRA 合规压力下的软件厂商 | CRA 报告义务 2026-09 启动,ControlPlane 已以此为卖点 | [已验证] |
| 不上 K8s 的传统企业 / 政企 | 官方 Helm 之外没有生产级交付物 | [已验证:官方文档] |
| AI/自动化重度用户 | 机器凭证数量随 agent 化膨胀 | [已验证:行业报道] |

**非目标用户(v0.1):** 已深度 K8s 化的团队(指向官方 Helm)、需要跨区 DR 复制的超大规模用户(上游 2026-2027 路线图项)。

## 3. 与 Pigsty 逐项对照

Pigsty 的组件构成为本项目提供参考系 **[已验证:Pigsty 官网组件清单]**。对照的意义:凡 Pigsty 有而我们无的,判断"是否 OpenBao 场景需要";凡结构性差异,说明原因。

| 能力 | Pigsty(PostgreSQL) | bao-stack(OpenBao) | 差异说明 |
|---|---|---|---|
| 内核 | PostgreSQL + 12 种可换内核 | OpenBao(单内核) | v0.1 不做内核变体 |
| 共识/HA | Patroni + etcd 外置共识 | **Raft 集成存储**,内核自带共识 | 结构性简化:无需外置 etcd,这是 OpenBao 优势 |
| 连接池 | PgBouncer | 无需 | 密钥服务无连接池概念 |
| 负载均衡/接入 | HAProxy + VIP | HAProxy(基于健康检查路由 active 节点)+ 可选 keepalived VIP | 语义不同:OpenBao 写请求只能到 active 节点,LB 规则据此设计 |
| 备份/PITR | pgBackRest,支持 PITR | Raft snapshot 定时 + 加密 + 异地存储 + 恢复演练 playbook | 无 WAL 概念,快照粒度恢复;PITR 不适用 |
| 监控 | Victoria 全家桶 + Grafana,3000+ 指标 | Prometheus 兼容 telemetry + Grafana 大盘 + 告警规则集 | 指标面小得多,但 seal 状态/租约/证书过期是密钥场景特有告警 |
| IaC | Ansible 幂等 playbook | 同 | 直接继承方法论 |
| 本地软件仓库 | Nginx + 本地 YUM/APT repo | 同(air-gap 需求) | 直接继承 |
| CA/证书 | 自签 CA | 自签 CA 引导 + 证书轮换 playbook | OpenBao 场景 TLS 是硬性项而非可选项 |
| 扩展生态 | 500+ PG 扩展打包 | 插件生态尚小,v0.1 不做 | 上游插件注册表在规划中,跟踪即可 |
| Web 门户 | Nginx 统一入口 + 各类 UI | v0.1 仅 Grafana | 避免过早做壳 |
| 收费模式 | 明码标价订阅:大版本全周期支持、专家时长、SLA | 同构(见 §11 闸门三) | [已验证:Pigsty 定价页] |

**对照结论:** OpenBao 场景比 PG 场景**结构更简单**(自带共识、无连接池、无 PITR),这对单人项目是利好——Pigsty 需要驯服十几个组件,bao-stack v0.1 只需驯服六七个。

## 4. 架构总览

沿用 Pigsty 的模块化划分,三个模块:

```
┌─────────────────────────────────────────────────┐
│ INFRA(每套部署 1 份,可复用已有设施)           │
│  · 监控栈:Prometheus 兼容 TSDB + Grafana        │
│    + Alertmanager                               │
│  · 本地软件仓库(air-gap 安装源)                │
│  · 部署级自签 CA                                 │
├─────────────────────────────────────────────────┤
│ NODE(每台主机 1 份)                            │
│  · node_exporter、时间同步校验、日志收集         │
│  · 系统调优(文件句柄、内存锁定 mlock 等)       │
├─────────────────────────────────────────────────┤
│ BAO(集群模块,3 或 5 节点)                     │
│  · OpenBao server(raft 集成存储)               │
│  · HAProxy 接入层(health 检查路由 active)      │
│  · 快照备份定时任务 + 恢复流程                   │
│  · audit device 落盘 + 采集                      │
└─────────────────────────────────────────────────┘
```

关键架构决策与理由:

**D1 — Raft 集成存储为唯一 v0.1 存储后端。** 消除外部依赖(无 etcd/Consul/数据库),三节点即成 HA。其他存储后端(如 PostgreSQL)的支持状态待核实后再议 **[假设:raft 是上游主推路径——高置信,但存储后端全景需查官方文档确认]**。

**D2 — 接入层用 HAProxy 而非 DNS 轮询。** OpenBao 的写路径只在 active 节点,standby 返回特定健康状态码;HAProxy 基于 `/v1/sys/health` 做主备感知路由是 Vault 时代验证过的成熟模式 **[假设:具体状态码语义在 OpenBao 中不变——高置信,实验室第一周验证]**。

**D3 — Unseal 策略双轨。** 默认 Shamir 分片 + 书面化 runbook(含节点重启后的人工 unseal 演练);auto-unseal(KMS / PKCS#11 等)作为可选配置项。注意主权矛盾:依赖美国云 KMS 的 auto-unseal 会破坏主权卖点,优先调研 PKCS#11 / softhsm 路线 **[假设:OpenBao 的 auto-unseal 机制种类与成熟度需实验室核实]**。

**D4 — 备份三件套。** 定时 raft snapshot → 客户端加密 → 推送到本地路径或 S3 兼容存储(Garage/SeaweedFS 友好,呼应主权场景);恢复不只给 playbook,还给**演练脚本**——"没演练过的备份等于没有备份"作为产品立场。

**D5 — 监控告警面向密钥场景特化。** 大盘与告警不做通用抄写,聚焦密钥服务的生死指标:seal 状态突变、leader 丢失、raft peer 异常、快照年龄超限、TLS 证书临期、token/lease 异常增长、audit device 写入失败(此项会阻塞请求)。

## 5. 组件选型

| 位置 | 选型 | 理由 | 备选与取舍 |
|---|---|---|---|
| 配置管理 | Ansible | Pigsty 验证过的方法论;目标客户群(政企、传统企业)接受度最高 | Pulumi/Terraform 更现代但受众窄;不排除后期提供 TF module 外壳 |
| TSDB | VictoriaMetrics 单机版 | 资源占用低、兼容 Prometheus 协议;Pigsty 同款路线 | Prometheus 原生更"官方",作为 profile 可选 |
| 大盘 | Grafana | 无争议 | — |
| 告警 | vmalert / Alertmanager | 与 TSDB 选型联动 | — |
| LB | HAProxy | 健康检查路由能力成熟 | Nginx stream 亦可,HAProxy 的 http-check 表达力更强 |
| VIP | keepalived(可选) | 政企网络常见需求 | 云环境用 LB 替代 |
| 日志 | 先落盘 + logrotate,采集器 v0.2 再定 | 避免 v0.1 引入重组件 | Vector/Fluent Bit 候选 |
| 快照存储 | 本地目录 / 任意 S3 兼容端点 | 主权友好 | — |

## 6. 仓库与 Playbook 结构

```
bao-stack/
├── configure              # 交互式生成 inventory(探测网卡/主机,输出 config.yml)
├── install.yml            # 总装 playbook:infra → node → bao 全链
├── infra.yml              # INFRA 模块单独安装
├── node.yml               # NODE 模块单独安装
├── bao.yml                # BAO 集群安装/扩容
├── backup.yml             # 手动触发快照(定时任务由 bao.yml 布置)
├── restore.yml            # 恢复(含 --check 演练模式)
├── upgrade.yml            # 滚动升级:standby 逐台 → leader step-down → 收尾
├── cert.yml               # 证书签发与轮换
├── roles/
│   ├── infra_repo / infra_monitor / infra_ca
│   ├── node_tune / node_exporter
│   └── bao_install / bao_config / bao_ha / bao_backup / bao_lb
├── files/dashboards/      # Grafana 大盘 JSON
├── files/alerts/          # 告警规则集
├── docs/
│   ├── runbook-unseal.md  # 人工 unseal / break-glass 手册
│   ├── runbook-restore.md
│   └── security-model.md  # 明确"发行版不触碰密钥内容"的边界声明
└── ci/                    # 多 OS 矩阵集成测试(见 §8)
```

**声明式配置示意(config.yml):**

```yaml
all:
  children:
    infra:
      hosts: { 10.0.0.10: {} }
    bao-main:
      hosts:
        10.0.0.11: { bao_seq: 1 }
        10.0.0.12: { bao_seq: 2 }
        10.0.0.13: { bao_seq: 3 }
      vars:
        bao_cluster: main
        bao_version: "2.x"          # 锁定的已测版本
        bao_unseal: shamir           # shamir | auto(<provider>)
        bao_snapshot_cron: "0 */6 * * *"
        bao_snapshot_target: s3://backup-bucket/bao/
        bao_tls: internal-ca         # 部署级 CA 自动签发
```

## 7. 与上游的分工映射

原则:**通用能力进上游,编排集成留发行版**。上游正在做声明式初始化与 audit device 声明式配置 **[已验证:OpenBao 博客]**,发行版必须跟踪而非绕过。

| 能力 | 归属 | 动作 |
|---|---|---|
| 声明式 init / audit 配置 | 上游(进行中) | 跟踪 + 测试反馈 + 补边角 PR |
| 备份/恢复流程改进 | 上游路线图 Operator Experience 项 | **首选贡献切入点**:把 D4 中通用部分做成上游 PR |
| 监控指标缺口 | 上游 | 实验室发现缺什么指标就提 issue/PR |
| break-glass 流程 | 上游路线图项 | 参与设计讨论 |
| Ansible 编排、大盘、告警集、runbook | 发行版 | 差异化资产 |
| 多组件集成(LB/CA/TSDB) | 发行版 | 差异化资产 |

这个分工同时是**信任策略**:每个上游 PR 都在为发行版的可信度背书。

## 8. 质量与发布工程

这是你的主场,也是对 8 家人天服务商的差异化核心:

发布流程按"每个 OpenBao 新版本 → CI 矩阵全量回归 → 出具兼容性声明 → 锁版本发布"运转。CI 矩阵覆盖主流 OS(Debian/Ubuntu/RHEL 系起步)× 集群形态(单机/3 节点)× 关键场景(全新安装、扩容、备份恢复演练、滚动升级、节点故障注入)。所有产物签名 + SBOM——借鉴 Chainguard/Astral 索引的发布纪律(不可变工件、先完成收集再替换索引、防半截发布)**[已验证:astral-sh-build/_build-index README 的发布流程设计]**。"经过测试的升级路径"本身就是订阅卖点的一半。

## 9. v0.1 非目标

Kubernetes 部署(官方 Helm 领域);DR/跨区复制(上游 2026-2027 路线图,未 GA 前不承诺);多集群联邦;Web 管理控制台;插件打包生态;Windows。每一项都写进 README 的 Non-goals,防止范围蔓延。

## 10. 安全模型声明(信任生意的地基)

发行版永不读取、缓存、转发 secret 明文;备份产物默认加密且密钥归属用户;所有 playbook 可审计(无编译黑盒);不引入任何 phone-home;发布产物可复现构建(长期目标)。这一节将来扩成独立的 `security-model.md`,是给潜在客户安全团队看的第一份文件。

## 11. 里程碑与验证闸门

| 时间 | 里程碑 | 闸门(不过则重估) |
|---|---|---|
| 2026-08 ~ 10 | 实验室手动跑通全生命周期;Zulip/周四社区会潜水;scm-bench 收尾 | 产出完整踩坑笔记(= playbook 需求规格) |
| 2026-11 ~ 2027-01 | 首批上游 PR(备份/监控方向);发布生产化系列文章 | **闸门一**:成为上游可见贡献者;文章有真实提问/阅读信号 |
| 2027-Q1 | bao-stack v0.1 公开发布(install + HA + 备份 + 监控 + 升级) | **闸门二**:MVP 一条命令可拉起并通过 CI 矩阵 |
| 2027-Q2 ~ Q3 | 找 design partner | **闸门三**:3 个真实生产部署,≥1 个愿谈年费 |

## 12. 开放问题清单

按"下一步动作"排序:

1. **存储后端全景**:raft 之外上游还支持/计划什么?(查官方文档 + 问社区)
2. **auto-unseal 机制清单与主权友好选项**(PKCS#11 成熟度)——实验室第 1-2 周核实
3. **health 端点状态码语义**与 HAProxy 检查写法——实验室第 1 周核实
4. **命名与商标**:OpenBao 为 LF 商标;候选名 bao-stack / baodock / 其他,进社区混熟后在周四会或 GitHub discussion 公开征询,让项目以社区知情方式诞生
5. **许可选择**:Pigsty 用 AGPLv3(防云厂商白嫖)**[已验证]**;Apache-2.0 更利传播。倾向 AGPLv3 + 商业订阅双轨,但留到闸门一之后决定
6. **与 8 家支持商的关系**:目标是"它们共同部署的工件"。何时、以什么方式接触 Adfinis / ControlPlane 谈合作,闸门一后评估
7. **snapshot 一致性语义**(活跃写入下快照的一致性保证)——实验室核实并写进 runbook
