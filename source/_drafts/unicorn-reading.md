---
title: UNICORN阅读笔记
date: 2020-08-18 16:36:50
tags:
---

## Introduction

APT的特征：

* APT攻击是长期行为（与传统攻击最大的区别）
* 经常利用0day漏洞

传统解决方案：

* 基于Malware Signature的方式无法发现利用0day漏洞的攻击【Before we knew it: an empirical study of zero-day attacks in the real world】
* 基于异常行为（Anomaly）检测的方法
  * 无法发现长期的行为模式
  * 只是分析事件和syscall的小片段，更容易被攻击者规避
* 长期行为建模【Long-span program behavior modeling and attack detection】，只局限于分析事件共同发生这种现象，避免过度使用计算和内存资源

新的基于Provenance的APT检测：

* 把整个系统的运行信息构造为一个有向无环图（DAG），描述系统的信息流动
  * 构造：
    * subject，一般为进程；object，一般为各种OS对象（除了进程之外）
    * 用事件event作为边来连接subject和object
  * 优势：
    * 即使两个相关的event间隔时间很长，图也能保留这种相关信息，提供APT追踪的更丰富的上下文
    * 这种上下文信息可以更好地区分benign和malicious事件

现有工作的缺陷（静态代表FRAP，动态代表StreamSpot）：

1. 静态模型无法捕捉和获取系统的长期行为
2. 由于APT攻击是长期行为，可能会逐渐污染动态模型（使模型将malicious的行为识别为benign行为）
3. 需要将整个Provenance Graph放在主内存当中，因此不适合长期攻击的检测

## Background

基于Syscall信息追踪的APT分析：

* 早期研究只是单纯地对syscall日志进行分析，并尝试发现异常点（Outlier Point），或者通过一段syscall调用日志来分析和判断异常情况。这些都很难有效果，因为这些分析忽略了这一点或者一段syscall的历史信息，会导致很多FP。
* 后来有研究尝试从syscall信息重建Provenance Graph，但这种方法很难保证图的正确性：
  * 由于syscall记录本身可能是不正确、不完善和不可靠的
  * 很多Hook系统调用的系统有因为并行执行导致的问题
  * 很多内核级的记录系统有TOATTOU的bug，这个bug会导致记录的系统调用参数和实际传入内核的参数不同
  * 由于很多内核线程并不使用系统调用，根据系统调用日志建图可能出现很多图不连通的问题

当前研究（FRAP，StreamSpot，HOLMES）的局限：

1. 预定义的边匹配规则过于敏感，且无法检测APT中常见的0day漏洞（针对HOLMES）
2. 图分析只分析了局部特征，或是节点周围的小邻域，或者只是节点/边的属性，或者只是一部分子图
3. 对系统行为的建模无法适应对APT的检测，静态模型无法描述长时间运行系统的动态特征（针对FRAP），而动态模型有被APT攻击污染的风险（针对StreamSpot）
4. Provenance Graph存储在内存当中，随着时间跨度增加、图的规模增大无法扩展

## Design

### Overview

1. 输入一个带标签的流式图。这个图由CamFlow系统生成，只有一个图，包含整个系统的Provenance信息。
2. 在线维护一个图的直方图。通过在线的迭代算法维护一个直方图，直方图中每个元素代表着某一类子图结构的数量，这个结构通过节点标签考虑了图的异质性，通过边权重考虑了时间序。同时，引入指数遗忘机制，缓慢减少直方图元素的分量，这样在保留信息流的同时可以适应系统状态变化。
3. 周期性计算图的Sketch。采用HistoSketch算法，把变长且不断更新的直方图周期性地转换成长度相同的Sketch，这种Sketch可以用来计算相似度，并且本身也提供了指数遗忘和对Concept-Drift的支持。
4. 构造正常运行时系统行为的模型。通过定期生成Sketch形成描述正常行为的Cluster，同时并不修改这些模型保证攻击行为不会污染模型。

### Histogram

直方图生成核心是1维WL算法的变种：

* 概括来说就是通过迭代和重标签，逐步概括每一个节点周围的节点的结构信息（准确来说，是以每个节点为根的生成子树），生成一个直方图。最初用来比较小规模图的结构相似性。
* 针对原算法的改进：
  * 引入边标签，并将边标签也考虑到新的Label的生成过程中（原算法只有节点有标签）。
  * 引入边的时间戳，在原算法的Label排序过程中利用时间戳进行排序。

