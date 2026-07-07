---
title: 【论文阅读】Metapath Aggregated Graph Neural Network for Heterogeneous Graph Embedding
date: 2022-12-14 10:40:25
tags: 异质图学习
categories: 
    - 深度学习
mathjax: true
---

![](/img/MAGNN_fig0.png)

**论文标题**：MAGNN: 用于异构图嵌入的元路径聚合图神经网络
**论文链接**：https://arxiv.org/pdf/2002.01680.pdf
**代码链接**：https://github.com/cynricfu/MAGNN
**发表会议**：WWW 2020
<!--more-->

## 引言
本文提出MAGNN模型，围绕元路径的聚合问题展开。元路径聚合主要有两部分，元路径内部的聚合、元路径之间的聚合。同时，在建模过程中结合了节点的属性信息。

现有的基于元路径的嵌入学习方法有以下的局限性：
1. **忽略了节点的属性信息**，不能很好的处理节点属性特征丰富的异质图。例如 metapath2vec, ESim, HIN2vec, HERec 。
2. **舍弃了元路径内部的节点信息**，只考虑元路径的起始节点和末尾节点，造成信息损失。例如 HERec, HAN 。
3. **只依赖于单个路径**，因此需要人工选择元路径，丢失了来自其他元路径的部分信息，导致性能不佳。例如 metapath2vec 。

为解决上述问题，本文提出 MAGNN: Metapath Aggregated Graph Neural Network for Heterogeneous Graph Embedding

## 定义和符号
异质图、元路径、元路径实例不在赘述。
相关图示如下：
![](/img/MAGNN_fig1.png)

### Metapath-based Neighbor
给定异质图中的一个元路径 {% katex %}P{% endkatex %} ，节点 {% katex %}v{% endkatex %} 的 metapath-based 邻居 {% katex %}\mathcal{N}^P_v{% endkatex %} 为和 {% katex %}v{% endkatex %} 相连的遵循元路径 {% katex %}P{% endkatex %} 模式的节点集合。
由两个不同的元路径实例与 {% katex %}v{% endkatex %} 相连的同一个邻居节点，被视为 {% katex %}\mathcal{N}^P_v{% endkatex %} 中的两个节点。另外，如果元路径 {% katex %}P{% endkatex %} 是对称的，{% katex %}\mathcal{N}^P_v{% endkatex %} 中也包含节点 {% katex %}v{% endkatex %} 自身。

### Metapath-based Graph
给定异质图 {% katex %}\mathcal{G}{% endkatex %} 中的元路径 {% katex %}P{% endkatex %} ，基于元路径的图 {% katex %}\mathcal{G}^P{% endkatex %} 由原始图 {% katex %}\mathcal{G}{% endkatex %} 中所有的基于元路径 {% katex %}P{% endkatex %} 的邻居点对组成（去掉了元路径实例的中间节点，只保留元路径实例两端的节点，并在两点间建立连边）。
如果元路径 {% katex %}P{% endkatex %} 是对称的，则 {% katex %}\mathcal{G}^P{% endkatex %} 是同质图。如上图(d)所示。

### 异质图嵌入
给定 {% katex %}\mathcal{G}=(\mathcal{V}, \mathcal{E}){% endkatex %} 和节点属性矩阵 {% katex %}\mathbf{X}_{A_i} \in \mathbb{R}^{ | \mathcal{V}_{A_i} | \times d_{A_i} }{% endkatex %} ，其中 {% katex %}A_i{% endkatex %} 表示节点类型。
异质图嵌入学习的目的是从图中捕获丰富的结构信息和语义信息，为每个节点学习 {% katex %}d{% endkatex %} 维表示 {% katex %}\mathbf{h}_v\in \mathbb{R}^d{% endkatex %}。

![](/img/MAGNN_fig2.png)

## 方法
MAGNN由**节点内容转换**、**元路径内部聚合**、**元路径间的聚合**三部分组成，下图展示单个节点的嵌入生成流程。
![](/img/MAGNN_fig3.png)

