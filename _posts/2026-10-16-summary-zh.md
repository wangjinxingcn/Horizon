---
layout: default
title: "Horizon Summary: 2026-10-16 (ZH)"
date: 2026-10-16
lang: zh
---

> 从 35 条内容中筛选出 5 条重要资讯。

---

**工程干货精选**
1. [美团有界生成 SLM 推理引擎](#item-tech-top5-1) ⭐️ 9.0/10
2. [美团外卖多业务统一推荐基座](#item-tech-top5-2) ⭐️ 9.0/10
3. [LLM 语义表征注入搜索精排](#item-tech-top5-3) ⭐️ 9.0/10
4. [GeoRA：RLVR 版 LoRA](#item-tech-top5-4) ⭐️ 8.5/10
5. [美团 Agent 评测体系全景](#item-tech-top5-5) ⭐️ 7.0/10

---

## 工程干货精选

<a id="item-tech-top5-1"></a>
### [美团有界生成 SLM 推理引擎](https://tech.meituan.com/2026/10/15/Small-Language-Model-in-meituan.html) ⭐️ 9.0/10

rss · 美团技术团队 · 10月15日 07:11

**「技术方案」** 美团将搜推 SLM 的在线负载归结为「有界生成」（Prefill-Only 与固定深度 Beam Search，控制流和状态容量可在部署前界定），并以 Kingman 排队近似论证 SGLang/vLLM 面向无界生成设计的优化在此场景是负向的，据此自研推理引擎。系统以静态 DAG 为执行边界、完整业务请求作为调度与容量规划的共同单元：离线经 torch.export 与 AOTInductor 将模型（固定 Beam Search 的 N\_gen−1 次状态更新表达为静态张量程序）编译为 AOTI 节点产物，在线由 C++ Runtime 承担协议处理、节点调度与资源预分配，热路径无 Python 解释与 JIT。节点内专项优化包括：Packed Prefill 从 tokenizer 起保留 sequence boundary 以定义 block-diagonal mask，配合不物化 attention score 的 Triton Packed GQA，线性层计算量由 B×L\_max 降至 ΣL\_i；Beam 状态采用请求内 shared-prefix K/V 布局，规模从 W\(L\_p+L\_s\) 降至 L\_p+W·L\_s；候选选择先每 beam 取 2W 局部候选再合并为 W×2W 全局候选，Token ID 在 BF16 logits 上 Top-K、分数取 FP32 log-softmax；MLP 等密集矩阵施行 per-token 动态 E4M3 量化的选择性 FP8，敏感位置保留 FP32。实测约 0.5–0.6B 模型平均端到端时延低于 20ms、P99 约 70ms，Prefill-Only 混合精度有效 MFU 达 67%（据称为 SGLang 约 10 倍）、短 prompt Beam Search 为 41%（约 6 倍），三类负载分别约 100/300/120 request QPS。

**「踩坑与收获」** 可复用的教训：FlashAttention-2 变长接口无法同时满足 AOTI 集成与目标 GPU 兼容，FlashInfer 共享 page table 的 Paged Decode 集成后存在稳定性问题（仅约 +3% 收益即弃用），最终自研 Triton kernel；无约束峰值吞吐不等于时延约束下的容量——选定容量点后继续加并发，QPS 仅增几个百分点而平均时延上升 30% 以上并突破 20ms 目标。FP8 覆盖范围须按模型与 GPU 分别固化，较小或计算占比低的 linear 无法稳定获益；候选实现应以完整 DAG 的 A/B 而非 microbenchmark 定夺（专用 GPU Top-K 因无稳定端到端收益而保留 Torch 实现）；跨请求 prefix cache 因公共前缀短、命中率低被明确放弃，只利用请求内 prompt K/V 共享。

**标签**: `#SLM推理`, `#AOTInductor`, `#Packed Prefill`, `#FP8量化`, `#Beam Search`

---

<a id="item-tech-top5-2"></a>
### [美团外卖多业务统一推荐基座](https://tech.meituan.com/2026/09/22/Meituan-Foundation-Model-for-Recommendation.html) ⭐️ 9.0/10

rss · 美团技术团队 · 10月15日 07:11

**「技术方案」** 美团在 MTGR 基础上推出统一推荐基座大模型 MTFM，首次实现外卖多业务统一精排：异构 Tokenizer 将输入分为跨业务历史 H-Token、实时行为 R-Token 与预估目标 T-Token，配合动态掩码（历史全可见、实时按时间因果、目标仅自可见）实现多场景特征免对齐。骨干为 16 层 HSTU 类 Block，按 Target:Full=3:1 交错堆叠 Full Attention 与 Target Attention 并引入 GQA，显存下降使 batch 扩大 1.7 倍、吞吐达全 Full 结构的 2.9 倍；V2 再以 AutoSplit 将每个 Target 拆为 4 个 Token、配独立 Q Projection 与 SwiGLU FFN 实现 Target Scale-Up，并以 RankMixer 零参数 Mixing 防止拆分后表征坍缩。训练采用&quot;共享基座+MMoE+业务独立任务塔&quot;，User-Level 样本把同一用户当日多业务曝光聚合为一条以共享长序列 Attention 计算，部署时各业务导出共享基座与独享任务塔，实现统一训练、多业务部署。据作者披露，各业务订单提升 2.06%~6.68%，推理成本降 24%，单样本计算量达 192 GFLOPs，并在层数、Token 维度（192~768）、序列长度（400~2000）三个维度验证了 Scaling 能力。

**「踩坑与收获」** 多业务联合训练叠加深层网络曾使随机重复实验的 train AUC 波动超 0.4pp，作者借鉴 GPT-2 按深度缩放输出投影层初始化（L=16，其余线性层 std 取 0.02），并将 Muon 仅作用于 HSTU 的 UQKV 与 Output Projection，波动降至 0.1pp。损失由各业务独立归一化改为按 batch 内 PV 占比动态加权（等价于全局归一化）以消除小流量业务的梯度噪声，并让优化器跳过零梯度参数的 Weight Decay、辅以步长为 2 的梯度累积。此外 flash-muon 的上三角对称技巧在 MTFM 的小矩阵上反被 Triton 启动开销拖慢，改为同形状梯度堆叠批量执行 Newton-Schulz 后，该阶段加速 241%、端到端训练吞吐提升 19%。

**标签**: `#推荐系统基座模型`, `#HSTU`, `#混合注意力`, `#Muon优化器`, `#多场景建模`

---

<a id="item-tech-top5-3"></a>
### [LLM 语义表征注入搜索精排](https://tech.meituan.com/2026/08/20/01-meituan-Query-3.0.html) ⭐️ 9.0/10

rss · 美团技术团队 · 10月15日 07:11

**「技术方案」** 美团搜索团队在 2025 Q4 至 2026 Q2 分三期将 LLM 语义表征注入服务零售精排：微调中等参数的 Embedding 基座生成 Query/POI/Deal 向量，将 cosine 相似度按预设边界分桶（一期边界为\[-0.40,-0.30,-0.18,-0.12,0,0.10,0.16,0.22,0.30\]共 10 桶）转为可学习 Embedding 拼接进模型。一期用三个特殊 Token 作聚合锚点、Attention Mask 三次前向隔离 Query 与 Item 信息，双 Loss（cosine 相似度 BCE+融合表征预测 CTR）训练 3000 余万样本，线上大盘订单+0.20%、长尾 NDCG@5 +2.21pp。二期系统性重构：五元组训练数据引入&quot;同请求曝光未点击&quot;难负样本，InfoNCE+Triplet（margin=0.5，网格搜索确定）对比学习，LoRA 微调（r=8、α=32、q\_proj/v\_proj），表征提取改为各实体独立序列+nn.Parameter 覆写末位向量，MRL-E 嵌套降维，Q2I-Order-AUC 较一期+5.37pp，线上 QV\_CTR +0.10pp 且 TP90 仅+0.2ms。三期将二期表征迁移至下挂精排，按目标场景样本分布重新圈选 Embedding 使双覆盖率从 73.61%升至 89.81%，对比四种注入方式后选定 PEPNet 门控（离线+25bp，优于底层拼接+18bp），叠加全域交叉统计特征后线上订单+0.32%。

**「踩坑与收获」** 作者发现 Embedding 场景的 Prompt 做减法优于复杂任务指令，未经筛选引入 CPV 商品属性反而带来负向，说明信息质量比信息量更重要。跨场景迁移表征的核心风险在覆盖率而非模型参数：Query 分布从商家意图词变为商品意图词导致双覆盖率仅 73.61%，必须按目标场景重圈；覆盖率低于 0.1%的统计特征即使方向正确也在消融中被移除。线上曾出现 QV 略降但 CTR 显著提升的现象，下钻确认是排序优化减少了无效曝光；LoRA 在 NDCG 上优于全参但 AUC 略逊，作者明示该现象仅限本场景、普适性待验证。

**标签**: `#LLM语义表征`, `#搜索精排`, `#对比学习`, `#难负样本`, `#PEPNet门控`

---

<a id="item-tech-top5-4"></a>
### [GeoRA：RLVR 版 LoRA](https://tech.meituan.com/2026/08/27/ACL-Outstanding-Paper-GeoRA.html) ⭐️ 8.5/10

rss · 美团技术团队 · 10月15日 07:11

**「技术方案」** 美团与北大的 ACL 2026 杰出论文 GeoRA 的核心洞察是：LoRA（高斯初始化）、PiSSA（取主奇异方向）、MiLoRA（取尾方向）的谱先验均按 SFT 几何设计，而 RLVR 的有效更新分散在稀疏子空间并避开预训练主方向，直接沿用构成几何错位——PiSSA 强推主成分的归纳偏置与 RLVR 相冲突，实验中更易崩溃。GeoRA 在训练前一次性执行&quot;定位+压缩&quot;：用共享稀疏率 ρ 的双掩码——谱先验选秩 r 低秩近似中幅值最小的 ρ 位置（保稳定）、欧氏先验选原始权重中最小幅值的 ρ 位置（保可塑）——取并集得到几何子空间 W\_Geo（Qwen3-8B 上 ρ=0.2 时两掩码 Jaccard 仅 0.128，互补不冗余），再对 W\_Geo 而非原始权重做截断 SVD，取 top-r 分量初始化低秩适配器，并冻结残差 W\_res 作锚点，保证初始化函数不变、避免冷启动扰动 rollout 采样；训练期计算图与标准 LoRA 完全一致。作者报告在 1.5B–32B 的 Qwen/Llama 上以 GRPO 做数学、医学、代码 RLVR，GeoRA 稳定优于低秩基线并与全参微调相当（Qwen3-8B AIME24 达 23.75% 略高于全参），分布外遗忘明显少于全参，相较全参可训练参数降 99.5%、单步耗时降 19.9%、显存降 28.5%。业务侧已在 AI 骑手招聘的 Agentic RL 中落地：效果与全参相当、较 LoRA 提升约 12%，显存较全参降 54%。

**「踩坑与收获」** 原文的对照实验揭示一个效率陷阱：直接对 RLVR 偏好子空间做稀疏微调（SparseFT）虽把参数降了 68.0%，单步耗时反而上升 10.8%，因为非结构化稀疏不被 GPU 高效支持，参数效率必须落成稠密低秩计算才能真正兑现。另有两条可直接迁移的技巧：SVD 初始化只需前 r 个奇异分量，改用随机 SVD（随机投影到小空间分解）后 72B 模型处理不到 1 分钟；利用&quot;原始权重=残差+初始适配器&quot;的函数不变性，在 actor 内多冻结一份低秩初始适配器即可现算 KL 参考策略，省去一整份参考模型权重。

**标签**: `#GeoRA`, `#LoRA`, `#RLVR`, `#参数高效微调`, `#GRPO`

---

<a id="item-tech-top5-5"></a>
### [美团 Agent 评测体系全景](https://tech.meituan.com/2026/09/10/Agent-Evaluation-White-Paper-01.html) ⭐️ 7.0/10

rss · 美团技术团队 · 10月15日 07:11

**「技术方案」** 美团评测团队基于两年多业务实践提出&quot;四模块、三能力、双 Loop、一套资产&quot;的评测体系：离线评测以固定评测集和贴近生产的固定执行环境充当版本变更门控；在线评测（AB、影子模式、巡检）与在线监控分别从质量与运维视角观察线上；Case 挖掘与归因作为枢纽，把线上真实样本分流进 Agent 迭代与评测体系迭代两条 Loop；Trace 观测基建是一切的地基。评测集由（问题、参考答案、评价标准）三元组构成 Task，分为端到端与过程两类——前者站用户视角回答&quot;要不要拉警报&quot;、后者回答&quot;警报响了找谁&quot;，并可按功能模块分层拆解以对齐责任归属。回测采用 Pass@k 多次试验取通过率而非单次结论，安全与数据准确性类 Rubric 一票否决、体验类设阈值，且必须接入 CI/CD 才有约束力。作者在 FAQ 中还主张 Agent 不宜像基座模型那样打榜，业务指标与线上 AB 才是评价能力的黄金标准。

**「踩坑与收获」** 实践中最易被破坏的是离线执行环境保真度，可用同一批样本分别在离线环境与线上影子流量各跑一遍、比对两边结论是否一致来校验。归因时不要默认&quot;评测说不行就是 Agent 不行&quot;，需定期检查 Rubric 是否判错，否则 Agent 会被优化成迎合评测标准而非解决用户问题；同时随机采样虽成本最高、命中率最低，但作为唯一能发现&quot;没想到的问题&quot;的手段不可省略。评测集的分层拆分应由真实问题驱动而非完备性焦虑驱动，不要过早拆细；并警惕&quot;评测集与监控看板齐备却缺 Case 挖掘归因枢纽&quot;的割裂形态——体系能力上限由最短的那块板决定。

**标签**: `#Agent评测`, `#评测集构建`, `#回测门禁`, `#在线监控`, `#Trace观测`

---