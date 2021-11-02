
---
title: "常用工具"
linkTitle: "常用工具"
weight: 16
draft: false
---


在 Proxima BE 发布的镜像中，附带了几个常用工具，主要是方便客户管理 collection 和文档。



## 1. 如何获取

相关工具位于镜像 ghcr.io/proximabilin/proxima-be 的目录 /var/lib/proxima-be/bin/

## 2. admin_client

### 2.1 使用方法

```s
$ admin_client  -h
Usage:
 admin_client <args>

Args:
 --command      Command type: create | drop
 --host         The host of proxima be, http port
 --collection   Specify collection name
 --schema       Specify collection schema format
 --help, -h     Display help info
 --version, -v  Dipslay version info
```



### 2.2 创建Collection

```s
$ admin_client --command create --host 127.0.0.1:16001 --collection test_collection \
    --schema '{"collection_name":"test_collection", "index_column_params":[{"column_name":"test_column", 
    "index_type": "IT_PROXIMA_GRAPH_INDEX", "data_type":"DT_VECTOR_FP32", "dimension":8}]}'
```



### 2.3 删除Collection

```s
$ admin_client --command drop --host 127.0.0.1:16001 --collection test_collection
```



## 3. bench_client

### 3.1 使用方法

```s
$ bench_client -h
Usage:
 bench_client <args>

Args:
 --command        Command type: search|insert|delete|update
 --host           The host of proxima be
 --collection     Specify collection name
 --column         Specify column name
 --file           Read input data from file
 --protocol       Protocol http or grpc
 --concurrency    Send concurrency (default 10)
 --topk           Topk results (default 10)
 --perf           Output perf result (default false)
 --help, -h       Display help info
 --version, -v    Display version info
```



### 3.2 插入数据

```s
$ bench_client --command insert --host 127.0.0.1:16000 --collection test_collection --column test_column --file 
data.txt
```



数据格式支持明文和二进制两种，key与向量之间用";"分隔，多维向量采用空间分割，样例数据如下:

```
0;-0.009256 -0.079674 -0.070349 0.007072 -0.064061 -0.010632 0.083429 -0.074821
1;-0.061519 -0.001263 -0.016528 0.031539 0.041385 -0.017736 -0.005704 0.129443
2;-0.039616 -0.063191 0.057591 -0.090278 -0.007452 -0.035939 -0.021892 -0.037860
3;0.042097 0.050037 0.055060 0.150511 -0.052841 -0.005502 -0.018618 0.054607
```



### 3.3 查询数据

query数据格式同上述的插入数据格式相同。

```s
$ bench_client --command search --host 127.0.0.1:16000 --collection test_collection --column test_column --file 
query.txt
```



### 3.4 删除数据

data数据格式同上述的插入数据格式相同。

```s
$ bench_client --command delete --host 127.0.0.1:16000 --collection test_collection --column test_column --file 
data.txt
```

## 4. index_builder

对于数据集已经提前准备好的场景，为加速索引构建，可通过离线构建工具加速构建。
### 4.1 使用方法
```shell
Usage:
 index_builder <args>

Args:
 --schema           Specify the schema of collection
 --file             Specify input data file
 --output           Sepecify output index directory(default ./)
 --concurrency      Sepecify threads count for building index(default 10)
 --help, -h         Dipslay help info
 --version, -v      Dipslay version info
```

### 4.2 数据文件格式说明
每行一条记录，由 ';' 分隔。分别是 key，向量，正排属性。其中 key 为 uint64 类型，向量各维度用 ' ' 分隔。属性可选。例如
```text
111;1.0 1.1 1.2 1.3;a,b
```

### 4.3 使用示例
test.txt 内容为上述内容的文件。则构建索引可用如下命令:
```shell
index_builder --output index --schema '{"collection_name":"test_collection", "forward_column_names":["k1"], "index_column_params":[{"column_name":"test_column",
"index_type": "IT_PROXIMA_GRAPH_INDEX", "data_type":"DT_VECTOR_FP32", "dimension":4}]}' --file test.txt
```

### 4.3 使用注意
由于离线工具构建的索引没有相应 meta 信息。如果作为 Proxima SE 提供检索能力。需要创建一个对应的 collection。
例如启动 proxima_se 服务时，配置好相应的索引位置为 index:
```yaml
common_config {
  grpc_listen_port: 16000
  http_listen_port: 16001
}
index_config {
    index_directory: "index/"
}
query_config {
  query_thread_count: 8
}
```

启动后，使用如下命令创建相应集合。
```shell
curl -X POST http://0.0.0.0:16001/v1/collection/test_collection -d '{"collection_name":"test_collection", "forward_column_names":["k1"], "index_column_params":[{"column_name":"test_column","index_type": "IT_PROXIMA_GRAPH_INDEX", "data_type":"DT_VECTOR_FP32", "dimension":4}]}'
```

然后进行检索
```shell
curl -X POST \
  http://0.0.0.0:16001/v1/collection/test_collection/query \
 -d '{
    "knn_param":{
        "column_name":"test_column",
        "topk":10,
        "matrix":"[[1.0, 2.0, 3.0, 4.0]]",
        "batch_count":1,
        "dimension":4,
        "data_type":"DT_VECTOR_FP32",
        "is_linear":true,
    }
}'

# 返回
{"status":{"code":0,"reason":"Success"},"results":[{"documents":[{"score":11.34,"primary_key":"111","forward_column_values":[{"key":"k1","value":{"string_value":"a,b"}}]}]}],"debug_info":"","latency_us":"902"}
```

