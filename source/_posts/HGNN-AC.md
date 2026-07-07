---
title: 【论文阅读】Heterogeneous Graph Neural Network via Attribute Completion
date: 2022-11-04 14:43:26
tags: 异质图学习
categories: 
    - 深度学习
mathjax: true
---

![](/img/HGNN-AC_fig0.png)

**论文标题**：基于属性补全的异质图神经网络
**论文链接**：https://doi.org/10.1145/3442381.3449914
**发表会议**：WWW2021
<!--more-->

## 一、概述
异质图拥有多种类型的节点和边，相比于同质图包含更多的信息，能够更好的描述现实世界。已经有许多研究采用GNN的方法针对异质图进行研究，取得了很好的成果。但这些基于GNN的异质图模型需要所有的节点都具有属性特征。现实中，这种要求并不能够满足。

因为某些原因，一些节点并没有属性信息，例如带有个人敏感信息。在异质图中更加明显，我们通常不能得到所有类型节点的属性特征，这也影响到了GNN-based模型的性能。

我们将异质图的属性缺失分为两类:
1. 需要被分析的节点属性缺失，例如下图中DBLP数据集的author类型节点；
2. 不需要被分析的节点属性缺失，例如下图中IMDB数据集的actor类型节点。

![](/img/HGNN-AC_fig1.png)

上图中只有DBLP的paper类型节点和IMDB的movie类型节点有属性，其余节点的属性缺失，我们需要对图中有颜色的节点做预测。虽然有些节点没有属性，但是这些没有属性的节点会连接到有属性的节点。以往的研究都是采用手工的方式处理异质图中属性缺失的问题。在代码中，往往是采用一个one-hot向量来表示，这会提供一些无效的信息。

因此，本文提出一种**节点属性补全**的模型HGNN-AC。**本文以节点间的拓扑关系为指导，通过对属性节点的属性进行加权聚合来补全无属性节点的属性**。

HGNN-AC首先使用HIN-Embedding的方法来通过节点的拓扑关系来获取节点embedding，然后在进行加权聚合时通过attention机制来计算节点embedding之间的注意力系数，来区分不同节点的贡献。这种补全机制可以轻松的和现有的HIN模型结合，实现端到端训练。

## 二、概念
### 1.异质图
一个异质图表示为

{% katex %}
\mathcal{G}(\mathcal{V}, \mathcal{E}, F, R, \varphi, \phi)
{% endkatex %}

其中 {% katex %}\mathcal{V}{% endkatex %} 和 {% katex %}\mathcal{E}{% endkatex %} 分别表示节点和边的集合，{% katex %}F{% endkatex %} 和 {% katex %}R{% endkatex %} 分别代表节点类型集合和边的类型集合，满足 {% katex %}|F|+|R|>2{% endkatex %}。
每个节点 {% katex %}i \in \mathcal{V}{% endkatex %} 关联一个节点类型映射函数 {% katex %}\varphi: \mathcal{V} \rightarrow F{% endkatex %}，每条边 {% katex %}e \in \mathcal{E}{% endkatex %} 关联一个边类型映射函数 {% katex %}\phi: \mathcal{E} \rightarrow R{% endkatex %}。

### 2.异质图中的不完全属性
给定一个异质图 {% katex %}\mathcal{G}(\mathcal{V}, \mathcal{E}, F, R, \varphi, \phi){% endkatex %}，{% katex %}X{% endkatex %} 代表节点属性。节点属性不完全意味着 {% katex %}\exists F^{\prime} \subset F,\ F^{\prime} \neq \varnothing{% endkatex %}，其中所有满足 {% katex %}\varphi(v) \in F'{% endkatex %} 的节点 {% katex %}v{% endkatex %} 均无原始属性。

本文限定场景：同一类型节点属性状态完全统一，要么全部有属性，要么全部无属性，不存在单类型内部分节点缺失的情况。

### 3.异质图Embedding
给定一个异质图 {% katex %}\mathcal{G}{% endkatex %}，任务是学习一个 {% katex %}d{% endkatex %} 维的节点表示

{% katex %}
h_{v} \in \mathbb{R}^{d},\quad v \in \mathcal{V}
{% endkatex %}

