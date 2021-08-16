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
| flag  | int32 | 断点续传标记，上次收到的最后一个blockIndex, 跳过blockIndex小于flag的交易  |

例

```curl

# 在高度692281 的区块中选取hex中包含'63393139326663353435373766613537'的生交易
curl 'https://stream.metasv.com/block/692281?filter=63393139326663353435373766613537' -H  "Authorization: Bearer YOUR_JWT_ISSUED_BY_METASV"

# 在区块00000000000000000704af34c063d04e2152d57f3925bd0c184797f636128926中寻找hex包含'b8d6dc3b97abe94d'的生交易
curl 'https://stream.metasv.com/block/00000000000000000704af34c063d04e2152d57f3925bd0c184797f636128926?filter=b8d6dc3b97abe94d' -H  "Authorization: Bearer YOUR_JWT_ISSUED_BY_METASV"

# 在区块00000000000000000704af34c063d04e2152d57f3925bd0c184797f636128926中寻找hex包含'b8d6dc3b97abe94d'的生交易，并跳过blockIndex <= 3 的交易
curl 'https://stream.metasv.com/block/00000000000000000704af34c063d04e2152d57f3925bd0c184797f636128926?filter=b8d6dc3b97abe94d&flag=3' -H  "Authorization: Bearer YOUR_JWT_ISSUED_BY_METASV"

```

### /blockHeader

订阅区块头消息

此接口是无限流，通过30秒一次的HEARTBEAT消息，判断连接存活

|  参数   | 类型  | 说明  |
|  ----  | ----  | ----  |
| flag  | int32 | 断点续传标记，上次收到的最后一个高度, 跳过blockIndex小于等于flag的交易，最多可以回溯500个块  |

例

```curl

# 从最新高度开始订阅
curl 'https://stream.metasv.com/blockHeader' -H  "Authorization: Bearer YOUR_JWT_ISSUED_BY_METASV"

# 从高度698576开始订阅（此高度不得小于最新高度-500，否则只订阅最新区块头）
curl 'https://stream.metasv.com/blockHeader?flag=698576' -H  "Authorization: Bearer YOUR_JWT_ISSUED_BY_METASV"

```

返回值示例

```json

data:{"bits":403862476,"blockHash":"00000000000000000da4cdfc91752e510a219f18682ce1c2f96796ac6c08b2e4","coinBase":"03d1a80a2f7461616c2e636f6d2f506c656173652070617920302e3520736174732f627974652c20696e666f407461616c2e636f6d8c1d33989acd493c0e720000","height":698577,"inputCount":454,"medianTime":1627890971000,"merkleRoot":"faa5d6c8c98cd0ddcfb8f00ac99c69d1466557d2ca2fbb5e44ef9799e653e583","miner":"Taal","nonce":2213484660,"outputCount":693,"prevBlock":"00000000000000000a239fe625579031effb38a7768cd7e4fe628f8bbf1febcf","reward":632829727,"size":14258909,"timestamp":1627894128000,"txCount":237,"version":536879104}

```

### /tx

订阅内存池生交易

可以通过此接口来持续监听并过滤内存池以及zmq生交易事件。

此接口是无限流，通过30秒一次的HEARTBEAT消息，判断连接存活。

|  参数   | 类型  | 说明  |
|  ----  | ----  | ----  |
| rewind  | boolean | 是否回溯整个内存池，默认为false，从最新交易开始监听，true意味着先爬取既存的内存池交易，再监听新交易  |
| filter  | string | 过滤器，提取生交易hex中包括filter字符的交易  |


例

```curl

# 从最新交易开始订阅
curl 'https://stream.metasv.com/tx' -H  "Authorization: Bearer YOUR_JWT_ISSUED_BY_METASV"

# 从最新交易开始订阅hex中包含'63393139326663353435373766613537'的生交易
curl 'https://stream.metasv.com/tx?filter=63393139326663353435373766613537' -H  "Authorization: Bearer YOUR_JWT_ISSUED_BY_METASV"

# 先重放整个内存池，过滤出hex包含63393139326663353435373766613537的生交易，然后从最新交易开始订阅hex中包含'63393139326663353435373766613537'的生交易
curl 'https://stream.metasv.com/tx?filter=63393139326663353435373766613537&rewind=true' -H  "Authorization: Bearer YOUR_JWT_ISSUED_BY_METASV"

```

返回值示例

```text

data:010000000166505ea5a55ac6544ecdd890686f32874159eb92db8c2bbbae95ea241bdb782a000000006b48304502210090a855b77738946506c5260b9a313965725dcd8aa306b72fe2e723d43ba6357c02201992e4bc69e93438cc2a73704ae5dee65189d3c648e5b9c72de6fc47695cf461412103c376ceca89f1bcb1f27fa9c0ef1f7c71152ac4932b6850009a6d29bd991ba81bffffffff0123020000000000001976a9142e1c2c72b45586b1d3e90f7df33bc8b64386fa9288ac00000000

```

### To be continued...

更多流推送功能正在开发中，尽情期待

1. 区块头订阅
2. 生交易订阅
3. address订阅
4. xpub订阅
5. sensible订阅
6. ...