---
title: weekly report 2020-10-18
date: 2020-10-18 08:41:03
tags:
    - report
categories:
    - note
mathjax: true
---
- [华为比赛](#华为比赛)
- [知识图谱研究](#知识图谱研究)
  - [Formulation](#formulation)
  - [TransE](#transe)
  - [TF-IDF](#tf-idf)
  - [Concatenate](#concatenate)
  - [Experiments](#experiments)
    - [条件说明](#条件说明)
    - [实验结果](#实验结果)
    - [对比分析](#对比分析)
  - [想法](#想法)
  
# 华为比赛

---
- 由于华为NAIE平台操作不便,所以先从清华netman智能运维实验室的网站上下载了一个KPI异常检测数据集,其规模和格式均与比赛数据集基本相同.在该数据集上使用light-gbm(基于决策树)搭建了一个模型. 具体见代码`kpi_anomaly.ipynb`
- 比赛数据集更新, 训练集数据条数从3800+增加到30000+,测试集从500+增加到3000+


# 知识图谱研究

---
## Formulation
- 设训练/验证/测试集中的三元组对应的知识图谱为$G=\{(h,r,t)|h,t \in E,r \in R\}$, $G$的背景知识图谱$G_b=\{(h,r,t)|h,t \in E_b,r \in R_b\}$, $E$和$E_b$是实体集,$R$和$R_b$是关系集. 其中$E \subset E_b$,$R \cap R_b=\varnothing$.
- 对$e \in E_b$, $e$的邻居集合为$N_e$, 邻居个数为$|N_e|$, 有$N_e=\{(r_i,e_i)|(e,r_i,e_i) \in G_b\}$, 对$n_i \in N_e$,$n_i= (r_i,e_i)$.
- TransE的训练目标: $\forall (h,r,t) \in G$, 有$v_h+v_r \approx v_t$
- 设任务集合为$T$, 对$task \in T$, $task=(support,query)$, $support=\{(h,r,t)|(h,r,t) \in G\}$,$query=\{(h,r,t),(h,r,t')|(h,r,t)\in G,(h,r,t') \notin G\}$. $|support|$为hyper-parameter, $|support|=1$表示one shot. 在$Meta_{train}$阶段,query中的正确三元组与错误三元组的个数相等. 在$Meta_{test}$阶段,正确三元组个数为1,错误三元组数量为候选集实体数量减1, 即$|candidates|-1$
## TransE
    用transe表示不使用任何神经网络模型,直接通过预训练的实体嵌入完成预测任务
1. $\forall task\in T$, 其中$task=(support,query)$, 设$r_{s}$为根据$support=\{(h,r,t)|(h,r,t) \in G\}$计算得到的关系嵌入表示,$n$为$support$中三元组的个数. 
$$r_s=\frac{1}{n}\sum_{i=1}^n(t_i-h_i)$$
2. 将$r_s$迁移到$query$中,计算每个三元组的$score$, 用$score^+$和$score^-$分别表示有效三元组(正样本)和无效三元组(负样本)的得分.
$$ score=||h+r_s-t||_2$$
3. 在$Meta_{test}$阶段,对$query$中所有三元组的$score$排名,根据$score^+$的排名计算评测指标(*MMR,Hits@10,Hits@5,Hits@1*). 在$Meta_{train}$阶段,则根据$query$中所有三元组的$score$计算$Margin Loss$.
   $$MarginLoss=max(0,margin-(score^+-score^-))$$
    
    ```Python
    def rank(positive_score,negative_score):
        scores=concat(negative_score,positive_score)
        sort(scores)
        rank=scores.indexof(len(scores)-1)
        return rank
    ```
## TF-IDF 
    通过计算实体e的所有邻居的tfidf值对邻居的向量表示进行加权求和,作为e的初始向量
1. 设训练/验证/测试集中的三元组对应的知识图谱为$G=\{E,R\}$,$G$的背景知识图谱$G_b=\{E_b,R_b\}$, 其中$E \subset E_b$,$R \cap R_b=\varnothing$. 根据$G_b$中的三元组计算$e_i \in E_b$的邻居$N_i$,设$N=\{N_1,N_2,...,N_i\}$为所有邻居信息的集合, 其中$N_i=\{(r_1,e_1),(r_2,e_2),...,(r_k,e_n)|k\in range(|R_b|),n \in range(|E_b|)\}$, 计算$N_i$中$(r_k,e_n)$的$r_k$的TF-IDF值,将其作为$e_i$的邻居$(r_k,e_n)$的权重.
$$TF_{r_k}=\frac{r_k在N_i中的出现次数}{N_i中r的个数}$$
$$IDF_{r_k}=\lg{\frac{|N|}{包含r_k的N的个数+1}}$$
$$TF{-}IDF_{r_k}=TF_{r_k}\times IDF_{r_k}$$
2. 用第j个邻居$n_j=(r_k,e_n)$的$r_k$的$tfidf$值作为$n_j$的权重$w_j$,对邻居的向量$v_{n_j}$进行加权求和作为$e_i$的向量$v_{e_i}$,标记为``` pre-train=tfidf-neighbor```
$$v_{n}=concat(v_r,v_t)$$
$$v_{e_i}=\sum_{j=1}^{|N_i|}{w_j v_{n_j}}$$
也可以将每个邻居的TF-IDF组成的向量作为e的向量.标记为``` pre-train=tfidf```
$$v_{e_i}=[w_1,w_2,...,w_j]$$
也可以将每个邻居中关系向量的TFIDF加权之和作为e的向量.标记为``` pre-train=tfidf-relation```
$$v_{e_i}=\sum_{j=1}^{|N_i|}{w_j v_{r_j}}$$
## Concatenate
    通过拼接三元组中头实体和尾实体的向量得到实体对的向量,在预测过程中,比较support与query中实体对的余弦相似度.
1. 对于一个三元组$triplet$$(h,r,t)$,$v_{triplet}=concat(v_h,v_t)$. 计算$v_{spt}$与query中所有三元组向量的余弦距离.
   $$v_{spt}=\frac{1}{n}\sum_{i=1}^{n}{v_{triplet_i}}$$
   $$score=cosine(v_{spt},v_{triplet}), triplet \in query$$
2. 根据query中有效三元组的分数排名计算评测指标.


## Experiments

### 条件说明
 + cosine表示用余弦距离替换欧式距离计算score
 + transe表示用[TransE](#transe)中说明的方法做预测
 + concat表示用[concat](#concat)中说明的方法做预测
 + average表示将[TF-IDF](#tf-idf)中的$w_j$替换为$\frac{1}{n}$
 + finetune表示允许根据损失函数更新实体嵌入
 + random表示随机初始化实体嵌入
 + pre-train=transe表示用transe初始化实体嵌入
 + pre-train=tfidf表示用邻居关系的tfidf向量$[w_1,w_2,...,w_j]$作为初始实体嵌入
 + pre-train=tfidf-relation表示用邻居关系向量$v_r$的加权之和作为初始实体嵌入,权重是关系的tfidf
 + pre-train=tfidf-neighbor表示用邻居向量$concat(v_r,v_t)$的加权之和作为初始实体嵌入,权重是关系的tfidf
 + pre-train=average-neighbor表示用邻居向量$concat(v_r,v_t)$求和平均作为初始实体嵌入
 + attention表示在n-shot条件下(n>1)用attention机制聚合support中的$v_r$
 + batch_size=4, 在不finetune条件下,模型为基本的数学计算,所以速度很快
  
### 实验结果
<!-- + NELL数据集 -->
|#|Method|initial|n-shot|finetune|MRR|Hits@10|Hits@5|Hits@1|
|---|---|---|---|---|---|---|---|---|
|1|cosine transe attention|transe| 5||0.230| 0.387|0.301|0.149
|2|transe  attention|transe| 5 ||0.231 |0.381  |0.309  |0.149|
|3|transe |transe| 5 ||0.225 |0.389  |0.300  |0.144|
|4|cosine transe |transe| 5 ||0.231 |0.387  |0.308  |0.150|
|5|concat |tfidf-relation| 1||0.130      | 0.145  | 0.130   | 0.120|
|6|transe |tfidf-relation| 1|| 0.092      | 0.116  | 0.096   | 0.078|
|7|transe |transe| 1||0.139      | 0.227  | 0.172   | 0.089|
|8|transe  |random|1||0.110      | 0.122  | 0.119   | 0.097|
|9|transe  |random|5||0.092      | 0.134  | 0.113   | 0.065|
|10|transe |transe|1 |T|0.159      | 0.272  | 0.204   | 0.097|
|11|transe   |random|1|T|0.122      | 0.162  | 0.139   | 0.098|
|12|transe  |random|5|T|0.069      | 0.099  | 0.075   | 0.049|
|13|concat |tfidf-neighbor|1||0.190      | 0.237  | 0.213   | 0.157|
|14|concat  |tfidf-neighbor|5||0.069      | 0.140  | 0.108   | 0.021|
|15|concat |average-relation| 1||0.070      | 0.136  | 0.110   | 0.038|
|16|concat |average-relation|5||0.085      | 0.146  | 0.138   | 0.039|


### 对比分析
1. **[1, 2, 3, 4]** 表明使用attention,更适合用欧式距离计算分数而不是cosine. 不使用attention,则更适合用cosine而不是用欧式距离.
2. **[8, 9]** 表明其他方法需要高于**0.11**才能说明模型起正面作用. 5-shot结果比1-shot结果差, 初步分析是因为预测是针对query的, query中部分能预测正确的三元组在5-shot条件下被放入了support中,而这些三元组对评测指标的贡献较大,导致5-shot比1-shot差.
3. **[5]** 表明邻居信息的tfidf值对部分数据起正面作用, 即考虑不同实体的邻居信息中的关系种类的相似性对预测有正面作用
4. **[13, 14, 15, 16]** 表明不加权的邻居信息起负面作用. 加权的邻居信息对部分数据起正面作用, 即考虑不同实体的邻居信息之间的语义相关性对预测有正面作用. 
## 想法
1. 将tf-idf加权的邻居信息拼接在实体的原始向量后,计算匹配分数时分别计算查询集与支持集之间关系的相似度与实体邻居信息的语义相似度.
   $$score=||h+r_s-t||_2+cosine(N_{spt},concat(N_h,N_t))$$
1. 目前只考虑了关系的tfidf作为一个邻居的权重,还可以考虑尾实体的tfidf.
$$ v_{e_i}=\sum_{j=1}^{|N_i|}w_{r_i}concat(r_i,t_i) \tag{1}$$ 
$$ v_{e_i}=\sum_{j=1}^{|N_i|}concat(w_{r_i}r_i,w_{t_i}t_i)\tag{2}$$
1. 不同实体的邻居信息可以看作一个序列数据, 通过Seq2Seq与Auto-Encoder(自编码器)结合的方式编码邻居信息,通过目标函数学习不同邻居的权重.