来捕捉 {% katex %}\mathcal{G}{% endkatex %} 丰富的结构和语义信息，其中 {% katex %}d \ll |\mathcal{V}|{% endkatex %}。

下表总结了本文所使用的符号表示：
![](/img/HGNN-AC_fig2.png)

## 三、方法
HGNN-AC遵循的原则：无属性节点 {% katex %}v \in \mathcal{V}^{-}{% endkatex %} 的生成属性全部来自有属性节点集合 {% katex %}\mathcal{V}^{+}{% endkatex %}。核心思路：利用拓扑信息作为指导，计算无属性节点 {% katex %}v{% endkatex %} 的一阶有属性邻居 {% katex %}u \in \mathcal{N}_{v}^{+}{% endkatex %} 的贡献权重，加权聚合完成属性补全。

### 1. 整体流程概述
给定异质图 {% katex %}\mathcal{G}{% endkatex %}，完整流程分为五步：
1. 基于图拓扑邻接矩阵 {% katex %}A{% endkatex %} 预学习全部节点拓扑Embedding {% katex %}H{% endkatex %}；
2. 通过注意力机制，依托拓扑Embedding计算邻居贡献得分；
3. 根据得分聚合 {% katex %}\mathcal{V}^{+}{% endkatex %} 节点原始属性，补全 {% katex %}\mathcal{V}^{-}{% endkatex %} 节点缺失特征；
4. 随机丢弃部分 {% katex %}\mathcal{V}^{+}{% endkatex %} 节点属性，复用补全流程重建特征，构造监督补全损失；
5. 将属性完备的新图送入下游HIN模型，联合优化「任务预测损失+属性重建损失」，全程端到端训练。

![](/img/HGNN-AC_fig3.png)

### 2.拓扑Embedding的预学习
在异质图中，节点拓扑结构与属性语义存在同质性：拓扑相近的节点，属性分布也高度相似。因此拓扑Embedding可以作为属性关联的先验信息。

HGNN-AC改进传统metapath2vec仅使用单一路径的缺陷：基于多条元路径随机游走生成节点序列，再送入Skip-Gram模型训练拓扑表征，捕捉更全面的异质语义关联。

### 3.使用attention机制进行属性补全
传统均值聚合无法区分邻居节点的贡献差异（局部拓扑密度不同，邻居重要性存在区分度），本文引入掩码多头注意力，仅对有属性邻居计算权重。

给定相连节点对 {% katex %}(v, u){% endkatex %}，{% katex %}v{% endkatex %} 为待补全节点，{% katex %}u \in \mathcal{N}_{v}^{+}{% endkatex %} 为有属性邻居，注意力原始得分：

{% katex %}
e_{vu} = \sigma\left(h_{v}^{T} W h_{u}\right)
{% endkatex %}

其中 {% katex %}h_{v}{% endkatex %}、{% katex %}h_{u}{% endkatex %} 为拓扑Embedding，{% katex %}W{% endkatex %} 为可学习权重矩阵，{% katex %}\sigma{% endkatex %} 为激活函数。

仅对 {% katex %}u \in \mathcal{N}_{v}^{+}{% endkatex %} 的得分做Softmax归一化：

{% katex %}
a_{vu} = \operatorname{softmax}\left(e_{vu}\right) = \frac{\exp\left(e_{vu}\right)}{\sum_{s \in \mathcal{N}_{v}^{+}} \exp\left(e_{vs}\right)}
{% endkatex %}

根据归一化权重加权聚合邻居属性，得到补全特征：

{% katex %}
x_{v}^{C} = \sum_{u \in \mathcal{N}_{v}^{+}} a_{vu} x_{u}
{% endkatex %}

若节点无任何有属性邻居，即 {% katex %}\mathcal{N}_{v}^{+} = \varnothing{% endkatex %}，补全向量置零向量，该场景真实数据集极少出现，对性能影响可忽略。

为降低异质图表征高方差、稳定训练过程，引入多头注意力并行聚合，最终特征取多头均值：

{% katex %}
x_{v}^{C} = \operatorname{mean}\left( \sum_{k=1}^{K} \sum_{u \in \mathcal{N}_{v}^{+}} a_{vu}^{k} x_{u} \right)
{% endkatex %}

