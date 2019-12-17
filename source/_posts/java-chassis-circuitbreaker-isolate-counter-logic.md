---
title: 【转载】CSEJavaSDK重试和隔离如何判断错误？
tags: [ ServiceComb-Java-Chassis,CSE,microservice ]
date: 2019-12-17 21:45:36
categories:
- 软件技术
<!-- 留一个首页缩略图和文章页顶部大图的例子在这里，万分感谢 http://www.polayoutu.com ~  : ) -->
index_img: http://ppe.oss-cn-shenzhen.aliyuncs.com/collections/176/3/thumb.jpg
banner_img: http://ppe.oss-cn-shenzhen.aliyuncs.com/collections/176/3/thumb.jpg
---
mark一篇CSEJavaSDK熔断、隔离判断逻辑的帖子，同样适用于ServiceComb-Java-Chassis。

原文链接： https://bbs.huaweicloud.com/forum/thread-12790-1-1.html
<!-- more -->

# [技术交流]CSE重试和隔离如何判断错误？

> 简单的描述：
> 
> 1. 网络错误 + 503错误码等情况可以触发重试。
> 
> 2. 网络错误 + 503错误码 + 超时等错误可以触发隔离。



细节内容可以参考代码：

## 负载均衡模块能力

LB模块具备请求重试和实例隔离能力。

- 请求重试的条件参考`org.apache.servicecomb.loadbalance.DefaultRetryExtensionsFactory`
- 实例隔离的条件参考`org.apache.servicecomb.loadbalance.LoadbalanceHandler#isFailedResponse`
- 实例隔离状态恢复的逻辑参考`org.apache.servicecomb.loadbalance.filter.IsolationDiscoveryFilter#allowVisit`
- Java-Chassis定期ping机制会在某实例近期没有被调用时定时去检查其状态，该检查如果失败会产生和调用失败一样的错误计数，但是检查成功不会触发实例隔离恢复，相关逻辑在`org.apache.servicecomb.loadbalance.ServiceCombLoadBalancerStats#timer`的定时任务中。

## 熔断隔离模块能力

CSE还提供了bizkeeper模块，这个模块集成了Hystrix的隔离仓等能力，这个隔离能力是针对微服务或者接口级别的，在实际业务系统中，发挥作用比较少。 这个模块也有判读错误条件：

- consumer: 490异常。 比如网络操作、超时等。 参考`org.apache.servicecomb.bizkeeper.ConsumerBizkeeperCommand#isFailedResponse`
- provider: 590异常。比如业务处理抛出的未知RuntimeException等。 参考`org.apache.servicecomb.bizkeeper.ProviderBizkeeperCommand#isFailedResponse`
