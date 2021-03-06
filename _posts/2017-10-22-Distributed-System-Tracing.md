---
date: 2017-10-22 18:42
status: public
title: 分布式系统-全链路追踪系统
---

## 参考文章:

http://blog.jobbole.com/69642/

http://www.cnblogs.com/binyue/p/5703812.html

http://blog.csdn.net/manzhizhen/article/details/52811600

http://blog.csdn.net/manzhizhen/article/details/53865368

http://blog.csdn.net/u011277123/article/details/71402810

## 存在的必要性

互联网业务中日益复杂的分布式系统，一个前端(用户)的请求，需要经过多个系统处理,如果由分布式追踪系统，对
每次请求可以进行跟踪、明确请求经过的系统、耗时、异常等。

本人曾在QQ基础后台工作，QQ基础后台涉及的后台模块由上百个，用户在客户端的一次请求可能要经由十多个系统处理，每一步
都由可能出问题、异常、超时等。由于历史原因，QQ基础后台没有分布式追踪系统把用户的请求在各个模块的信息关联起来，处理
问题时，只能每个模块的负责人查自己模块的日志，如果发现下游的模块返回异常或者超时，再让下游模块负责人查日志，定位问题
时比较痛苦。

幸运的是QQ后台有统一日志系统，monitor打点监控（可以设置告警条件，波动，最低值，最大值等)， 不过这需要应用代码中，开发人员在代码种开发，如果开发时没有做好Log记录，
属性监控上报,就无法及时发现问题，定位问题。

为何QQ后台没有一套分布式链路追踪系统？曾经和QQ后台的同事们讨论，主要是几个说法：

1. 无统一RPC框架，业务模块多，一些模块间调用采用原始的业务组包、网络收发包、业务解包的方法
2. 协议不统一, QQ后台有二进制协议,protobuf协议, json协议等
3. 模块多、历史久，改动风险大、成本高


不仅仅QQ，腾讯内部都没有这在分布式追踪系统，只有WX内有个WX内自己用的分布式追踪系统

## 业界有名的分布式追踪系统

1. google   dapper
2. twitter  zipkin
3. alibaba  鹰眼


## 分布式追踪系统设计目标

1. 低侵入性、应用层透明，减少开发人员负担, 这里如果没有统一的rpc框架、比较难以实现,例如QQ

2. 低损耗，实际应用时可以配置采样

3. 扩展，本身时一个分布式系统，要考虑扩展、容灾

## 分布式追踪系统几个主要流程

1. 埋点、日志产生. 日志通常要包含：traceId, rpcId, 调用开始时间、耗时、协议命令、协议类型，调用方ip.port.
请求的服务名，服务方ip.port ,调用结果，异常信息，可扩展字段，应用设置字段

2. 日志抓取、日志存储.  分散在各个服务器的日志汇总，可以参考分布式日志采集系统,同一个traceId的日志要汇总在一起

3. 分析、统计调用链路. 按traceId汇总，然后按rpcid进行调用链顺序整理

4. 计算、展示


## 分布式追踪系统的应用场景

1. 异常定位，耗时定位

2. 链路分析： 拓扑形态分析，识别不合理来源；

3. 链路分析：识别易故障点、性能瓶颈，强依赖;

4. 链路分析: 容量估算，QPS统计；

5. 链路分析：个体异常识别




