# MetaSV stream api

Reactive Server Send Stream(SSE) api for BitcoinSV

MetaSV stream是基于响应式编程思想（Reactive programming）全新设计和构建的非阻塞io流推送api。专门用来在大区块，大数据场景下进行数据的同步和解析。

## 为什么使用非阻塞异步推送

### 背景

在BitcoinSV大区块，高交易量的场景下。使用同步api（例如RestApi）传输生交易，生区块等数据，会存在很高的网络延迟，最终请求同步阻塞在服务器，占用大量内存和带宽等资源，很难提高并发请求处理，也就很难拓展服务。

因此MetaSV重新设计了Stream接口类型，使用异步流推送的方式来推送数据，客户端使用SSE长链接链接到MetaSV endpoint后，MetaSV会持续推送字符串流到客户端，并且根据客户端承载能力调节推送速度，也就是背压（Back Pressure）。

### 使用场景

1. 大区块生数据下载和过滤（包括生区块和生交易）
2. 区块和交易数据实时同步
3. 实时监听链上事件（包括监听未确认交易，监听新区块，监听地址，监听xpub，监听sensible交易等）

用户可以使用流推送接口来`过滤`和`筛选`交易，从MetaSV获取和只自己业务相关的交易，下载到客户端进行分析，直接过滤掉99%的业务无关交易。这样可以极大提高处理性能。

## 使用说明

### 鉴权

Stream API由于消耗资源较大，属于付费接口。因此需要使用jwt或clientPublicKey进行访问。

JWT鉴权： 在Header中加入如下字段进行JWT鉴权，JWT联系MetaSV获取

```
Authorization: Bearer YOUR_JWT_ISSUED_BY_METASV
```

ClientPublicKey 鉴权， 参考以下的内容：

[https://github.com/metasv/metasv-client-signature](https://github.com/metasv/metasv-client-signature)

### API Endpoint域名

[https://stream.metasv.com](https://stream.metasv.com)

## 接口说明

### /block/{blockId}

从指定区块中过滤生交易，下载过滤后的全部生交易

|  参数   | 类型  | 说明  |
|  ----  | ----  | ----  |
| blockId（path）  | string/int | 区块hash或区块高度  |
| filter  | string | 过滤器，提取生交易hex中包括filter字符的交易  |

例

```curl

# 在高度692281 的区块中选取hex中包含'63393139326663353435373766613537'的生交易
curl 'https://stream.metasv.com/block/692281?filter=63393139326663353435373766613537' -H  "Authorization: Bearer YOUR_JWT_ISSUED_BY_METASV"

# 在区块00000000000000000704af34c063d04e2152d57f3925bd0c184797f636128926中寻找hex包含'b8d6dc3b97abe94d'的生交易
curl 'https://stream.metasv.com/block/00000000000000000704af34c063d04e2152d57f3925bd0c184797f636128926?filter=b8d6dc3b97abe94d' -H  "Authorization: Bearer YOUR_JWT_ISSUED_BY_METASV"

```
