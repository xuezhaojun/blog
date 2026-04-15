---
title: "Repo-as-Agent：用 Git 仓库构建 AI Agent"
date: 2026-04-15
draft: false
tags: ["AI Agent", "Claude Code", "DevOps", "软件工程", "中文"]
summary: "如果一个 AI Agent 的身份、技能和知识全都是 Git 仓库里的文件呢？本文介绍一种我们实践过的模式——不引入任何新框架，只用目录、文件和 Git 来构建生产级 AI Agent。"
ShowToc: true
TocOpen: true
ShowReadingTime: true
---

## Agent 的实体到底是什么？

当人们说"构建一个 AI Agent"的时候，脑海里浮现的通常是：一个框架、一个协议、一个编排器、一个向量数据库、一套插件系统。还没开始干活，基础设施就已经一大堆了。

但如果答案可以非常朴素呢？

**一个 Git 仓库 + 一个 AI 编程助手（Claude Code、Cursor 等）= 一个完整的 Agent。**

不需要新框架，不需要新协议。目录、文件、Git——每个工程师最熟悉的基础工具。仓库*本身*就是 Agent。它的身份、技能、知识、工作流，全部以文件的形式存在，用 Git 做版本管理，和代码一样接受 code review。

## 仓库结构

实际的 Agent 仓库长这样：

```
my-team-agent/
├── CLAUDE.md                  # Agent 的身份与行为规则
├── skills/                    # 20+ 可复用的任务定义
│   ├── bug-analyze/
│   ├── workspace-clone/
│   ├── jira-triage/
│   └── ...
├── workflows/                 # 定时或触发式的多步骤流程
│   ├── daily-bug-triage.md
│   ├── daily-standup-prep.md
│   └── weekly-pr-report.md
├── solutions/                 # "错题本"——已知问题与解决方案
├── repos/                     # 20+ 关联项目（只读引用）
├── team-members/              # 团队花名册与组件归属
├── docs/                      # 参考文档，按需加载
├── build/
│   └── Dockerfile             # 可复现的执行环境
└── deploy/                    # Kubernetes 部署清单
```

根目录下的 `CLAUDE.md` 是入口。AI 助手打开这个仓库时，首先读取这个文件，从中了解自己是谁、能做什么、知识在哪里。后续所有交互都建立在这个上下文之上。

这里面最重要的一点：**一切都可审计**。谁添加了一个技能？什么时候改了工作流？为什么废弃了某个解决方案？`git log` 和 `git blame` 能回答所有这些问题。团队成员审核 Agent 的变更，和审核代码一模一样——通过 Pull Request。

## 跨仓库视野

在真实的企业环境里，几乎没有任务只存在于单个仓库中。一个 bug 可能在 API 网关层暴露，但根因在认证服务里，修复又需要改动共享 SDK。如果 Agent 一次只能看到一个仓库，它根本做不了端到端的问题分析。

`repos/` 目录解决了这个问题。它包含 20+ 个相关项目的浅克隆，Agent 可以查阅但不会修改：

```yaml
# repos/repos.yaml
categories:
  core:                    # 团队直接负责的核心项目
    - api-server
    - auth-service
    - sdk-go
    - cli-tool
  dependencies:            # 有定制修改的上游依赖
    - network-proxy
    - grpc-fork
  build:                   # 构建与 CI 配置
    - ci-config
    - release-pipeline
  cross-team:              # 跨团队协作的组件
    - monitoring-stack
    - notification-service
```

有了这个全局视野，Agent 可以跨仓库追踪调用链、理解依赖关系、分析一个项目的变更对其他项目的影响。它看到的代码全貌，等于一个资深工程师脑中对所有项目的业务全景——只不过它可以在几秒内 `grep` 搜索所有代码。

## Git Worktree：并行任务隔离

当任务是端到端执行的——从分析到编码到测试到创建 PR——每个任务的执行时间都很长。你不可能顺序执行，坐在那等一个任务跑完再给下一个。但如果两个任务同时修改同一个仓库，又会互相冲突。

Git Worktree 是原生的解决方案。从一个 bare clone 出发，可以创建多个工作目录，每个目录在自己的分支上：

