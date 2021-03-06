---
title: "性能测试"
linkTitle: "性能测试"
weight: 251
draft: false
---

# 1. 测试数据

5 份公开测试数据集分别从 [ann-benchmarks](https://github.com/erikbern/ann-benchmarks/)
与 [big-ann-benchmarks](http://big-ann-benchmarks.com/index.html#bench-datasets) 中选取（为了测试不同数据量级下的性能表现，部分数据集只选取了其中一部分）：

| dataset    | dimension    | type    |distance    |datasize|
| --- | --- | --- | --- | --- |
|sift-128-euclidean    |128    |FP32    |Euclidean    |1,000,000 |
|gist-960-euclidean    |960    |FP32    |Euclidean    |1,000,000 |
|deep-image-96-angular    | 96    |FP32    | Cosine |10,000,000|
|yandex-text-to-image    |200    |FP32    | InnerProduct |100,000,000|
|yandex-deep    |96    |FP32    |Euclidean    |1,000,000,000|

# 2. 测试环境

* CPU：84 Core（Intel(R) Xeon(R) Platinum 8163 CPU @ 2.50GHz）
* 内存：464 GiB
* 磁盘：SSD
* GCC：8.3.0
* 指令集：Intel skylake-avx512

# 3. 测试说明

* 数据集分别采用 fp32/fp16/int8/int4 四类类型量化方式进行测试
* 索引构建采用 index_builder 工具来进行索引的构建
* 性能测试采用 bench_client 测试工具进行性能测试
* 构建采用图索引类型，相关参数为：max_neighbor_count=64，ef_search=200，ef_construction=500。例如如采用 Int8 量化，索引构建时使用的 schema 如下：

```json
{
  "collection_name": "yandex",
  "index_column_params": [
    {
      "column_name": "feature",
      "index_type": "IT_PROXIMA_GRAPH_INDEX",
      "extra_params": [
        {
          "key": "quantize_type",
          "value": "DT_VECTOR_INT8"
        },
        {
          "key": "engine",
          "value": "HNSW"
        },
        {
          "key": "max_neighbor_count",
          "value": "64"
        }
      ],
      "data_type": "DT_VECTOR_FP32",
      "dimension": 96
    }
  ]
}
```

# 3. 测试结果

## 3.1 sift-128-euclidean

### 3.1.1 构建信息

|量化类型|FP32|FP16|INT8|INT4|
|---|---|---|---|---|
|索引大小|734M|490M|399M|329M|
|构建时间|66s|57s|50s|48s|

### 3.1.2 查询性能

|量化类型|FP32|FP16|INT8|INT4|
|---|---|---|---|---|
|1 并发 QPS/RT | 811/s	1132us |922/s 980us| 982/s	932us | 1058/s	879us |
|8 并发 QPS/RT | 6533/s 1212us |7326/s	1084us|7868/s 1011us|8636/s	924us |
|16 并发 QPS/RT| 12273/s 1339us | 14579/s 1165us | 15860/s 1077us | 15997/s 963us |
|24 并发 QPS/RT| 15846/s 1454us | 18820/s 1261us | 20171/s	1140us | 22213/s 1029us
|32 并发 QPS/RT| 20671/s 1562us | 23389/s 1343us | 26223/s	1192us | 28353/s 1089us
|48 并发 QPS/RT| 24579/s 1898us |29864/s	1547us | 34159/s 1364us | 38135/s 1231us |
|64 并发 QPS/RT| 26829/s 2301us |35575/s 1723us | 41072/s 1512us | 45778/s 1379us |

![QPS](/images/sift-128-euclidean-qps.png)

![RT](/images/sift-128-euclidean-rt.png)

### 3.1.3 召回率

|量化类型|FP32|FP16|INT8|INT4|
|---|---|---|---|---|
| Top 1 召回率 | 99.99% | 99.99%| 96.33%|    61.46%|
| Top 10 召回率 | 99.94%| 99.94%| 97.25%|    71.88%|
| Top 20 召回率 | 99.89%| 99.89%| 97.54%|    74.48%|
| Top 30 召回率 | 99.84%| 99.84%| 97.68%|    75.76%|
| Top 40 召回率 | 99.79%| 99.79%| 97.82%|    76.58%|
| Top 50 召回率 | 99.74%| 99.74%| 97.85%|    77.28%|
| Top 60 召回率 | 99.68%| 99.68%| 97.88%|    77.82%|
| Top 70 召回率 | 99.61%| 99.61%| 97.90%|    78.21%|
| Top 80 召回率 | 99.54%| 99.54%| 97.91%|    78.58%|
| Top 90 召回率 | 99.47%| 99.46%| 97.91%|    78.90%|
| Top 100 召回率 | 99.39%|99.39%| 97.92%|    79.19%|

![Recall](/images/sift-128-euclidean-recall.png)

## 3.2 gist-960-euclidean

### 3.2.1 构建信息

|量化类型|FP32|FP16|INT8|INT4|
|---|---|---|---|---|
|索引大小|3.8G|2.0G|1.2G|686M|
|构建时间|411s|257s|164s|157s|

### 3.2.2 查询性能

|量化类型|FP32|FP16|INT8|INT4|
|---|---|---|---|---|
|1 并发 QPS/RT | 221/s	4230us    |235/s	3978us    |391/s	2518us    |400/s	2398us|
|8 并发 QPS/RT | 1722/s	4625us    |1868/s	4257us    |3051/s	2593us    |3204/s	2483us|
|16 并发 QPS/RT| 2948/s	5179us    |3676/s	4410us    |5920/s	2694us    |6224/s	2550us|
|24 并发 QPS/RT| 3835/s	6333us    |4876/s	4951us    |7975/s	3046us    |8689/s	2744us|
|32 并发 QPS/RT| 4218/s	7652us    |6135/s	5242us    |10005/s	3179us    |11013/s	2870us|
|48 并发 QPS/RT| 4308/s	11180us    |6935/s	6956us    |11896/s	4054us    |14549/s	3302us|
|64 并发 QPS/RT| 4396/s	14622us    |7295/s	8817us    |12418/s	5077us    |17216/s	3761us|

![QPS](/images/gist-960-euclidean-qps.png)

![RT](/images/gist-960-euclidean-rt.png)

### 3.2.3 召回率

|量化类型|FP32|FP16|INT8|INT4|
|---|---|---|---|---|
| Top 1 召回率 | 99.68%    |99.66%    |96.06%    |55.64%|
| Top 10 召回率 | 99.43%    |99.34%    |96.76%    |66.81%|
| Top 20 召回率 | 99.19%    |99.10%    |96.96%    |69.53%|
| Top 30 召回率 | 98.98%    |98.91%    |97.00%    |70.96%|
| Top 40 召回率 | 98.78%    |98.70%    |97.02%    |71.91%|
| Top 50 召回率 | 98.57%    |98.50%    |96.94%    |72.48%|
| Top 60 召回率 | 98.38%    |98.32%    |96.86%    |73.02%|
| Top 70 召回率 | 98.18%    |98.12%    |96.76%    |73.47%|
| Top 80 召回率 | 97.98%    |97.92%    |96.68%    |73.82%|
| Top 90 召回率 | 97.79%    |97.73%    |96.57%    |74.11%|
| Top 100 召回率 | 97.63%|97.57%    |96.48%    |74.40%|

![Recall](/images/gist-960-euclidean-recall.png)

## 3.3 deep-image-96-angular

### 3.3.1 构建信息

|量化类型|FP32|FP16|INT8|INT4|
|---|---|---|---|---|
|索引大小|6.3G|4.6G|4.0G|3.3G|
|构建时间|854s|671s|597s|550s|

### 3.3.2 查询性能

|量化类型|FP32|FP16|INT8|INT4|
|---|---|---|---|---|
|1 并发 QPS/RT | 646/s	1547us    |750/s	1333us    |729/s	1352us    |736/s	1345us|
|8 并发 QPS/RT | 4910/s	1602us    |5891/s	1349us    |5756/s	1410us    |5817/s	1367us|
|16 并发 QPS/RT| 9100/s	1696us    |11055/s	1408us    |10474/s	1462us    |11222/s	1449us|
|24 并发 QPS/RT| 12915/s	1860us    |15766/s	1520us    |15379/s	1560us    |15679/s	1538us|
|32 并发 QPS/RT| 15917/s	1998us    |19746/s	1611us    |19252/s	1659us    |19545/s	1625us|
|48 并发 QPS/RT| 19863/s	2423us    |25792/s	1852us    |24753/s	1929us    |25421/s	1883us|
|64 并发 QPS/RT| 22562/s	2825us    |30531/s	2101us    |29083/s	2184us    |30160/s	2107us|

![QPS](/images/deep-image-96-angular-qps.png)

![RT](/images/deep-image-96-angular-rt.png)

### 3.3.3 召回率

|量化类型|FP32|FP16|INT8|INT4|
|---|---|---|---|---|
| Top 1 召回率 | 99.68%    |99.66%    |96.06%    |55.64%|
| Top 10 召回率 | 99.43%    |99.34%    |96.76%    |66.81%|
| Top 20 召回率 | 99.19%    |99.10%    |96.96%    |69.53%|
| Top 30 召回率 | 98.98%    |98.91%    |97.00%    |70.96%|
| Top 40 召回率 | 98.78%    |98.70%    |97.02%    |71.91%|
| Top 50 召回率 | 98.57%    |98.50%    |96.94%    |72.48%|
| Top 60 召回率 | 98.38%    |98.32%    |96.86%    |73.02%|
| Top 70 召回率 | 98.18%    |98.12%    |96.76%    |73.47%|
| Top 80 召回率 | 97.98%    |97.92%    |96.68%    |73.82%|
| Top 90 召回率 | 97.79%    |97.73%    |96.57%    |74.11%|
| Top 100 召回率 | 97.63%    |97.57%    |96.48%    |74.40%|

![Recall](/images/deep-image-96-angular-recall.png)

## 3.4 yandex-text-to-image

### 3.4.1 构建信息

|量化类型|FP32|FP16|INT8|INT4|
|---|---|---|---|---|
|索引大小|100G|64G|46G|37G|
|构建时间|4h|2.7h|2.5h|2.1h|

### 3.4.2 查询性能

|量化类型|FP32|FP16|INT8|INT4|
|---|---|---|---|---|
|1 并发 QPS/RT | 500/s	1999us    |530/s	1942us    |565/s	1821us    |572/s	1765us|
|8 并发 QPS/RT | 3839/s	2151us    |4175/s	1967us    |4251/s	1846us    |4366/s	1832us|
|16 并发 QPS/RT| 6606/s	2275us    |7207/s	2067us    |8023/s	1915us    |8404/s	1881us|
|24 并发 QPS/RT| 9746/s	2534us    |10172/s	2280us    |11246/s	2082us    |11887/s	2060us|
|32 并发 QPS/RT| 11368/s	2847us    |13136/s	2375us    |14709/s	2189us    |15081/s	2140us|
|48 并发 QPS/RT| 12888/s	3708us    |16914/s	2894us    |18466/s	2591us    |19356/s	2467us|
|64 并发 QPS/RT| 13523/s	4711us    |18051/s	3517us    |20718/s	3018us    |22637/s	2823us|

![QPS](/images/yandex-text-to-image-qps.png)

![RT](/images/yandex-text-to-image-rt.png)

### 3.4.3 召回率

|量化类型|FP32|FP16|INT8|INT4|
|---|---|---|---|---|
| Top 1 召回率 |99.80%    |99.75%    |98.07%    |69.50% |
| Top 10 召回率 |99.80%    |99.69%    |97.78%    |74.08% |
| Top 20 召回率 |99.72%    |99.59%    |97.73%    |75.07% |
| Top 30 召回率 |99.61%    |99.48%    |97.91%    |75.83% |
| Top 40 召回率 |99.52%    |99.39%    |97.84%    |76.58% |
| Top 50 召回率 |99.44%    |99.31%    |97.92%    |77.11% |
| Top 60 召回率 |99.35%    |99.22%    |97.90%    |77.56% |
| Top 70 召回率 |99.26%    |99.13%    |97.87%    |77.95% |
| Top 80 召回率 |99.15%    |99.03%    |97.85%    |78.31% |
| Top 90 召回率 |99.08%    |98.96%    |97.85%    |78.62% |
| Top 100 召回率 |98.99%    |98.87%    |97.77%    |78.83% |

![Recall](/images/yandex-text-to-image-recall.png

## 3.5 yandex-deep

### 3.5.1 构建信息

|量化类型|INT8|
|---|---|
|索引大小|386G|
|构建时间|71h|

### 3.5.2 查询性能

|量化类型|INT8|
|---|---|
|1 并发 QPS/RT | 417/s	2398us |
|8 并发 QPS/RT |3393/s	2574us |
|16 并发 QPS/RT| 5886/s	2793us |
|24 并发 QPS/RT| 7368/s	3257us |
|32 并发 QPS/RT| 8085/s	3918us |
|48 并发 QPS/RT| 8933/s	5556us |
|64 并发 QPS/RT| 8536/s	7554us |

![QPS](/images/yandex-deep-qps.png)

![RT](/images/yandex-deep-rt.png)

### 3.5.3 召回率

|量化类型|INT8|
|---|---|
| Top 1 召回率 | 92.07% |
| Top 10 召回率 | 93.11% |
| Top 20 召回率 | 93.05% |
| Top 30 召回率 | 92.88% |
| Top 40 召回率 | 92.59% |
| Top 50 召回率 | 92.27% |
| Top 60 召回率 | 91.96% |
| Top 70 召回率 | 91.63% |
| Top 80 召回率 | 91.30% |
| Top 90 召回率 | 91.00% |
| Top 100 召回率 | 90.64% |

![Recall](/images/yandex-deep-recall.png)

