---
layout: default
title: "Horizon Summary: 2026-10-23 (ZH)"
date: 2026-10-23
lang: zh
---

> 从 35 条内容中筛选出 5 条重要资讯。

---

**工程干货精选**
1. [美团 SLM 有界生成推理引擎](#item-tech-top5-1) ⭐️ 9.0/10
2. [美团外卖统一推荐基座 MTFM 落地](#item-tech-top5-2) ⭐️ 9.0/10
3. [美团搜索 3.0：LLM 语义表征在排序模型的探索与应用](#item-tech-top5-3) ⭐️ 9.0/10
4. [面向 Qwen 系列模型线性注意力的高性能优化实践](#item-tech-top5-4) ⭐️ 9.0/10
5. [无消息代理的分布式批处理](#item-tech-top5-5) ⭐️ 8.0/10

---

## 工程干货精选

<a id="item-tech-top5-1"></a>
### [美团 SLM 有界生成推理引擎](https://tech.meituan.com/2026/10/15/Small-Language-Model-in-meituan.html) ⭐️ 9.0/10

rss · 美团技术团队 · 10月22日 07:03

**「技术方案」** 美团将搜推场景中 decode 次数有固定上界的请求定义为「有界生成负载」（Prefill-Only 与固定 Beam Search，输入平均不足 500 Token，beam 宽度不超过 400、生成不超过 5 步），并以排队论 G/G/1 模型分析说明序列分批使请求到达严重偏离泊松分布，SGLang/vLLM 为无界生成设计的优化在此均为负向。系统以静态 DAG 组织完整业务请求，作为调度、测量与容量规划的共同边界，请求内多条样本一次进入模型节点而非拆成独立生成任务；离线经 AOTInductor 编译为 AOTI 节点产物，固定 Beam Search 的逐 step 控制也编译进单个生成节点，在线由 C++ Runtime 完成协议处理、节点调度与资源管理。节点级专项优化包括：sequence boundary 从 tokenizer 贯穿到 attention 的 Packed Prefill 与 Triton Packed GQA；请求内 shared-prefix K/V 布局，状态量从 W\(L\_p+L\_s\) 降至 L\_p+W·L\_s；Beam 候选裁剪将全局合并从 W×V 降至 W×2W，Token ID 走 BF16 Top-K、分数走 FP32 log-softmax 分路；选择性 FP8（activation 用 per-token 动态 E4M3 量化，softmax、累计分数等敏感位置保留 FP32）；byte trie tokenizer。部署为单卡完整模型副本、无 TP/PP。作者报告实测：约 0.6B Prefill-Only 模型约 100 request QPS（约 1500 sample/s），0.5B Beam Search 短/长 prompt 约 300/120 QPS，平均时延均低于 20ms、P99 约 70ms，Prefill-Only 混合精度有效 MFU 达 67%、较 SGLang 约 10 倍，分层 A/B 各项优化收益 12%-39%；正确性按算子数值对照、双 Runtime 逐字节一致与业务指标验收分层验证，开放式生成、动态 decode 长度等场景作者明确建议继续使用 vLLM/SGLang。

**「踩坑与收获」** 作者坦诚记录的多处取舍可直接复用：FlashAttention-2 与 FlashInfer 路径集成进 AOTI 编译流程后存在稳定性问题（后者虽有约 3% 收益仍被弃用），专用 GPU Top-K 在完整 DAG 中无稳定收益即保留 Torch 实现，避免仅凭 microbenchmark 决定线上实现。FP8 并非越广越好，小矩阵或低计算占比路径无法稳定获益，kernel 与精度配置须按目标 GPU 逐模型固化，直接沿用其他模型配置可能同时损害业务效果与性能；容量选点也不能追无约束峰值 QPS——超过时延约束容量点后再加并发，QPS 仅增数个百分点而平均时延升 30% 以上；跨请求 prefix cache 因模板过早进入请求特有内容、命中率很低，索性不维护。

**标签**: `#SLM推理`, `#AOTInductor`, `#Packed Prefill`, `#FP8量化`, `#静态DAG`

---

<a id="item-tech-top5-2"></a>
### [美团外卖统一推荐基座 MTFM 落地](https://tech.meituan.com/2026/09/22/Meituan-Foundation-Model-for-Recommendation.html) ⭐️ 9.0/10

rss · 美团技术团队 · 10月22日 07:03

**「技术方案」** 美团在生成式精排框架 MTGR 之上推出统一推荐基座 MTFM，首次把外卖多业务精排合并为一个模型：异构 Tokenizer 将输入统一为 H-Token（跨业务历史）、R-Token（实时行为）、T-Token（预估目标）三类 Token 序列，多场景特征免显式对齐，动态掩码防止跨场景与时间维度的信息泄漏。注意力采用 Target:Full=3:1 的 16 层混合架构（少量 Full Attention 维持全局依赖、主要堆叠 Target Attention），显存下降使 batch size 扩大 1.7 倍、吞吐达全 Full Attention 的 2.9 倍；V2 进一步用 AutoSplit 将每个 Target 拆为 4 个 Token、配独立 Q 投影与 FFN，并以 RankMixer Mixing 防止表征坍缩，实现 Target 侧 Scaling。训练采用共享基座 + MMoE + 业务任务塔的解耦结构，User-Level 统一样本让同一用户当日多业务曝光共享最昂贵的长序列 Attention 计算。叠加 kernel 级优化（Triton 重构 Mask 构造 22ms→1ms、批量化 Newton-Schulz 阶段加速 241%、查表合并显存 22GB→5.9GB），作者报告各业务订单提升 2.06%~6.68%、推理成本降低 24%，并在层数、Token 维度、序列长度三个维度验证了 Scaling 趋势。

**「踩坑与收获」** 最值得复用的是深层联训稳定性经验：初期随机重复实验离线指标波动超 0.4pp，定位到与部分层梯度大小强相关后，用 GPT-2 式按深度缩放初始化（L=16）加 Muon（仅作用于 UQKV 与 Output Projection）将波动压到 0.1pp。多业务联训中小流量业务有效 batch 小、梯度噪声大，损失需按 batch 内 PV 占比动态归一化，零梯度步要跳过 Weight Decay；作者还发现 flash-muon 的上三角技巧在小矩阵场景反被 Triton 启动开销拖慢，改批量化 NS 才见效。

**标签**: `#推荐基座模型`, `#混合注意力`, `#Muon优化器`, `#Triton算子优化`, `#多场景建模`

---

<a id="item-tech-top5-3"></a>
### [美团搜索 3.0：LLM 语义表征在排序模型的探索与应用](https://tech.meituan.com/2026/08/20/01-meituan-Query-3.0.html) ⭐️ 9.0/10

美团服务零售搜索团队三期实践复盘：以对比学习（InfoNCE+Triplet、难负样本）训练 LLM 语义表征，经相似度分桶与 PEPNet 门控注入精排模型，带来订单 +0.20%~0.32% 的实测线上收益，并沉淀出可迁移的表征工程方法论。

rss · 美团技术团队 · 10月22日 07:03

**标签**: `#LLM语义表征`, `#搜索精排`, `#对比学习`, `#LoRA微调`, `#PEPNet门控`

---

<a id="item-tech-top5-4"></a>
### [面向 Qwen 系列模型线性注意力的高性能优化实践](https://www.infoq.cn/article/o9OcWNkkwgKvsZmgY45d?utm_source=rss&amp;utm_medium=article) ⭐️ 9.0/10

Qwen 团队深度复盘 FlashQLA：通过序列并行矫正矩阵、GDN 滑动窗口性质利用、大 Gamma 代数改写和 TileLang 手写 warp specialization 算子，在 Hopper 与 Blackwell 上实现 GDN 线性注意力前向 2-3 倍、反向 2 倍加速并成为 FLA 官方后端。

rss · InfoQ 中文站 · 10月22日 17:08

**标签**: `#线性注意力`, `#GDN`, `#GPU算子优化`, `#warp specialization`, `#序列并行`

---

<a id="item-tech-top5-5"></a>
### [无消息代理的分布式批处理](https://www.infoq.com/articles/spring-batch-without-broker/?utm_campaign=infoq_content&amp;utm_source=infoq&amp;utm_medium=feed&amp;utm_term=Architecture) ⭐️ 8.0/10

rss · InfoQ Architecture · 10月21日 09:00

**「技术方案」** 作者基于自己维护、经 JOSS 同行评审的开源实现，提出用 Spring Batch 本就必需的关系数据库充当唯一协调平面，替代消息代理且无需 ZooKeeper：节点注册、分区生命周期、作业协调分别落在 BATCH\_NODES、BATCH\_PARTITIONS、BATCH\_JOB\_COORDINATION 三张普通表，集群状态用 SELECT 即可检查。协调去中心化——发起作业的节点仅担任该次执行的 master，无选主机制；开发者只替换 Partitioner 和 PartitionHandler 两个 bean，worker 步骤照旧，一个配置开关即启用节点注册、心跳与轮询。master 切分前先查询存活节点数以定分片规模，再按 round-robin、固定节点数或 least-loaded 一次性分派（放弃了 broker 拉取语义免费提供的运行时再均衡）；worker 通过 compare-and-set 的 UPDATE（限定 assigned\_node 且 status IN \(&\#x27;PENDING&\#x27;,&\#x27;CLAIMED&\#x27;\)）认领分区，重派沿用同一 CAS 纪律，终态分区不会被复活。节点故障走两阶段心跳恢复：约 10 秒无心跳标记 unreachable，约 30 秒后移除并重派其未完成的可转移分区，两阈值之间的宽限期专门容忍 GC 停顿与网络抖动。正确性是显式契约：可转移分区至少执行一次、必须幂等，不可转移分区至多一次、失败而不迁移；master 丢失时由幸存节点原子认领搁浅的协调行并标记失败，靠幂等重启恢复。

**「踩坑与收获」** 作者明示天花板是数据库争用而非固定节点数，适用约 2-20 节点：心跳、轮询、状态更新都是数据库流量，会与批处理争抢连接池和 I/O，规模到几十节点或需亚秒级协调延迟时应仍选 broker。护栏（fencing）只是尽力而为——硬暂停的 JVM 醒来可能继续执行已被重派的分区，可转移任务必须真正幂等；master 丢失是失败重启而非原地续接，不可转移任务若须跨重启存活需在对账步骤补齐；数据库成为硬依赖，须按关键组件标准运维，并为热轮询与完成扫描查询建索引。

**标签**: `#Spring Batch`, `#分布式批处理`, `#远程分区`, `#数据库协调`, `#故障恢复`

---