```bash
# 任务 1：修复 auth 服务的一个 bug
git worktree add ../workspace/auth-fix-1234 bugfix/auth-1234

# 任务 2：给同一个 auth 服务添加新功能
git worktree add ../workspace/auth-feature-5678 feature/oauth-5678
```

每个任务在自己的目录和分支上操作，互不影响。任务完成、PR 合并后，worktree 自动清理。

这个模式已经内置到 Agent 的工作区管理技能中。Agent 接到任务后，自动创建隔离的 worktree，在里面干活，完成后自动清理。同一个仓库上的多个任务可以真正并行执行。

## 团队与项目知识

光能看代码是不够的。要完成端到端的任务，Agent 还需要理解代码背后的组织和流程。

想象这个场景：Agent 分析了一个 bug，判断它属于 import controller 组件，写了修复代码，创建了 PR，然后……需要在 Slack 上通知对应的负责人。谁负责 import controller？他的 Slack ID 是什么？应该发到哪个频道？

`team-members/` 目录回答了这些问题：

```markdown
## Core Team

| Name       | GitHub       | Email              | Components                        |
|------------|-------------|--------------------|-----------------------------------|
| Alice Chen | alice-c     | alice@example.com  | api-server, sdk-go                |
| Bob Park   | bob-park    | bob@example.com    | auth-service, cluster-proxy       |
| Carol Wu   | carol-wu    | carol@example.com  | import-controller, lifecycle-mgr  |
```

除了团队结构，Agent 还知道：
- **项目管理流程** —— 怎么创建 Jira 工单、哪些字段是必填的、工单状态怎么流转
- **版本发布策略** —— 哪个分支对应哪个产品版本、backport 的规则是什么
- **沟通渠道** —— bug 分析报告发到哪个群、周报发到哪个频道

说白了，这就是你带新人入职时需要告诉他的那些东西——不只是"代码怎么写"，还有"我们怎么协作"。

## 渐进式知识加载

AI 模型有一个 Context Window（上下文窗口）的硬限制——一次能处理的文本量是有上限的。如果在启动时把所有文档、所有技能定义、所有团队花名册全塞进去，Agent 会被无关信息淹没，反而表现更差。

解决方案是**渐进式加载（Progressive Disclosure）**：只加载当前任务需要的知识。

每个技能文件都明确声明了它的文档依赖：

```markdown
## Reference Loading

| Document              | When to Load                          |
|-----------------------|---------------------------------------|
| team-members.md       | Always (for assignment lookups)       |
| docs/jira.md          | When creating or updating Jira tickets|
| docs/build-release.md | When the bug involves release branches|
| repos/repos.yaml      | When identifying which repo to analyze|
```

当用户说"帮我看看今天有哪些新 bug"，Agent 加载 bug 处理工作流、团队成员映射和仓库清单——但跳过版本发布策略和 CI 配置文档。当任务是"把这个修复 backport 到 release-2.8"，Agent 加载分支策略和版本对应表——但跳过 Jira API 文档。

不同任务，不同知识集。Context Window 始终聚焦在真正重要的内容上。

## Dockerfile：可复现的执行环境

我们很早就遇到了一个经典问题：同样的工作流，在一个工程师的机器上跑得好好的，换个人就报错。原因总是环境不一致——不同的操作系统、不同的工具版本、缺少某个命令行工具。

Agent 的执行依赖多个层面：

1. **操作系统和 Shell** —— macOS 和 Linux 在路径处理、命令参数、甚至 `sed` 语法上都有差异
2. **工具版本** —— `kubectl`、`helm`、`jq`、`yq`、编程语言运行时——任何一个版本不一致都可能导致难以排查的错误
3. **Agent 运行时** —— 不同的 AI 编程工具（Claude Code、Cursor、OpenCode）有不同的能力和工具调用行为

Dockerfile 把这一切锁定：

