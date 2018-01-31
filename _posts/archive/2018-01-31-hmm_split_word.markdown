---
layout: archive
title: HMM模型分词理解笔记 
date: 2018-01-31 10:28
categories: nlp
tag: jieba split_word hmm viterbi

---

## HMM模型的几个要点：
1. observed_set: 可观测的序列
2. status_set: 隐性状态序列，一般为要求的结果
3. init_status: 初始化status的概率
4. transfer_prob: 隐性状态概率之间的转移概率，比如总共status_set有n个，tranfer_prob就有n*n个数值，一般用二维数组保存
5. emit_prob: 隐性状态下表现出可观测序列的概率，有size(observed_set)*
size(status_set)个数值，也可以用二维数组保存

## 以中文分词为例子：
* observed_set:可观测的序列就是每一个单词组成的序列
* status_set:是我们要求的结果，分词中设定是该词的'词性'(并非真正意义上的词性，
这里的词性指该词再词组中是作为开始[Begin]、结束[End]、中间词[Midle]、单个词[Single])
因为求出了词性后我们就可以直接使用组合推导出分词的结果了
* init_status: 初始话status的概率，分词中，第一个单词的词性概率，基本上只有Begin和Single有概率数值，其余两个End和Midle不可能存在(也就是概率为0)
* transfer_prob: status之间的转移概率，也就是Begin->Begin, Begin->Midle, Begin->End,
Begin->Singel, End->Begin, End->Midle,...等等[4*4]16个状态转移的概率值, 也就begin词后接
begin词的概率，begin词后接end词的概率，begin词后接midle词的概率等等
* emit_prob: 隐性状态下表现出可观测序列的概率, 具体到分词中就是：假设该位置的词是
begin/end/midle/single词的情况下，其显示为具体字（上万个汉字中的某个字）的概率

**NOTE**: 关于概率值的计算，暂时没有去深入了解，说是根据语料统计出来的，有时间再去了解如何
统计这些概率数值。所以下面的python代码中那些概率数值都是随便取个值代替，并非真正可用的概率
值

### Viterbi算法:
![](/assets/images/hmm_split_word_viterbi.png)

python代码示意：
```
#!/usr/bin/python
#coding=utf-8

# Copyright (c) 2018 smtihemail@163.com. All rights reserved.
# Author：smithemail@163.com
# Time  ：2018-01-31

import sys
import os

## 可观测序列
observe_list = "我爱北京天安门"

## 隐性状态
status_list = ["begin", "end", "midle", "single"]

## 初始化状态序列
init_status_prob = [0.5, 0, 0, 0.5] #第一单词概率有0.5是begin, 0.5是single

## 状态之间转换概率,数值是我瞎编的，正确的数值应该是统计出来的
transfer_prob = [
            [0, 0.3, 0.4, 0.5],
            [0.3, 0, 0, 0.7],
            [0, 0.4, 0.3, 0.3],
            [0.4, 0, 0, 0.6]
        ]

## 隐性状态下表现出可观测序列的概率, 也叫发射概率,也是瞎编的
emit_prob = [
            [b1, b2, b3, b4,....],
            [e1, e2, e3, e4,....],
            [m1, m2, m3, m4,....],
            [s1, s2, s3, s4,....],
        ]

## Viterbi算法
def viterbi():
    weight = [[0 for j range(len(observe_list))] for i in range(4)]
    path = [[0 for j range(len(observe_list))] for i in range(4)]

    for i in range(len(weight[0])): #
        for j in range(len(weight)):
            if i == 0:
                weight[i][j] = init_status_prob[j];
            else:
                word = observe_list[i]
                for p in range(len(weight)):
                    preb = weight[i-1][p] * transfer_prob[p][j] * emit_prob[j][word]
                    if preb > weight[i][j]:
                        weight[i][j] = preb #保存最大的概率
                        path[i][j] = p; #最大概率的情况下，其上一个词的词性
    return weight, path

weight, path = viterbi()

# 最后一个单词是end或single
end_prob = weight[len(observe_list)-1][1]
single_prob = weight[len(observe_list)-1][3]

predict_list = ['' for i in range(len(observe_list))]
status_idx = 0;
if end_prob > single_prob:
    predict_list[len(observe_list)-1] = status_list[1]
    status_idx = 1;
else:
    predict_list[len(observe_list)-1] = status_list[3]
    status_idx = 3
# 根据最后一个词的最大可能性的词性反推出上一个词的词性（之前计算的时候保存在path中）
for i in range(len(observe_list)):
    idx = len(observe_list) - 2 - i
    pre_status_idx = path[idx][status_idx]
    predict_list[idx] = status_list[pre_status_idx];

print(predict_list)

```
最后我们就可与根据predict_list中词性切割成一个个词组了，切割原则begin->(midle)->end, single   
**参考文档**： 
1. [隐马尔可夫模型维基百科](https://zh.wikipedia.org/wiki/%E9%9A%90%E9%A9%AC%E5%B0%94%E5%8F%AF%E5%A4%AB%E6%A8%A1%E5%9E%8B#%E9%A6%AC%E5%8F%AF%E5%A4%AB%E6%A8%A1%E5%9E%8B%E7%9A%84%E6%A9%9F%E7%8E%87)
1. [维特比算法维基百科](https://zh.wikipedia.org/zh-hans/%E7%BB%B4%E7%89%B9%E6%AF%94%E7%AE%97%E6%B3%95) 
2. [中文分词之HMM模型详解](<https://yanyiwu.com/work/2014/04/07/hmm-segment-xiangjie.html) 
3. [一文搞懂HMM（隐马尔可夫模型）](http://www.cnblogs.com/skyme/p/4651331.html) 
