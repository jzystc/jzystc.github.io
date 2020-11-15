---
title: weekly report 2020-11-15
date: 2020-11-14 09:57:41
tags:
    - report
categories:
    - note
mathjax: true
---
### 二维卷积
在二维卷积中, 将头实体与尾实体的向量上下拼接, 得到2\*100的向量. 第一层卷积层, 使用128个2\*1的卷积核从左往右滑动得到维度为1\*100\*128的输出. 后面的卷积层使用1\*1卷积,主要作用是将多个通道的feature_map线性加权求和,以此减少通道数(降维). 这部分主要借鉴ConvKB的思想.
```python
class ProtoNet(nn.Module):
    def __init__(self):
        super(ProtoNet, self).__init__()
        self.encoder_conv2d = nn.Sequential(
        # 第一层输入通道为1,输出通道为128,卷积核大小为(1,2)
        # 第一层卷积操作之后,feature_map=1*100*128
        Conv2d(1, 128, (1, 2)),
        ReLU(),
        # 第二层输入通道为128,输出通道为64,卷积核大小为(1,1)
        # 第二层卷积操作之后,feature_map=1*100*64
        Conv2d(128, 64, (1, 1)),
        ReLU(),
        Conv2d(64, 32, (1, 1)),
        ReLU(),
        Conv2d(32, 1, (1, 1)))

    def forward(self, x):
        batch_size, num_triples, input_length, dim = x.size()
        x = x.view(batch_size*num_triples, 1,
                   x.shape[2], x.shape[3]).transpose(2, 3)
        x = self.encoder_conv2d(x)
        return x.view(batch_size, num_triples, -1)
```
### 一维卷积
一维卷积是卷积神经网络在自然语言处理中的常用方法.以```I like coding.```为例,[I, like, coding]是词汇表,每个单词都用100维的向量表示,则输入是3\*100的矩阵.一维卷积是固定住输入矩阵的第二维,卷积核只在第一维上纵向滑动.当句子长度为3时,可以使用2\*100的卷积核提取前后两个单词之间的特征. 但是由于头尾实体只有2个单词, 所以卷积核只能设置为2\*100, 无法在第一维上滑动.
```python
class ProtoNet(nn.Module):
    def __init__(self):
        super(ProtoNet, self).__init__()
        self.encoder_conv1d = nn.Sequential(
            # 第一层输入通道为100,对应实体嵌入为100维.
            # 有256个卷积核,则输出通道为256,卷积核大小为(2,100)
            # 第一层卷积操作之后,feature_map=1*1*256
            Conv1d(100, 256, 2),
            ReLU(),
            # 第二层卷积操作之后,feature_map=1*1*128
            Conv1d(256, 128, 1))

    def forward(self, x):
        batch_size, num_triples, input_length, dim = x.size()
        x = x.view(batch_size*num_triples,
                   x.shape[2], x.shape[3]).transpose(1, 2)
        x = self.encoder_conv1d(x)
        return x.view(batch_size, num_triples, -1)
```

### 基于ConvKB的负样本选择器
+ ConvKB的模型是1个2维卷积层+1个全连接层, 其中2维卷积层由n个3\*1的卷积核组成,输出维度为1\*100\*n的feature_map. 全连接层接收卷积层的输入将feature_map的维度降为1, 作为衡量三元组有效性的分数.
+ 构建一个卷积神经网络负样本选择器,对负样本实体对打分,选取其中分数最低的作为负样本的代表
```python
def support_negative_triples_selector(self, support_negative):
    # batch_size是任务数,num_triples=few*num_support_negative
    # num_support_negative是支持集中每个有效三元组对应错误三元组的数量
    # length=2,每个实体对由1个头实体和1个尾实体组成
    # dim=100,实体嵌入维度为100
    batch_size, num_triples, length, dim = support_negative.size()
    support_negative_score = self.convkb(
        support_negative.view(-1, length, dim))
    # (batch_size,few,num_sn/few,1)
    support_negative_score = support_negative_score.view(
        batch_size, self.few, self.num_support_negative, 1)
    # 对每个正样本对应的n个负样本按分数增序排列
    support_negative_score_sorted = torch.sort(
        support_negative_score, dim=2, descending=False)
    # 取分数最低的 (batch_size,few,1)
    choices = support_negative_score_sorted.indices[:, :, 0, :]
    idx_1 = torch.LongTensor(
        np.arange(batch_size).repeat(self.few)).to(self.device)
    idx_2 = torch.LongTensor(
        np.tile(np.arange(self.few), batch_size)).to(self.device)
    idx_3 = choices.view(batch_size*self.few).to(self.device)
    support_negative = support_negative.view(
        batch_size, self.few, self.num_support_negative, 2, -1)
    support_negative = support_negative[idx_1, idx_2, idx_3]
    support_negative = support_negative.view(
        batch_size, self.few, 2, self.dim)
    return support_negative

```

### 实验结果

```
dataset=NELL es_p=5
```
| Method | finetune | n-shot | epoch | MRR   | Hits@10 | Hits@5 | Hits@1 |
| ------ | -------- | ------ | ----- | ----- | ------- | ------ | ------ |
| conv2d | T        | 1      | 24000 | 0.189 | 0.290   | 0.232  | 0.137  |
| conv2d | F        | 1      | 11000 | 0.134 | 0.179   | 0.149  | 0.108  |
| conv1d | T        | 1      |       |       |         |        |        |
| conv1d | F        | 1      | 5000  | 0.138 | 0.265   | 0.208  | 0.075  |

```
dataset=Wiki    es_p=1
```
| Method    | finetune | n-shot | epoch | MRR   | Hits@10 | Hits@5 | Hits@1 |
| --------- | -------- | ------ | ----- | ----- | ------- | ------ | ------ |
| prototype | T        | 1      | 2000  | 0.206 | 0.298   | 0.251  | 0.158  |
| MetaR     | T        | 1      | 7000  | 0.319 | 0.411   | 0.361  | 0.268  |
```

dataset=Wiki    es_p=5
```
| Method    | finetune | n-shot | epoch | MRR   | Hits@10 | Hits@5 | Hits@1 |
| --------- | -------- | ------ | ----- | ----- | ------- | ------ | ------ |
| prototype | T        | 1      | 10000 | 0.278 | 0.367   | 0.318  | 0.231  |
| MetaR     | T        | 1      | 9000  | 0.272 | 0.386   | 0.336  | 0.208  |
