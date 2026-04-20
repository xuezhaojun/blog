---
title: "Red Hat 项目经历"
layout: "single"
url: "/resume/"
summary: "Red Hat 项目经历详述"
ShowToc: false
draft: false
hideMeta: true
---


### 集群管理

**[Cluster-Proxy](https://github.com/open-cluster-management-io/cluster-proxy) 核心负责人** — [106 merged PRs](https://github.com/open-cluster-management-io/cluster-proxy/pulls?q=is%3Apr+is%3Amerged+author%3Axuezhaojun)

- 在 OCM 的 Hub-Spoke 架构下，通过反向隧道使用户可以从 Hub 侧直接访问 Agent 侧的目标 Service（包括 KubeAPI Server），解决 Pull Mode 下 Hub 无法主动连接被管集群的核心问题
- ACM 内部 Console、Application、Observability 等核心组件均依赖此组件作为 Hub→Spoke 访问的基础网络层

**[Import Controller](https://github.com/stolostron/managedcluster-import-controller) 核心开发者** — [183 merged PRs](https://github.com/stolostron/managedcluster-import-controller/pulls?q=is%3Apr+is%3Amerged+author%3Axuezhaojun)

- 从开源 OCM 到企业级 ACM 的关键衔接组件：提供 auto-import 自动接入和跨云多厂商集群集成能力（AWS/Azure/GCP/私有云），使 ACM 具备真实生产环境的可用性

**OCM Core Maintainer** — [1400+ merged PRs, 270+ PR reviews](https://github.com/xuezhaojun)，覆盖 Registration、Workload、Placement、Add-on Framework、SDK 等核心模块。全栈 Go 开发，5 年 CRD + Controller / Operator 模式实践。

- Registration 模块 [Approver](https://github.com/open-cluster-management-io/ocm/blob/d770e1655234d34fd8df03ab5d297a34b5d42ce2/pkg/registration/OWNERS#L4)：负责集群注册与身份认证（CSR 签发、证书自动轮换、Lease 心跳监控），负责该模块的代码审批和质量把关
- Switch Hub：实现集群在 Hub 间的在线迁移，支撑 Global Hub（Hub of Hub）横向扩展，突破单 Hub 集群数量上限

---

### AI + K8s 平台能力

**[KubeOpenCode](https://github.com/kubeopencode/kubeopencode)** — K8s 原生 AI Agent 平台，独立完成全部工作

- **K8s 原生设计**：Agent、Task、CronTask 全部作为 CRD，通过 K8s API 管理 AI 工作负载的生命周期和调度
- 架构设计 + 前后端开发 + 文档网站 + 内外部推广，99% AI 辅助开发
- 被 Distinguished Engineer 主动推动纳入 Red Hat 内部 AI 孵化项目之一

**[Repo-as-Agent](https://xuezhaojun.github.io/posts/repo-as-agent/) 方法论** — 自主提出并落地，Git repo = Agent 本体（身份、技能、知识、工作流全部版本控制），28 个可复用 skill，覆盖 20+ 仓库

**生产使用案例**：
- Tekton image 变更自动处理：与 [konflux-build-catalog](https://github.com/stolostron/konflux-build-catalog) 集成（[workflow L247](https://github.com/stolostron/konflux-build-catalog/blob/8eb3352732e44d44ea9ec2923d50bd2731871a47/.github/workflows/update-tekton-task-bundles.yaml#L247)），检测 → 分析 → 修改 → PR → 合并全自动化
- CronTask 定时任务：每日 Scrum 情况自动分析、每日新 bug 自动分析、每周 bot PR 自动处理

---

### 主动性

**敏捷落地**：主动考取 PSM 认证，推动团队敏捷开发流程落地

**效能提升**：创建 [konflux-build-catalog](https://github.com/stolostron/konflux-build-catalog) 集中化方案，消除 60-70 个 repo 每周数百个重复 Tekton 更新 PR，方案被整个 org 采用

**成本优化**：主动 review 全组 AWS 测试集群配置（存储类型 io1→gp3 降幅 96%、实例类型 m5→t3、按测试场景分层为 HA/Lite cluster — HA 用于高可用场景、Lite 用于常规测试），月费 $5,000 → $2,000，年省 $36K

**可维护性**：主导 Registration/Work/Placement 等多仓库合并为 [Mono Repo](https://github.com/open-cluster-management-io/ocm/issues/128#issuecomment-1552536628)，统一依赖管理和 CI/CD 流程，减少跨仓库维护开销

**文档与社区**：主导 OCM 社区文档网站重构（[PR #429](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/pull/429)，+1,856 / -11,626 行）— 迁移 Google Docsy 主题（K8s/Istio/gRPC 等 CNCF 项目标准选择），砍掉无实际维护的中文文档（48/55 文件仅标题中文），文件数减少 46%，降低社区参与门槛

---

### 早期经历

**爱美购** — Software Engineer（2019 – 2020）跨境电商平台开发，深圳

**共济科技** — Software Engineer（2017 – 2019）数据中心基础设施管理软件，深圳