适应流式系统的优化：

* 在流式设计中，只对新的节点和新节点邻域内的节点运行一遍算法，这样就显著减少了计算量。
* **偏序性**：SOTA的Provenance记录系统应该保持偏序性，也就是说对于每个节点，其入边都会比出边早一步到达。在CamFlow这种用节点的Version来区分节点的系统，更新操作只需要在边对应的目标节点中进行。

遗忘机制：

UNICORN在直方图生成上就直接引入了遗忘机制，在直方图构建时就按照时间将历史数据乘以与时间相关的权重。

### Sketch

从Histogram生成Sketch这一步比较简单，根据数据的Streaming（在线更新）和Histogram分量非固定的特点，采用**HistoSketch**方法将一个Histogram转换为定长的Sketch。

### 模型构建和异常检测

Sketch生成后，采用K-medoids算法进行分组，分组情况用Silhouette参数进行调优，对每一个Sketch赋予最佳分组号。

（K-medoids算法是常见的K-means算法的变种，将每个Cluster的中心作为分组基准；Silhouette是基于相似性提出的分组评价依据，衡量每个分组内元素的平均相似度）

通过定期生成Sketch并进行归类，可以形成一个Instance上系统运行的状态转移序列，这个状态转移序列就成为系统行为的建模。在部署时进行同样的步骤，对于每个Sketch的归类，如果和当前/下一状态的分类不同，则判定为异常。

## Implementation

涉及到Provenance信息采集的部分基于CamFlow：

* 基于Linux Security Models（LSM）框架来确保高质量和可靠的信息流追踪和记录
* LSM解决了常见Syscall记录系统的竞争条件问题，通过将记录点直接设置在内核中而不是系统调用界面

涉及到的图处理基于GraphChi：

* GraphChi是基于磁盘而非内存的图处理系统，可以很快地在规模很大的图上进行运算
* GraphChi在内部用平行滑窗（PSW）算法把图分成边数类似的Shard，在每一个Shard上进行并行计算以加快速度
* GraphChi的IO模型和批量操作使得图可以存储在磁盘当中，且在计算时保持相当的性能

## Evaluation

UNICORN评估角度：

1. 准确性
2. 哪些设计上的考虑比较重要
3. 指数遗忘的策略是否有效
4. 系统长期运行建模的有效性对比
5. 速度，是否满足实时性要求
6. 内存、CPU使用

> Testing guidelines: Benchmarking crimes: an emerging threat in systems security

### 和StreamSpot比较

StreamSpot数据集：

* 6个情景，其中5个对应Benign情形，有1个情景中发生了攻击事件，主要对应浏览器浏览行为
* 通过自动化，每个情景重复100遍，对应100个Graph，用Linux下的SystemTap系统记录所有Syscall

与StreamSpot的比较：

* 主要参数是图算法中的迭代参数R。在UNICORN的实现中，R增大（从1到3）对应着FP案例减少，也对应着算法提取更大范围邻域的信息
* 注：文中并未提到具体训练/测试集选取等细节，StreamSpot文章中有相应的分析

### DARPA TC 数据集测试

* 总共3次实验，分别有Benign情形和APT攻击情形
* 训练/测试集分割：每次实验数据中90%划分为训练数据，10%为测试数据
* 结果：正确率极高，在三次实验中正确率达到98%以上。但文中并未说明这个正确率计算的**载体**（即这个正确率是节点正确率还是攻击检测正确率）。

### 参数选取问题

UNICORN有如下参数：

* 跳数/邻居半径。过小则无法提取足够信息，过大则会包括噪音。测试中3为最好。
* Sketch大小。过大虽然可以包含更多信息，但也会导致维度爆炸。2000为最佳。
* Sketch生成周期（以边数为单位）。过大则无法准确记录状态变化。3000为最佳。

## Discussion

### Anomaly

问题（基于异常行为检测的方法存在的通病）：

* 需要一段训练的时间，假设在这段时间内系统中只会有Benign行为，并且能够全面捕捉到。
* 在训练期内发生的行为概括了绝大部分Benign行为。因此如果发生了新的Benign行为，就会产生FP。
* 同样，由于需要对状态进行建模，当系统状态被人为大幅改变时就会出现问题（比如备份还原）。

UNICORN的改进点：

* UNICORN的算法更高效，能够稳定、高效、连续地对大规模系统进行建模。
* UNICORN的图算法捕捉了范围更大的子图结构，因此能够总结更多的局部图信息，也就更难进行欺骗性攻击（mimicry attack）。
* UNICORN的CWS算法引入了随机性，并不保证能通过模仿正常节点的特征来进行欺骗性攻击。

