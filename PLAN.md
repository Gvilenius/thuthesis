# 论文修改计划

## 项目概述

本文是一篇博士学位论文，基于三篇原始论文工程整合而成：
1. **VoltanaLLM/GreenServe** - 大模型推理能效优化
2. **WeiPipe** - 长上下文训练通信优化
3. **CATFlow** - 自动并行策略发现框架

## 修改状态：已完成 ✅

---

## 已完成的修改

### 第一阶段：GreenServe章节完善（第3章）✅

#### 3.1 观察与发现部分
- [x] 添加U型能耗-频率曲线图（figs/u-shaped-intro.pdf）
- [x] 添加Prefill/Decode阶段分离的能耗对比图
- [x] 添加能效阶梯现象的详细图表（figs/energy_tile.pdf）
- [x] 补充GEMM tile量化效应图（figs/voltanallm-gemm-tile.drawio.pdf）
- [x] 添加多时间尺度负载动态性图表（figs/TPS_IN.pdf, figs/TPS_OUT.pdf）
- [x] 补充Prefill负载波动图（figs/prefill-load-bs-time.pdf）

#### 3.2 问题形式化部分
- [x] 完善约束优化问题的数学表述
- [x] 添加控制理论视角的分析

#### 3.3 系统设计部分
- [x] 添加系统架构图（figs/voltanallm-overall-arch.drawio.pdf）
- [x] 补充EcoFreq控制器详细设计（figs/voltanallm-governor.drawio.pdf）
- [x] 补充EcoRoute路由器设计图（figs/router_motivation.pdf）
- [x] 添加状态空间引导路由的详细说明
- [x] 补充延迟预测器特征选择和在线自适应的详细内容

#### 3.4 实验评估部分
- [x] 添加主要结果对比图（figs/main_result_combined.pdf）
- [x] 添加不同SLO设置下的性能对比（figs/ablation-slo.pdf）
- [x] 添加频率控制时间窗口分析（figs/ablation-interval.pdf）
- [x] 添加预测器精度分析（figs/prediction_error.pdf）
- [x] 添加GH200泛化性验证结果（figs/gh200_result.pdf）
- [x] 添加各模块贡献分析（figs/ablation_w_router-2p2d.pdf）
- [x] 添加实时频率跟踪图（figs/ablation-freq-time-combined.pdf）

---

### 第二阶段：WeiPipe章节完善（第4章）✅

#### 4.1 问题分析部分
- [x] 添加激活值与权重通信量对比图（figs/fig1.png）
- [x] 补充临界条件推导的详细公式

#### 4.2 权重流水并行机制部分
- [x] 添加WeiPipe-Naive流程图（figs/fig2.png）
- [x] 补充前向/反向/更新阶段的详细描述

#### 4.3 高级调度策略部分
- [x] 添加WeiPipe-Interleave调度示意图（figs/fig3.png）
- [x] 添加WeiPipe-zero-bubble调度示意图（figs/fig4.png）

#### 4.4 实验评估部分
- [x] 添加吞吐量对比图表（figs/fig5.png）
- [x] 添加不同序列长度下的性能对比（figs/exp_fig3.png）
- [x] 添加可扩展性分析图表（figs/exp_fig4.png）

---

### 第三阶段：CATFlow章节完善（第5章）✅

#### 5.1 数据一元论抽象体系部分
- [x] 添加CAT抽象示意图
- [x] 补充轨迹向量表示的详细说明

#### 5.2 并行计划发现部分
- [x] 添加ZeRO策略对比图

#### 5.3 实验评估部分
- [x] 完善新型ZeRO策略性能分析
- [x] 补充搜索过程演化分析

---

### 第四阶段：总结与展望章节完善（第6章）✅

#### 6.1 研究工作总结
- [x] 系统总结三项核心工作的贡献
- [x] 阐述三项工作之间的内在联系
- [x] 强调研究的整体价值

#### 6.2 研究局限性
- [x] 分析GreenServe的局限性（硬件依赖性、负载假设、SLO设置）
- [x] 分析WeiPipe的局限性（适用场景、实现复杂度、框架集成）
- [x] 分析CATFlow的局限性（搜索空间、模拟精度、泛化能力）

#### 6.3 未来研究方向
- [x] 能效优化方向：更广泛的硬件支持、训练阶段能效优化、跨层协同优化
- [x] 通信优化方向：与更多并行策略结合、异构网络环境适配、通信-计算更紧密协同
- [x] 自动并行方向：更丰富的策略空间、更高效的搜索算法、在线自适应优化
- [x] 跨领域融合：训练与推理统一优化框架、模型压缩与并行策略联合优化、边缘计算场景

---

### 第五阶段：图片和引用完善 ✅

#### 图片处理
- [x] 复制VoltanaLLM论文图片到data/figures/
- [x] 复制WeiPipe论文图片到data/figures/
- [x] 检查所有图片引用路径

---

## 修改总结

| 章节 | 图表补充 | 内容扩充 | 状态 |
|------|----------|----------|------|
| 第3章 GreenServe | 15+ | 约30% | ✅ 完成 |
| 第4章 WeiPipe | 5+ | 约25% | ✅ 完成 |
| 第5章 CATFlow | 2+ | 约15% | ✅ 完成 |
| 第6章 总结与展望 | - | 约200% | ✅ 完成 |

---

## 完成日期

2026-04-09
