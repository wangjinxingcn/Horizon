---
layout: default
title: "Horizon Summary: 2026-10-19 (ZH)"
date: 2026-10-19
lang: zh
---

> 从 22 条内容中筛选出 5 条重要资讯。

---

**工程干货精选**
1. [生产级大模型平台工程实战手册](#item-tech-top5-1) ⭐️ 8.0/10
2. [Pi Durable 的效应三明治架构](#item-tech-top5-2) ⭐️ 7.0/10
3. [移动驾照 mDL 企业架构蓝图](#item-tech-top5-3) ⭐️ 7.0/10
4. [dm-thin 块复用残留致跨租户泄露](#item-tech-top5-4) ⭐️ 6.0/10
5. [816B MoE 模型如何搬上桌面](#item-tech-top5-5) ⭐️ 5.0/10

---

## 工程干货精选

<a id="item-tech-top5-1"></a>
### [生产级大模型平台工程实战手册](https://www.infoq.cn/article/xQLd004RGWrzFhwqBOyC?utm_source=rss&amp;utm_medium=article) ⭐️ 8.0/10

一篇基于零售库存平台真实落地经验的 LLM 平台工程实战复盘，系统给出网关路由、Schema 校验重试、提示词版本管理、成本归因与评估门禁的完整架构及配套代码，将幻觉率从 15% 降至 1.5%。

rss · InfoQ 中文站 · 10月16日 17:00

**标签**: `#LLM平台工程`, `#多智能体编排`, `#结构化输出校验`, `#提示词版本管理`, `#幻觉治理`

---

<a id="item-tech-top5-2"></a>
### [Pi Durable 的效应三明治架构](https://www.infoq.cn/article/hQOHf1ZXoBs7uMpWJQss?utm_source=rss&amp;utm_medium=article) ⭐️ 7.0/10

rss · InfoQ 中文站 · 10月16日 11:53

**「技术方案」** Pi 1.0 将 Code Mode 与原生 MCP 转正，并推出约 1.2 万行的长时 Agent 框架 Pi Durable。它弃用事件溯源，改用“意图→执行→结果”的效应三明治做持久化执行：执行前先落盘意图，执行中靠幂等键扛崩溃（以 GitHub Actions 为例，先取并保存运行 ID，重启后凭 ID 判断续跑或等结果），完成后持久化结果、丢弃中间信息，进程崩溃即可从断点精确恢复。Harness 内 LLM 调用与工具执行皆为可嵌套的任务三明治；应用状态存于版本化文档库，文档版本与转录记录位置关联，可按时间点或取最新，团队称 JSON Patch 在此场景有所欠缺，改用 JavaScript Proxy 记录对象操作以供重放。MCP 接入不把工具全量塞进上下文，而是通过工具搜索与渐进式披露按需加载；Code Mode 让 Agent 在 QuickJS 沙箱中编写运行 JavaScript，并行调用工具、过滤去重，中间数据留在沙箱以省 Token，并可在 JS 循环内直接调用 System 1 模型 Jev 逐条分类，前沿模型只看汇总结果。

**「踩坑与收获」** 关键避坑点：不可逆效应（刷信用卡、无版本控制的删文件）无法与持久化系统直接组合，所有副作用必须先设计成幂等。对话者还观察到 Agent 修 bug 几乎必然增加代码量，全自动维护的代码库会随每次修复不断膨胀，Mario 自述曾数月耗在修复 AI 生成代码的 issue 上，拖慢了 Pi Durable；有 Oracle、可 fuzz 的二元任务（如靠测试套件驱动的 Bun Rust 重写）才是模型强项，涉及架构设计因缺少文本化训练数据仍需人把关。

**标签**: `#Pi Durable`, `#持久化执行`, `#MCP`, `#Agent Harness`, `#QuickJS 沙箱`

---

<a id="item-tech-top5-3"></a>
### [移动驾照 mDL 企业架构蓝图](https://www.infoq.com/articles/enterprise-mobile-id-mdl-blueprint/?utm_campaign=infoq_content&amp;utm_source=infoq&amp;utm_medium=feed&amp;utm_term=Architecture) ⭐️ 7.0/10

rss · InfoQ Architecture · 10月16日 09:00

**「技术方案」** 文章主张把 mDL 集成当作企业架构与信任边界问题而非 SDK 对接问题：身份证明（证件拍摄、活体检测、DMV 人脸比对加人工复核）与凭证签发必须解耦，后者由 Secure Enclave 生成设备密钥，签发方将身份属性摘要绑定设备公钥后签名进 Mobile Security Object。传输层在线场景用 HPKE（DHKEM\(P-256, HKDF-SHA256\) + HKDF-SHA256 + AES-128-GCM），近场出示走标准 NFC，服务器间用 mTLS；选择性披露通过 mSO 中逐属性 digestID 加随机值的摘要结构实现（ES256/SHA-256）。作者给出代表性证书拓扑：根信任锚 10–20 年（ECDSA P-384/RSA 4096，离线 HSM）、签发 CA 3–5 年、Document Signer 30–90 天（在线 HSM）、mTLS 与验证端证书各 1 年、设备证明证书 1–3 年。可用性设计上，操作被显式分类：离线出示与验证不应受供应商宕机影响，签发、在线验证与信任列表同步可依赖后端；离线验证器缓存 Document Signer 证书与吊销列表，凭证过期靠 mSO 的 validFrom/validUntil 时间戳在本地判定。全文以六维供应商尽调清单（密码架构、信任与 PKI、隐私、离线、互操作、运维韧性）和 Build/Buy/Govern 决策模型收尾。

**「踩坑与收获」** 作者提醒两条可直接用于选型的经验：一是供应商 SLA 再高也可能构成架构级单点故障——只要每笔验证都要求其在线可达，因此评测必须脱离高速网络下的演示环境，实测断网出示与验证、缓存信任材料、过期凭证、陈旧吊销信息、设备时钟异常及供应商宕机等失败路径，而非只看 happy path；二是供应商若只宣称&quot;加密通信&quot;却讲不清密钥边界、生命周期与轮换机制，等于制造虚假安全感。

**标签**: `#ISO/IEC 18013-5`, `#mDL`, `#PKI证书体系`, `#HPKE`, `#选择性披露`

---

<a id="item-tech-top5-4"></a>
### [dm-thin 块复用残留致跨租户泄露](https://www.infoq.cn/article/YmllSwKbZXY0bcVjjw70?utm_source=rss&amp;utm_medium=article) ⭐️ 6.0/10

rss · InfoQ 中文站 · 10月18日 16:00

**「技术方案」** Cloudflare Containers 中每个容器运行于 Firecracker microVM，可写根磁盘由 Linux device mapper 精简配置提供；受影响资源池使用 64 KiB 精简块并开启 skip\_block\_zeroing，即 dm-thin 在暴露新分配块前不做清零。问题出在写入粒度与物理块粒度的错配：向未映射区写入单个 4 KiB 块会促使 dm-thin 从跨租户共享池分配一个 64 KiB 物理块，写入仅覆盖 4 KiB，最多 60 KiB 仍残留前租户数据，原始读取即可取回新容器从未写入的字节。研究员 Oren Yomtov 利用 ext4 目录块校验和区分自写块与他人块，据其披露：六个生产部署中 5614 个可测目录块全部来自其他用户，识别出 2700 个外部 inode；四大洲 24 个部署点中 18 个、22 个底层节点中 20 个发现残留，成功恢复目录结构、数据库页乃至结构完整的 SQLite 数据库。修复在报告当日（9 月 4 日）即合并运行时修复并连夜分批部署、9 月 7 日全量完成，核心是移除 skip\_block\_zeroing 使新分配块恢复默认清零；但存量已映射块（含运行中容器磁盘与各主机 OCI 镜像层快照缓存）不会被动清理，新容器可能直接继承这些映射，因此 Cloudflare 停用全部运行容器磁盘、非高峰时段清空主机数据、重启虚拟机并逐台清空镜像缓存，9 月 19 日完成清理；同时以“先小写、后大读”特征模式构建检测签名回扫历史磁盘 I/O 遥测，仅命中研究员与其自家工程师的授权验证活动。

**「踩坑与收获」** 最直接的坑是“改配置不等于清存量”：关闭 skip\_block\_zeroing 只影响新分配块，存量映射（尤其各主机的镜像层快照缓存）仍会被新容器继承，必须配套停用磁盘、擦除主机数据与缓存才算闭环。可迁移的教训是存储层的租户隔离必须单独验证，而非停留在架构假设：对任何容器或无服务器平台，都应检查复用卷路径在分配时是否清零，并将共享存储上的残留数据视为首要隔离控制项；评估此类风险时也要纳入 Cloudflare 坦承的限制——攻击者无法选择受害者、工作负载或主机，残留数据是否存在并无保证。

**标签**: `#Cloudflare Containers`, `#Firecracker`, `#dm-thin 精简配置`, `#多租户隔离`, `#跨租户数据泄露`

---

<a id="item-tech-top5-5"></a>
### [816B MoE 模型如何搬上桌面](https://www.infoq.cn/article/EuYOPRUBAeBgBGBu5ktk?utm_source=rss&amp;utm_medium=article) ⭐️ 5.0/10

rss · InfoQ 中文站 · 10月18日 23:01

**「技术方案」** Meta 于 10 月 15 日开放 Muse Spark Open 权重：稀疏 MoE 架构，总参数 8160 亿、每 token 激活约 420 亿，支持约 105 万 token 上下文与文本、图像、音频、视频输入，主打软件工程与长程智能体任务。本地部署靠量化加多机并行：BF16 权重约 1.65TB、NVFP4 约 537GB（另需预留 KV Cache 等开销），GPU 参考配置为 BF16 用 8 张 B300、NVFP4 用 4 张 B300；两台 M5 Ultra Mac Studio 则经 MLX 框架和 Thunderbolt RDMA 跑 NVFP4 版本，苹果公布四台 Mac Studio 分布式推理生成速度超 36 token/s。Meta 公布的成绩中 Terminal-Bench 2.1 得 76.4、DeepSWE 1.1 得 60，低于 GLM 5.3 max 和 Kimi K3 max；集成于 SGLang 的 Helion 定制算子宣称在 NVIDIA/AMD 上有 1.03—3.95 倍算子级加速，AMD MI350 端到端仅约 3%。许可证规定提供模型即服务的企业年收入超 2000 万美元需通知 Meta 并另签书面协议，内部使用、研究等场景例外。

**「踩坑与收获」** 容量规划切勿按激活参数估算：MoE 每个 token 虽只激活 420 亿参数，全部专家权重仍需驻留内存，两台 512GB Mac 装不下 BF16 版本，只能靠 NVFP4 量化压到 537GB 才可行。另需区分算子级与端到端加速、单机统一内存带宽（M5 Ultra 为 1.2TB/s）与多机通信带宽——机器增多后跨设备同步与数据交换可能抵消扩展收益。

**标签**: `#MoE架构`, `#NVFP4量化`, `#MLX`, `#本地推理部署`, `#开源模型许可证`

---