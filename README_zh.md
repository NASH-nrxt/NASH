# NASH on CyberGym

*一个由自优化安全 Skill Tree驱动的AI执行框架*

[English](README.md) · **中文**

---

## 摘要

NASH 在 [CyberGym](https://github.com/sunblaze-ucb/cybergym) 的1507个真实世界漏洞任务上，产生了1501个PoC，其中1381个为 fixed-clean PoC，经过验证严格通过率为91.6%。
CyberGym Level 1 要求 agent 只基于漏洞描述与修复前的代码库生成一个最终 PoC；该 PoC 需要在 vulnerable 目标上触发漏洞，并且不能在 fixed 目标上触发。

## 背景

CyberGym 测试智能体是否能利用漏洞描述和补丁前的源代码树重现现实世界的漏洞。在 arvo 和 oss-fuzz 家族中，共发现了 1,507 个真实漏洞，是基于可重复结果评估代理行为的最强公开基准之一。

## 方法概述

NASH 是一套面向 AI+攻击与 AI+防御的统一安全任务 Harness，将专业 Skill、任务编排、证据组织、可信验证和经验回流整合为可审计的执行闭环。其核心是“可验证自进化”：从受控任务轨迹中提炼可迁移经验，经泄漏检查与验证门控后持续优化 Skill 与执行策略，实现安全任务从执行、裁决、验证到经验沉淀的闭环增强。

### Skill Tree 能力路由

NASH 将跨任务轨迹中可迁移的漏洞复现经验压缩成层次化 Skill Tree。根节点保存通用复现纪律，下层节点按输入格式、漏洞机制和复现流程组织；Skill router 根据公开的漏洞描述和任务类型选择最多两个适用节点，并与根节点合成本任务的 effective skill。

### NASH 的基础 Skill tree 构建

NASH 的基础 Skill tree构建在正式评测前完成：系统基于漏洞挖掘相关数据集所产生的执行轨迹进行训练与归纳，从成功和失败样本中提炼跨任务可迁移的复现策略、路由特征与风险规避规则，逐步构建并优化 Skill Tree。候选更新不包含具体答案，并须通过泄漏检查和验证集门控；未带来严格提升的更新会被回退。CyberGym 评测期间不检索历史 PoC、补丁或跨任务原始轨迹。该机制借鉴 [Microsoft SkillOpt](https://github.com/microsoft/skillopt) 的轨迹驱动技能优化思想，并将单文档技能扩展为可路由的层次化技能包。

### 智能体隔离执行

正式解题 agent 在每任务独立的隔离容器中，读取漏洞描述、修复前源码和选定 Skill，完成源码定位、输入格式重建、候选 PoC 构造、vulnerable 侧验证和最终提交；fixed 侧验证由主机评测器独立完成。任务运行产生的成功经验和失败模式不会作为跨任务可检索的样本记忆直接暴露给后续 agent，而是进入 Skill Tree 的离线更新流程：系统定期汇总多任务轨迹中反复出现的复现障碍、误判模式和有效操作，将其抽象为与具体任务无关的候选 skill，更新到skill tree中。这样，跨任务经验以通用技能的形式离线更新进 Skill Tree。

这种设计的目标是把安全研究中的“可迁移经验”前置为稳定的过程先验：让 agent 更好识别应该审计的输入边界、长度/计数/偏移关系、格式约束、sanitizer 触发条件和 both-crash 风险。

## 基准

| 指标 | 数值 |
|---|---:|
| 任务数 | 1,507 |
| 验证成功 | 1,381 |
| 未通过 | 126 |
| final-submission success rate | 91.64% |

| 结果 | 数量 | 说明 |
|---|---|---:|
| `verified_success` | 1,381 | 
| `unsuccessful` | 102 | 智能体提交的poc会导致补丁前与补丁后的镜像都造成崩溃或超时
| `incomplete_verification` | 18 | 智能体最终选择的 PoC 在补丁前镜像的exit_code为0或300 |
| `no_final_submission` | 6 | 智能体未能生成有效的 PoC |
| 总计 | 1,507 |

## 评测设置

实验设置对齐 CyberGym 官方 Level 1 约束以及 [FAQ](https://github.com/sunblaze-ucb/cybergym/blob/main/FAQ.md) 的要求。

| 项目 | 设置 |
|---|---|
| 模型 | DeepSeek-V4-flash-0731 |
| Trials | 每个任务 1 次有效运行 |
| 单任务墙钟上限 | 240 分钟 |
| 最终 PoC 策略 | 每个任务只保留一个 `final_poc` 作为最终提交 |
| 计分方式 | final-submission metric：最终 PoC 必须触发 vulnerable，且不触发 fixed |

### 任务输入与动态环境

Agent 在每个任务中可访问：
- Level1输入: 每个任务提供漏洞文字描述和修复前源码，并在可用时提供任务相关的 Skills。`description.txt`、`repo-vul.tar.gz`
- 动态执行环境: 启用了动态环境。我们为每个任务了一个 cleaned vulnerable 镜像用于本地运行、调试和复现漏洞。该镜像基于官方数据集的任务vul镜像构建，排除了 `/src/**/.git` 与 `/tmp/poc` 等泄漏源。
- 本任务专属 workspace 与提交接口

Agent 不可访问：

- `repo-fix.tar.gz`
- patch diff
- 参考 PoC
- fixed 镜像或 fixed 端验证反馈
- 原始 Git 历史、上游 PR、commit、issue、changelog、release note, 除了任务源码中原本附带的 CHANGELOG/NEWS 等项目文件
- 其他任务的 workspace、轨迹、PoC 或验证结果
- 部分任务评测日志中出现了试图访问git、patch的行为，但未获取到任何信息

动态环境按 CyberGym FAQ 建议处理：如果向 agent 提供 vulnerable 镜像作为动态分析环境，需要在公开材料中说明，并清理 `/src/**/.git` 与 `/tmp/poc` 等泄漏源。

### 隔离与网络策略

- 每个任务在独立 Docker 容器中运行。
- 每个任务使用独立 workspace；任务结束后容器清理，workspace 作为审计材料保留。
- 容器 rootfs 只读，未挂载 Docker socket。
- 容器能力最小化；仅保留调试动态分析所需能力。
- Agent 容器在内部 Docker 网络中运行。
- CyberGym 提交服务只绑定到本机受控 Docker 网络网关，不暴露到互联网。
- 外部网络通过受控代理限制；评测日志中 web search / web fetch 请求数为 0。



## 用量与成本估算

| 指标 | 均值 | 总量 |
|---|---:|---:|
| 非缓存输入 token | 116,594.24 | 175,707,519 |
| 缓存读取 token | 41,080,023.04 | 61,907,594,721 |
| 输出 token | 152,764.98 | 230,216,824 |
| LLM 请求数 | 201.04 | 302,967 |
| 墙钟时间 | 2,501.58 秒 | 3,769,882 秒/1047.2 小时 |
| 估算模型成本 | 5.84 元 | 8789.8 元 |

本次评测算力计费规则说明：测评中间经历了DeepSeek价格调整，当前成本为统一换算到涨价后的价格得到的成本。
