---
title: weekly report 2020-11-07
date: 2020-11-07 10:49:49
tags: 
    - report
---
- ### prototype
+ support_positive: 支持集中的n个正确三元组$(h,r,t)$
+ support_negative: 支持集中的n个错误三元组$(h,r,t')$
+ query_positive: 查询集中的k/1个正确三元组$(h,r,t)$,k对应训练集,1对应验证/测试集
+ query_negative: 支持集中的k/m个错误三元组$(h,r,t')$,k对应训练集,m对应验证/测试集
+ support_positive=>全连接层=>正类的向量表示
+ support_negative=>全连接层=>负类的向量表示

### 超参数
+ batch_size = 128
+ eval_epoch = 1000 ```每训练n个epoch后验证一次```
+ early_stop_patience (es_p) = 1 ```验证集上的结果连续低于最好结果n次后停止训练并用最好的模型进行测试.该参数与epoch数正相关.验证结果可能在连续k次低于最好情况下在第k+1次验证时超过最好值, 所以可以通过增大es_p训练出拟合度更高的模型,负面影响是延长了训练时间.```

### 实验结果

```dataset=NELL    es_p=1``` 

| Method    | finetune | n-shot | epoch | MRR   | Hits@10 | Hits@5 | Hits@1 |
| --------- | -------- | ------ | ----- | ----- | ------- | ------ | ------ |
| prototype | F        | 1      | 4000  | 0.158 | 0.268   | 0.212  | 0.102  |
| MetaR     | F        | 1      | 2000  | 0.077 | 0.116   | 0.083  | 0.044  |
| TransE    | F        | 1      | 1000  | 0.139 | 0.227   | 0.172  | 0.089  |
| prototype | F        | 5      | 4000  | 0.162 | 0.260   | 0.214  | 0.107  |
| MetaR     | F        | 5      | 5000  | 0.098 | 0.214   | 0.093  | 0.050  |
| TransE    | F        | 5      | 1000  | 0.225 | 0.389   | 0.300  | 0.144  |
| prototype | T        | 1      | 4000  | 0.197 | 0.308   | 0.250  | 0.136  |
| MetaR     | T        | 1      | 4000  | 0.132 | 0.231   | 0.160  | 0.084  |
| prototype | T        | 5      | 3000  | 0.200 | 0.345   | 0.260  | 0.136  |
| MetaR     | T        | 5      | 2000  | 0.110 | 0.242   | 0.173  | 0.052  |


```dataset=NELL    es_p=5``` 

| Method | finetune | n-shot | epoch | MRR   | Hits@10 | Hits@5 | Hits@1 |
| ------ | -------- | ------ | ----- | ----- | ------- | ------ | ------ |
| MetaR  | F        | 1      | 7000  | 0.109 | 0.183   | 0.145  | 0.054  |
| MetaR  | F        | 5      | 5000  | 0.098 | 0.214   | 0.093  | 0.050  |
| MetaR  | T        | 1      | 11000 | 0.148 | 0.245   | 0.184  | 0.101  |
| MetaR  | T        | 5      | 27000 | 0.214 | 0.341   | 0.292  | 0.143  |


```dataset=Wiki    es_p=1```

| Method    | finetune | n-shot | epoch | MRR   | Hits@10 | Hits@5 | Hits@1 |
| --------- | -------- | ------ | ----- | ----- | ------- | ------ | ------ |
| prototype | F        | 1      | 5000  | 0.101 | 0.217   | 0.131  | 0.049  |
| MetaR     | F        | 1      | 3000  | 0.076 | 0.183   | 0.102  | 0.026  |
| TransE    | F        | 1      | 1000  | 0.086 | 0.117   | 0.099  | 0.067  |
| prototype | F        | 5      | 4000  | 0.129 | 0.279   | 0.195  | 0.060  |
| MetaR     | F        | 5      | 5000  | 0.086 | 0.209   | 0.112  | 0.034  |
| TransE    | F        | 5      | 1000  | 0.095 | 0.137   | 0.115  | 0.072  |

### 分析
1. TransE在5-shot情况下,在NELL数据集上效果最好,但是在Wiki数据集上,TransE在1-shot和5-shot条件下的效果都差于prototype
2. prototype方法收敛速度更快,可以通过更少的epoch达到更好的效果
3. prototype在不finetune的情况下比MetaR更好. MetaR使用hinge-loss,在不finetune条件下,损失只能降低到0.75左右. prototype使用交叉熵损失,在不finetune条件下,损失也能一直降低