### Node Content Transformation
异质图内不同类型节点拥有不同属性，特征向量维度、特征空间均不统一。
为统一处理，需要将各类节点特征映射至同一个隐层向量空间。对类别为 {% katex %}A\in \mathcal{A}{% endkatex %} 的节点 {% katex %}v\in \mathcal{V}_{A}{% endkatex %}，转换公式：
{% katex %}
\mathbf{h}_{v}^{\prime}=\mathbf{W}_{A} \cdot \mathbf{x}_{v}^{A}
{% endkatex %}
其中 {% katex %}x_v\in \mathbb{R}^{d_A}{% endkatex %} 为原始特征向量，{% katex %}\mathbf{h}^{\prime}_v\in \mathbb{R}^{d^{\prime}}{% endkatex %} 是映射后节点特征；{% katex %}W_A\in \mathbb{R}^{d^{\prime}\times d_A}{% endkatex %} 是类型 {% katex %}A{% endkatex %} 专属可学习权重矩阵。

### Intra-metapath Aggregation
给定元路径 {% katex %}P{% endkatex %}，元路径内部聚合层对元路径实例编码，挖掘目标节点、元路径邻居、中间上下文蕴含的结构与语义信息。

定义连接目标节点 {% katex %}v{% endkatex %} 与元路径邻居 {% katex %}u\in \mathcal{N}^P_v{% endkatex %} 的完整元路径实例为 {% katex %}P(v, u){% endkatex %}。
元路径实例内部节点集合：{% katex %}\{m^{P(v, u)}\}=P(v, u)\setminus \{u, v\}{% endkatex %}。

#### 1. 元路径实例编码器
将整条元路径实例内所有节点特征编码为固定维度向量 {% katex %}\mathbf{h}_{P(v,u)}\in \mathbb{R}^{d^{\prime}}{% endkatex %}：
{% katex %}
\mathbf{h}_{P(v, u)}=f_{\theta}(P(v, u))=f_{\theta}\left(\mathbf{h}_{v}^{\prime}, \mathbf{h}_{u}^{\prime},\left\{\mathbf{h}_{t}^{\prime}, \forall t \in \left\{m^{P(v, u)}\right \}\right \} \right)
{% endkatex %}
节点 {% katex %}v, u{% endkatex %} 之间可存在多条元路径实例，后文提供三种编码器实现方案。

#### 2. 元路径内部多头注意力加权聚合
针对同一元路径 {% katex %}P{% endkatex %} 下多条元路径实例做注意力加权：
{% katex %}
\begin{aligned}
e_{v u}^{P} & =\operatorname{LeakyReLU}\left(\mathbf{a}_{P}^{\top} \cdot\left[\mathbf{h}_{v}^{\prime} \| \mathbf{h}_{P(v, u)}\right]\right), \\
\alpha_{v u}^{P} & =\frac{\exp \left(e_{v u}^{P}\right)}{\sum_{s \in \mathcal{N}_{v}^{P}} \exp \left(e_{v s}^{P}\right)}, \\
\mathbf{h}_{v}^{P} & =\sigma\left(\sum_{u \in \mathcal{N}_{v}^{P}} \alpha_{v u}^{P} \cdot \mathbf{h}_{P(v, u)}\right) .
\end{aligned}
{% endkatex %}
其中 {% katex %}a_P\in \mathbb{R}^{2d^{\prime}}{% endkatex %} 为元路径 {% katex %}P{% endkatex %} 专属注意力向量；{% katex %}e^P_{vu}{% endkatex %} 代表元路径实例 {% katex %}P(v,u){% endkatex %} 对目标节点 {% katex %}v{% endkatex %} 的重要程度，经Softmax归一化后加权求和，最后送入激活函数。

扩展多头注意力稳定训练、降低异质图高方差，多头拼接公式：
{% katex %}
\mathbf{h}_{v}^{P}=\|_{k=1}^{K} \sigma\left(\sum_{u \in \mathcal{N}_{v}^{P}}\left[\alpha_{v u}^{P}\right]_{k} \cdot \mathbf{h}_{P(v, u)}\right)
{% endkatex %}