```dockerfile
FROM debian:bookworm-slim

# Core tools
RUN apt-get update && apt-get install -y \
    git jq curl wget openssl unzip make gcc g++

# Go toolchain (pinned version)
ENV GO_VERSION=1.24.4
RUN curl -fsSL https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz \
    | tar -C /usr/local -xz

# CLI tools (pinned versions)
RUN curl -LO "https://dl.k8s.io/release/v1.31.4/bin/linux/amd64/kubectl" && ...
RUN curl -fsSL https://get.helm.sh/helm-v3.17.0-linux-amd64.tar.gz \
    | tar -xz && ...
RUN curl -LO https://github.com/mikefarah/yq/releases/download/v4.45.1/yq_linux_amd64 \
    && ...
```

Dockerfile 签入仓库后，任何工程师都能构建出完全相同的环境。再也没有"在我机器上能跑"的问题。Agent 在容器里运行，工具链完全确定，和宿主机无关。

## 从本地工具到常驻 Agent

上面所有内容作为本地方案已经很好用了——克隆 Agent 仓库，启动 AI 助手，开始干活。但自然的下一步是：**把 Agent 部署到云端，让它 24/7 在线运行**。

有了 Dockerfile 标准化环境，就可以构建容器镜像并部署到 Kubernetes 集群。我们基于 Kubernetes 开发了 [KubeOpenCode](https://github.com/kubeopencode/kubeopencode)，一个 AI Agent 管理平台，专门做这件事。Agent 部署上去之后，就变成了一个持久化的服务：

- **始终在线** —— 不需要在笔记本上开着终端
- **自动待机** —— 空闲时缩容节省资源，有任务时自动唤醒
- **知识热更新** —— Agent 仓库每隔几分钟从 Git 自动同步；团队成员合并了新技能，运行中的 Agent 自动获得新能力，无需重启
- **定时工作流** —— Cron 触发的任务，不需要人工启动

```yaml
# 示例：工作日每天早上 9 点自动梳理新 bug
apiVersion: kubeopencode.io/v1alpha1
kind: CronTask
metadata:
  name: daily-bug-triage
spec:
  schedule: "0 9 * * 1-5"    # 周一到周五 9:00
  taskTemplate:
    spec:
      description: |
        Run the daily bug triage workflow.
        Collect new bugs, analyze relevance, assign owners,
        post summary to Slack.
```

有了定时任务，Agent 从被动响应变成主动工作。它在团队站会前就把新 bug 梳理好、分配到人。每周五自动生成 PR 活动报告。每周一自动清理过期的 bot PR。团队早上看到的不再是一堆未整理的积压工作，而是已经分好类、分析过的任务清单。

凭证（API Token、Git 认证）统一管理在集群的 Kubernetes Secret 中——不再需要到处传 `.env` 文件或让每个人单独配置。

## 核心公式

整个模式可以归结为一个简单的公式：

```
代码仓库 + AI 编程助手 = Agent
```

仓库提供身份、技能、知识和工作流。AI 助手提供推理、执行和自然语言交互能力。两者结合，形成的 Agent 具备以下特性：

- **版本控制** —— 每一次变更都有记录、可审核、可回滚
- **可审计** —— `git log` 展示 Agent 演进的完整历史
- **协作式** —— 团队成员通过 PR 贡献技能和知识
- **可移植** —— 适用于任何能读取文件的 AI 编程助手
- **可复现** —— Dockerfile 确保跨环境的一致行为
- **可扩展** —— 部署到 Kubernetes 实现 24/7 运行和定时任务

没有对特定 Agent 框架的 vendor lock-in。没有私有的插件格式。没有复杂的编排层。只有文件、Git，以及你每天都在用的工程实践。

## 如何开始

如果你想尝试这个模式：

1. **创建一个仓库**，放一个 `CLAUDE.md`（或类似的入口文件）描述 Agent 的角色和规则
2. **添加 `skills/` 目录**，先写一两个任务定义——从团队反复执行的工作开始
3. **添加 `docs/` 目录**，放入 Agent 需要的参考资料——团队联系方式、项目规范、API 文档
4. **让你的 AI 助手打开这个仓库**，开始给它布置任务

不需要 Kubernetes，不需要 Docker，不需要任何基础设施就能开始。一个本地仓库加一个 AI 编程工具就够了。基础设施层（Dockerfile、云端部署、定时任务）可以随着需求逐步添加。

最好的一点？当队友问"这个 Agent 能干什么"——你把仓库链接发给他就行。所有答案都在文件里。