### 4.已有属性的Drop与重构监督
为让补全过程具备监督信号，设计属性丢弃重建策略：
1. 以超参数 {% katex %}\alpha{% endkatex %} 划分有属性节点集 {% katex %}\mathcal{V}^{+}{% endkatex %}：丢弃子集 {% katex %}\mathcal{V}_{drop}^{+}{% endkatex %}、保留子集 {% katex %}\mathcal{V}_{keep}^{+}{% endkatex %}，满足 {% katex %}|\mathcal{V}_{drop}^{+}|=\alpha \cdot |\mathcal{V}^{+}|{% endkatex %}；
2. 清空 {% katex %}\mathcal{V}_{drop}^{+}{% endkatex %} 节点原始属性，复用注意力补全逻辑，依靠 {% katex %}\mathcal{V}_{keep}^{+}{% endkatex %} 重建特征；
3. 使用欧式距离衡量重建偏差，构造补全损失函数：

{% katex %}
\mathcal{L}_{completion} = \frac{1}{\left|\mathcal{V}_{drop}^{+}\right|} \sum_{i \in \mathcal{V}_{drop}^{+}} \sqrt{\left(X_{i}^{C} - X_{i}\right)^2}
{% endkatex %}

### 5.与HIN模型联合训练
补全后全局完整节点属性矩阵定义：

{% katex %}
X^{new}= \left\{ X_i^C,\ X_j \mid \forall i \in \mathcal{V}^{-},\ \forall j \in \mathcal{V}^{+} \right\}
{% endkatex %}

将完整属性 {% katex %}X^{new}{% endkatex %} 与拓扑邻接矩阵 {% katex %}A{% endkatex %} 输入下游异质图主干模型 {% katex %}\Phi{% endkatex %}，得到预测结果并计算任务损失：

{% katex %}
\begin{aligned}
\tilde{Y} &= \Phi \left(A,\ X^{new}\right) \\
\mathcal{L}_{prediction} &= f\left(\tilde{Y},\ Y \right)
\end{aligned}
{% endkatex %}

整体联合损失加权融合两项损失，{% katex %}\lambda{% endkatex %} 为平衡超参数：

{% katex %}
\mathcal{L}=\lambda \cdot \mathcal{L}_{completion} + \mathcal{L}_{prediction}
{% endkatex %}

## 四、实验
### 1.数据集
采用DBLP，ACM和IMDB三组经典异质图数据集，覆盖两类属性缺失场景，统计信息如下：
![](/img/HGNN-AC_fig4.png)

### 2.整体对比实验
实验分为两大场景：
1. 预测目标节点无原始属性：DBLP数据集；
2. 仅预测目标节点有属性，其余类型全部缺失：ACM、IMDB数据集。

基线对比结果：
![](/img/HGNN-AC_fig5.png)
![](/img/HGNN-AC_fig6.png)

基于GTN的消融对比：开启HGNN-AC属性补全后基线性能稳定提升，验证补全机制有效性。
![](/img/HGNN-AC_fig7.png)

### 3.节点Embedding可视化
采用t-SNE降维可视化模型学习到的节点表征，HGNN-AC产出的特征聚类边界更清晰：
![](/img/HGNN-AC_fig8.png)

### 4.消融实验（不同填充策略对比）
基于MAGNN基线设置5组对照，验证本文补全方案优于传统One-Hot、均值填充：
- **MAGNN**：辅助节点特征采用直接相连paper节点均值聚合；
- **MAGNN-onehot1**：author节点One-Hot填充，subject节点均值聚合；
- **MAGNN-onehot2**：author、subject全部使用One-Hot填充；
- **MAGNN-AC1**：author使用HGNN-AC补全，subject One-Hot；
- **MAGNN-AC2**：author、subject均采用HGNN-AC补全。

实验结果：
![](/img/HGNN-AC_fig9.png)

### 5.超参数敏感性实验
调整属性丢弃比例、注意力头数等超参数观测性能波动，模型在合理区间内性能稳定：
![](/img/HGNN-AC_fig10.png)