### False Alarm

问题：

* UNICORN本质上是静态模型，来避免攻击行为污染模型。（实际上UNICORN定时从Provenance Graph生成Sketch作为当前系统运行状态的Snapshot；而模型一旦建立就不再改变）
* 由于静态模型的局限性，当正常的系统行为模式改变时，就会产生很多FP。

UNICORN的改进点：

* 通过HistoSketch引入Concept Drift，用于捕捉系统在正常运行的过程中产生的特征变化，并且避免产生过多的FP。

### 图分析

问题：

* 图算法实际上有很多参数（最重要的参数可能是邻域半径R），如何确定参数是一个重要的问题。

UNICORN的Argument：

* 使用OpenTuner进行参数自动优化，自动选择最优参数。
* 对所有的测试都使用了相同的参数，具有稳定性。

### 系统行为同质性

讨论：

* StreamSpot数据集是对近乎同质的行为进行重复建模（主要是网页浏览等行为）。
* UNICORN在StreamSpot上效果更好，说明这种方法在行为预定义的Host上效果更好。
* 而在Workstation这种环境下，APT攻击检测就更加困难。

### 数据集和交叉验证

讨论：

* 现有数据比较少，很多数据是过期数据，或者无法直接构成Provenance Graph，需要翻译。
* 现有IDS的研究很多未开源，且采用自己的数据集，这些数据集的产生也未详述。
* 这就导致系统之间的比较十分困难。

## Related Work

### Intrusion Detection

* 定长的Syscall序列来定位攻击，推广到可变长度的特征序列。但准确性较低，无法应付复杂情形。（37 -> 29，127）
* 为Syscall添加状态来提供上下文信息，用有限状态机进行行为建模。（109，36，62，81）但这种方法复杂度高于多项式复杂度，可行性较差。
* 总结性文章：113，78（扩展26，68，91）

### Provenance

初步工作：

* BackTracker：通过分析Provenance Graph找到攻击入口点。
* PriorTracker：通过为节点赋予优先级简化分析过程，并引入前向分析找出攻击路径。
* HERCULE：用Community Detection算法找出包含攻击路径的Community。
* Winnower：在Povenance Graph上进行语义推断，以减少Provenance数据导致的存储和网络需求。
* NoDoze：在Provenance Graph上实现攻击分流（attack triage），鉴别可疑的路径。
* CamQuery：基于CamFlow，提出了能够实时分析Provenance Graph的框架，也说明Provenance Graph在APT分析方面很有价值。

进一步工作：

* SLEUTH和HOLMES，注重于利用provenance数据进行攻击溯源和重建。背景框架是【Identifying the provenance of correlated anomalies】，都需要先验知识。
* Poirot同样也是依赖先验信息，通过匹配攻击入口点来追踪APT。一般是通过专业的报告来找到这些入口点。

## 参考文献分类

> 异常行为检测：
> 
> 114: Automated response using system-call delay
> 130: A sharper sense of self: Probabilistic reasoning of program behaviors for anomaly detection with context sensitivity
> 36: Anomaly detection using call stack information
> 81: Detecting intrusions through system call sequence and argument analysis
> 92: Exploiting execution context for the detection of anomalous system calls
> 109: A fast automatonbased method for detecting anomalous program behaviors
> 
> APT+图分析：
> 
> 15: Mining data provenance to detect advanced persistent threats
> 19: Aggregating unsupervised provenance anomaly detectors
> 83: Fast memory-efficient anomaly detection in streaming heterogeneous graphs
> 87: Holmes: Real-time apt detection through correlation of suspicious information flows
> 55: Nodoze: Combatting threat alert fatigue with automated provenance triage
> 53: Frappuccino: fault-detection through runtime analysis of provenance
> 
> Syscall相关的APT分析：
> 
> 37: A sense of self for unix processes
> 134: A markov chain model of temporal behavior for anomaly detection
> 61: Consistency analysis of authorization hook placement in the linux security modules framework
> 123: A markov chain model of temporal behavior for anomaly detection
> 43: Traps and pitfalls: Practical problems in system call interposition based security tools
> 
> 存在的问题讨论：
> 
> 125: Exploiting concurrency vulnerabilities in system call wrappers
> 98: A practical mimicry attack against powerful system-call monitors