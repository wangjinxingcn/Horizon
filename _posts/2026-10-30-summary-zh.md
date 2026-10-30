---
layout: default
title: "Horizon Summary: 2026-10-30 (ZH)"
date: 2026-10-30
lang: zh
---

> 从 38 条内容中筛选出 5 条重要资讯。

---

**工程干货精选**
1. [分布式存储 AI Coding 实践与边界](#item-tech-top5-1) ⭐️ 9.0/10
2. [SLM 有界生成推理引擎实践](#item-tech-top5-2) ⭐️ 9.0/10
3. [美团 MTFM 统一推荐基座落地](#item-tech-top5-3) ⭐️ 9.0/10
4. [美团搜索 LLM 语义表征实践](#item-tech-top5-4) ⭐️ 9.0/10
5. [美团算法自进化系统架构演进](#item-tech-top5-5) ⭐️ 8.0/10

---

## 工程干货精选

<a id="item-tech-top5-1"></a>
### [分布式存储 AI Coding 实践与边界](https://tech.meituan.com/2026/10/29/AI-Coding-IN-Distributed-Storage-System.html) ⭐️ 9.0/10

rss · 美团技术团队 · 10月29日 07:43

**「技术方案」** 美团存储团队在低容错的分布式存储系统中实践 AI Coding，核心判断是瓶颈已从「写代码」移到「验证」：AI 负责扩大搜索空间与执行可控验证，人负责系统语义和最终判断。方法论上沉淀了四层互补的证据体系——静态推理找候选、构建与单测判给定输入下是否出错、集成测试（在单进程内拉起真实 Raft 多节点集群）验证已有保护下的可达性、运行时日志证明线上是否发生——并明确各层的证明边界。评审结论按「测试确认/测试证伪/源码确认/未验证」四档输出：作者实测全仓评审的 117 个疑似问题中 36 条由红测试确认、3 条被证伪；11 天 67.8 万行 WARN 日志映射回源码定位出 5 个缺陷，人工确认全部属实。关键验证走红绿闭环（修复前红、修复后绿、撤回再红），Debug 环节用「观察事实/候选/已验证因素/根因/放大因素/缓解」的术语纪律防止结论被措辞悄悄升级，自动化则以「只读分析/受控修改/受控执行/人工决策」的分层授权完成 31 个构建文件的 CMake 迁移与鲲鹏 ARM 适配。

**「踩坑与收获」** 文中坑点多具体到代码级：AI 针对 32 位溢出给出的修复建议，因 get\_size\(\) 返回 int、size 在 2³¹~2³² 区间被截断为负数而本身静默失效，说明修复建议同样需要评审；多 Agent 投票共享同一份盲区、会在同一处错得一致，票数只代表重合度而非正确度。此外，AI 索引失败后会静默降级为 grep、CI 报告硬编码 failures=&\#x27;0&\#x27;，应要求报告声明实际使用的工具与降级环节，并让 AI 查看原始构建输出而非只信 CI 绿灯。

**标签**: `#AI Coding`, `#分布式存储`, `#AI 代码评审`, `#AI Debug`, `#自动化验证`

---

<a id="item-tech-top5-2"></a>
### [SLM 有界生成推理引擎实践](https://tech.meituan.com/2026/10/15/Small-Language-Model-in-meituan.html) ⭐️ 9.0/10

rss · 美团技术团队 · 10月29日 07:43

**「技术方案」** 美团将搜推 SLM 负载归纳为「有界生成」（Prefill-Only 与固定深度 Beam Search），据此自研推理引擎：以静态 DAG 为执行抽象，完整业务请求是调度、测量与容量规划的共同边界，节点为编译边界，模型计算经 PyTorch AOTInductor 离线编译为 AOTI 产物，协议处理与调度由进程内 C++ Runtime 承担，消除 Python 调度开销。节点内嵌专项优化：变长样本走 Packed Prefill，sequence boundary 从 tokenizer 贯穿自研 Triton Packed GQA；固定 Beam Search 的候选选择与状态更新编译进单一 AOTI 节点，请求内各 beam 共享一份 prompt K/V，状态规模由 W\(L\_p+L\_s\) 降至 L\_p+W·L\_s；MLP 等密集矩阵采用选择性 FP8，softmax、Top-K 与累计分数保留 FP32。据作者实测，0.6B Prefill-Only 模型达约 100 request QPS（约 1500 sample/s），beam width 约 100、decode 4 次的模型分别约 300/120 QPS，平均时延均低于 20ms、P99 约 70ms，Prefill-Only 混合精度有效 MFU 为 67%，吞吐约为 SGLang 的 10 倍。

**「踩坑与收获」** 作者总结：通用 serving 框架为无界生成设计的优化在此场景是负向的，跨请求 prefix cache 因公共前缀短、命中率低而收益有限，故只保留请求内 K/V 共享；无约束峰值吞吐也不等于可用容量，短 prompt Beam Search 在容量点外加并发，QPS 仅增几个百分点而平均时延涨逾 30%。FlashAttention-2 变长接口无法兼顾 AOTI 集成与目标 GPU 兼容，FlashInfer Paged Decode 集成后仅约 3% 收益且有稳定性问题，最终均改用自研 Triton kernel。FP8 对计算占比低的 linear 收益不稳定，覆盖范围须按模型与卡型单独验证，且每项优化都要回到完整请求链路做分层 A/B 复验，不能只看 microbenchmark。

**标签**: `#SLM 推理`, `#AOTInductor`, `#Packed Prefill`, `#FP8 量化`, `#Beam Search`

---

<a id="item-tech-top5-3"></a>
### [美团 MTFM 统一推荐基座落地](https://tech.meituan.com/2026/09/22/Meituan-Foundation-Model-for-Recommendation.html) ⭐️ 9.0/10

rss · 美团技术团队 · 10月29日 07:43

**「技术方案」** 美团在 MTGR 基础上构建外卖多业务统一精排基座模型 MTFM，以 HSTU 类 Transformer 为骨干，用 H/R/T 三类异构 Tokenizer 加动态掩码实现多场景特征免显式对齐，其中 T-Token 各业务独立维护以承载分布差异。注意力采用 Target:Full=3:1 的 16 层混合架构（借鉴 Qwen3-Next 等 LLM 混合设计），显存下降后 batch size 扩大 1.7 倍、吞吐达全 Full Attention 的 2.9 倍；V2 进一步用 AutoSplit 将每个 Target 拆成 4 个 Token 并配独立 Q 投影与 SwiGLU FFN，以 RankMixer Mixing 防止表征坍缩。训练侧借鉴 LLM 实践：GPT-2 式按深度缩放初始化（L=16）、Muon 优化器仅作用于 UQKV 与输出投影、损失由分业务归一化改为按 PV 占比动态加权（数学上等价于全局归一化）。工程上以 User-Level 样本聚合同一用户当日多业务曝光、复用长序列注意力计算，配合 FA2 动态 Mask 构造（22ms→1ms）、GLN 融合（带宽利用率 30%→60%）、Newton-Schulz 批量化（阶段加速 241%）等定制算子控制开销。据作者披露，各业务订单提升 2.06%~6.68%、推理成本降低 24%，并在层数、Token 维度（192~768）、序列长度（400~2000）三个维度验证了稳健的 Scaling 能力。

**「踩坑与收获」** 作者坦诚复盘了两处关键教训：多业务联训叠加深层网络曾导致随机重复实验指标波动超 0.4pp，通过追踪中间层梯度定位到局部梯度衰减与参数退化，用深度缩放初始化加 Muon 将波动压至 0.1pp；flash-muon 的上三角对称优化在小矩阵场景因额外 Triton 启动开销反而变慢，改为同形状梯度堆叠批量执行 Newton-Schulz 才获得收益。此外，稀疏 batch 下零梯度参数跳过 Weight Decay、以及删减近 500 个特征（GAUC 无损、训练吞吐 +6%）都是可直接迁移的实操结论。

**标签**: `#推荐系统基座模型`, `#多场景建模`, `#HSTU`, `#Muon优化器`, `#Triton算子优化`

---

<a id="item-tech-top5-4"></a>
### [美团搜索 LLM 语义表征实践](https://tech.meituan.com/2026/08/20/01-meituan-Query-3.0.html) ⭐️ 9.0/10

rss · 美团技术团队 · 10月29日 07:43

**「技术方案」** 美团服务零售搜索团队分三期将 LLM 语义表征注入精排模型。一期在小参数基座上新增&lt;\|query\|&gt;、&lt;\|item\|&gt;、&lt;\|qi\|&gt;三个特殊 Token 作聚合锚点，通过 Attention Mask 三次前向隔离 Query 与商家信息，两侧 64 维表征的 cosine 相似度按预设边界分 10 桶、映射为可学习 Embedding 拼接注入，线上长尾 NDCG@5 显著+2.21pp、大盘订单+0.20%。二期升级为 Query-POI-Deal 三元体系：训练数据扩展为含难负样本（同请求、同商家曝光未点击）的五元组，微调改用 LoRA（r=8、α=32、仅 q\_proj 与 v\_proj），特殊 Token 改为可学习向量以一次前向处理五路输入，降维采用 MRL-E 截取 128 维，损失为覆盖三元两两关系的三组 InfoNCE 加 margin=0.5 的 Triplet Loss，应用上采用底层分桶拼接+顶层 LHUC 双融合，线上 TP90 仅+0.2ms。三期将表征迁移至下挂精排，对比四种注入方式后 PEPNet 门控效果最优（离线+25bp，高于底层拼接的+18bp），并与全域交叉统计特征叠加，服务零售订单显著+0.32%、大盘订单+0.35%。

**「踩坑与收获」** 文中沉淀了几条反直觉经验：Embedding 场景的 Prompt 做减法优于复杂任务指令，且未经筛选引入 CPV 商品属性信息反而降低表征质量；跨场景迁移表征的核心风险在覆盖率而非模型参数——直接复用时双覆盖率仅 73.61%，按目标场景 Query 分布重新圈选后才升至 89.81%；三期还出现 QV 微降但 QV\_CTR 显著提升的现象，下钻归因为排序优化减少了无效曝光，转化效率反而更高。

**标签**: `#LLM语义表征`, `#搜索排序`, `#对比学习`, `#LoRA微调`, `#PEPNet门控`

---

<a id="item-tech-top5-5"></a>
### [美团算法自进化系统架构演进](https://tech.meituan.com/2026/10/22/LOOP-AutoResearch.html) ⭐️ 8.0/10

rss · 美团技术团队 · 10月29日 07:43

**「技术方案」** 美团技术团队基于 Karpathy 的 AutoResearch 原型，三个月内三次重构，把单线程 keep/reset LOOP 演进为企业云端可用的 CyclicGraph 图探索架构。V2 确立“无状态触发 + 最小 JSON 状态文件”：每次触发新建会话，靠 phase 字段互斥路由 Monitor 与 Iterator 两个角色，Agent 不同步等待数小时训练而在下次触发时恢复现场，并借此与底层 Agent 基座彻底解耦。V3 的 Spawn-Converge 由主 Agent 把问题拆成 3~4 个互斥假设，子 Agent 上下文隔离并行验证、主 Agent 收拢对比决策，作者称同一项目迭代速度从 0.14 升至 2.55 次/小时（约 18 倍），两个业务项目单任务耗时较人工基线降约 85% 与 83%。可信度原则是“能用代码表达的约束不进 Prompt”：已证伪方向写入状态文件硬约束、平台调用封装为幂等工具并做结构化校验、区分代码故障与基础设施故障，经验以 Skill 草稿经“系统初筛 + 人工审核”两级晋升入库，再配合业务知识库、渐进式调研与线上真实反馈回流形成自进化闭环。

**「踩坑与收获」** 长会话同步等待训练会造成上下文爆炸与状态不同步，改为全新会话加状态文件恢复后才稳定；循环中错误结论一旦进入下一轮上下文会逐轮放大，必须让指标与代码 diff 等原始证据成为不可绕过的校验锚点。作者强调核心工程量在状态机、硬约束、证据链等 Harness 侧而非模型侧，并坦承 85%/83% 为自报数据（人工基线按工作日折算）、适用边界为指标可量化、实验成本可接受且结果可逆，跨项目复用经验库存在泛化幻觉风险，当前靠人工审核拦截。

**标签**: `#AutoResearch`, `#Agent Harness`, `#状态机`, `#多Agent并行探索`, `#自进化系统`

---