#### 小节总结
输入映射后特征 {% katex %}h^{\prime}_u\in \mathbb{R}^{d^{\prime}}, \forall u\in \mathcal{V}{% endkatex %} 与元路径集合 {% katex %}\mathcal{P}_A=\{P_1, P_2, ..., P_M\}{% endkatex %}，内部聚合为节点 {% katex %}v{% endkatex %} 生成 {% katex %}M{% endkatex %} 条分语义向量：{% katex %}\{h^{P_1}_v,h^{P_2}_v, ..., h^{P_M}_v\}{% endkatex %}，每条 {% katex %}h^{P_i}_v\in \mathbb{R}^{d^{\prime}}{% endkatex %} 对应一种元路径语义。

### Inter-metapath Aggregation
元路径间聚合层融合多条元路径的语义表征。
对类型 {% katex %}A{% endkatex %} 的节点，拥有 {% katex %}M{% endkatex %} 组分语义向量 {% katex %}\{h^{P_1}_v, h^{P_2}_v, ..., h^{P_M}_v\}, v\in \mathcal{V}_A{% endkatex %}，使用跨元路径注意力分配各语义权重。

1. 对每条元路径 {% katex %}P_i\in \mathcal{P}_A{% endkatex %}，全局平均聚合该元路径下全部同类型节点表征：
{% katex %}
\mathbf{s}_{P_{i}}=\frac{1}{\left|\mathcal{V}_{A}\right|} \sum_{v \in \mathcal{V}_{A}} \tanh \left(\mathbf{M}_{A} \cdot \mathbf{h}_{v}^{P_{i}}+\mathbf{b}_{A}\right)
{% endkatex %}
{% katex %}M_A\in \mathbb{R}^{d_m\times d^{'}}, b_A\in \mathbb{R}^{d_m}{% endkatex %} 为可学习参数。

2. 注意力加权融合多条元路径表征：
{% katex %}
\begin{array}{l}
e_{P_{i}}=\mathbf{q}_{A}^{\top} \cdot \mathbf{s}_{P_{i}}, \\
\beta_{P_{i}}=\frac{\exp \left(e_{P_{i}}\right)}{\sum_{P \in \mathcal{P}_{A}} \exp \left(e_{P}\right)}, \\
\mathbf{h}_{v}^{\mathcal{P}_{A}}=\sum_{P \in \mathcal{P}_{A}} \beta_{P} \cdot \mathbf{h}_{v}^{P}
\end{array}
{% endkatex %}
{% katex %}q_A\in \mathbb{R}^{d_m}{% endkatex %} 是类型 {% katex %}A{% endkatex %} 专属注意力向量，{% katex %}\beta_{P_i}{% endkatex %} 代表元路径 {% katex %}P_i{% endkatex %} 对该类节点的语义重要性。

3. 输出层线性映射，得到节点最终嵌入：
{% katex %}
\mathbf{h}_{v}=\sigma\left(\mathbf{W}_{o} \cdot \mathbf{h}_{v}^{\mathcal{P}_{A}}\right)
{% endkatex %}
{% katex %}W_o\in \mathbb{R}^{d_o\times d^{\prime}}{% endkatex %} 为输出权重矩阵，{% katex %}\sigma(\cdot){% endkatex %} 是非线性激活函数。
该输出层可适配节点分类（线性分类器）、链接预测（相似度度量投影）等下游任务。

### Metapath Instance Encoders
针对元路径编码函数 {% katex %}f_{\theta}{% endkatex %}，作者提供三种可选实现：

1. Mean encoder 均值编码器
{% katex %}
\mathbf{h}_{P(v, u)}=\operatorname{MEAN}\left(\left\{\mathbf{h}_{t}^{\prime}, \forall t \in P(v, u)\right\}\right)
{% endkatex %}

2. Linear encoder 线性均值编码器（均值+线性变换）
{% katex %}
\mathbf{h}_{P(v, u)}=\mathbf{W}_{P} \cdot \operatorname{MEAN}\left(\left\{\mathbf{h}_{t}^{\prime}, \forall t \in P(v, u)\right\}\right)
{% endkatex %}

3. Relational rotation encoder 关系旋转编码器
借鉴知识图谱RotatE的复空间关系旋转建模，保留元路径序列顺序信息。
设元路径实例 {% katex %}P(v, u)=(t_0, t_1, ..., t_n), t_0=u, t_n=v{% endkatex %}，{% katex %}R_i{% endkatex %} 为节点 {% katex %}t_{i-1}, t_i{% endkatex %} 间关系，{% katex %}\mathbf{r}_i{% endkatex %} 为关系向量：
{% katex %}
\begin{array}{l}
\mathbf{o}_{0}=\mathbf{h}_{t_{0}}^{\prime}=\mathbf{h}_{u}^{\prime} \\
\mathbf{o}_{i}=\mathbf{h}_{t_{i}}^{\prime}+\mathbf{o}_{i-1} \odot \mathbf{r}_{i}, \\
\mathbf{h}_{P(v, u)}=\frac{\mathbf{o}_{n}}{n+1}
\end{array}
{% endkatex %}
{% katex %}\mathbf{h}^{\prime}_{t_i}, \mathbf{r}_i{% endkatex %} 为复数向量，{% katex %}\odot{% endkatex %} 代表逐元素乘积；最终取实部、虚部分别对应前后一半维度作为 {% katex %}d^{\prime}{% endkatex %} 维实数表征。

MAGNN完整前向传播算法流程图：
![](/img/MAGNN_fig4.png)

### 模型训练
完成三层聚合后得到节点嵌入，分半监督、无监督两种训练范式适配不同任务标签场景。

#### 1. 半监督学习（节点存在标注）
最小化分类交叉熵损失：
{% katex %}
\mathcal{L} = -\sum_{v \in \mathcal{V}_{L}} \sum_{c=1}^{C} \mathrm{y}_{v}[c]\cdot \log{h_v}[c]
{% endkatex %}
{% katex %}\mathcal{V}_L{% endkatex %} 为带标签节点集合，{% katex %}C{% endkatex %} 是类别总数，{% katex %}\mathbf{y}_v{% endkatex %} 是节点 {% katex %}v{% endkatex %} 的one-hot真实标签，{% katex %}\mathbf{h}_v{% endkatex %} 为模型预测输出。

#### 2. 无监督学习（无节点标注，负采样）
正负样本对比损失：
{% katex %}
\mathcal{L} = - \sum_{(u,v) \in \Omega} \log \sigma (\mathbf{h}_u^\mathsf{T} \cdot \mathbf{h}_v) - \sum_{(u^{\prime},v^{\prime}) \in \Omega ^ {-}} \log \sigma ( - \mathbf{h}_{u^{\prime}}^\mathsf{T} \cdot \mathbf{h}_{v^{\prime}})
{% endkatex %}
{% katex %}\Omega{% endkatex %} 为正样本节点对集合，{% katex %}\Omega^{-}{% endkatex %} 为负采样节点对集合。

## 实验
### 数据集
使用IMDB、DBLP、Last.fm三类异质图数据集，统计信息如下：
![](/img/MAGNN_fig5.png)

### 下游任务实验结果
1. 节点分类
![](/img/MAGNN_fig6.png)
2. 节点聚类
![](/img/MAGNN_fig7.png)
3. 链路预测
![](/img/MAGNN_fig8.png)

### 消融实验
对照组说明：
- {% katex %}\mathrm{MAGNN} _{rot}{% endkatex %}：使用Relational rotation编码器完整模型；
- {% katex %}\mathrm{MAGNN} _{feat}{% endkatex %}：移除节点原始属性输入；
- {% katex %}\mathrm{MAGNN} _{nb}{% endkatex %}：仅使用元路径邻居均值，无内部实例聚合；
- {% katex %}\mathrm{MAGNN} _{sm}{% endkatex %}：仅使用单一最优元路径，无跨元路径聚合；
- {% katex %}\mathrm{MAGNN} _{avg}{% endkatex %}：Mean均值编码器；
- {% katex %}\mathrm{MAGNN} _{linear}{% endkatex %}：Linear线性均值编码器。

消融实验结果图：
![](/img/MAGNN_fig9.png)

结论：
1. 对比 {% katex %}\mathrm{MAGNN} _{nb}{% endkatex %} 与三种编码器变体，**元路径内部实例聚合能显著提升效果**；
2. 对比 {% katex %}\mathrm{MAGNN} _{sm}{% endkatex %} 与完整模型，**多元路径注意力融合模块有效**；
3. 三种编码器中Relational rotation效果最优；全部变体均优于基线HAN。

### 节点表征t-SNE可视化
![](/img/MAGNN_fig10.png)