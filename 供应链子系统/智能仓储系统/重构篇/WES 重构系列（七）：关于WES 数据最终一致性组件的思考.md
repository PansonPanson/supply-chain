
## 一、WES 与第三方系统数据不一致困境
首先说一下智能仓储系统的场景：智能仓储系统的核心链路涉及多个系统，但是对于数据最终一致性有要求，且部分场景需要提供补偿机制。

我们需要与诸多二方系统和三方系统对接，比如：
1、车辆调度系统，可能就会遇到：
  + 车不来：下发调度信息的时候，消息发送失败，导致车辆不来
  + 来错车：下发调度信息的时候，消息乱序，导致来错车
  + 车不走：下发车辆离站消息的时候，消息发送失败，导致车辆不走
  
2、与算法服务对接
  + 离线任务下发失败
  + 调用算法计算热度
  
3、与外设系统交互，可能会遇到：
  + 灯不亮：
    + 发送亮灯消息的时候，消息发送失败，导致外设系统未接收到消息，灯不亮
    + 外设系统与物理设备交互，调用相关接口失败
  + 灯不灭：
    + 发送灭灯消息的时候，消息发送失败，导致外设系统未接收到消息，灯不灭
    + 外设系统与物理设备交互，调用相关接口失败，灯不灭

  + 亮错灯：
    + 发送亮灯消息的时候，消息乱序，导致外设系统亮灯错乱
    
4、与打印系统交互，可能会遇到：
  + 没打印：接口调用失败，导致单据打印失败

5、与上游系统交互：
  + 各种单据的实操结果未正常反馈上游
    + 出库单按单反馈
    + 出库单按箱反馈
    + 入库单按单反馈
    + 入库单按箱反馈
    + 盘点单按单反馈
    + 库存调整单按单反馈
    + ……
6、与基础数据系统交互
  + 货架热度计算结果更新失败
  + 料箱热度计算结果更新失败
  + 容器位置更新
![](https://github.com/PansonPanson/supply-chain/blob/main/%E4%BE%9B%E5%BA%94%E9%93%BE%E5%AD%90%E7%B3%BB%E7%BB%9F/%E6%99%BA%E8%83%BD%E4%BB%93%E5%82%A8%E7%B3%BB%E7%BB%9F/%E9%87%8D%E6%9E%84%E7%AF%87/image/7/file-20251129203301152.png?raw=true)
这些场景无法使用本地事务实现，因为是分布式系统。有些场景也不能纯用 MQ 的消息事务实现，因为 RocketMQ 事务消息重试机制不灵活。

## 二、关于重试逻辑的思考

一个健壮的系统是需要考虑关键节点的稳定性的，以与外部系统交互这种节点来讨论，我觉得核心的意识是不相信第三方系统，无论是数据的获取还是推送。

所以需要实现重试逻辑。

但二方系统、三方系统有许多，如果在每一个节点都去写重试逻辑，那么重试逻辑就会变得不可复用。比如说：
+ 重试次数怎么设置？
+ 每次重试的间隔如何考虑？
+ 重试逻辑是否影响主线程，需要异步化吗？
+ 是否考虑加降级呢？
+ 能不能加告警？
+ ……

## 三、抽象、复用与便捷

考虑以上问题后，我在阅读了转转和得物的关于重试组件的技术博客后，开始有了新的思考。是不是可以抽取出业务需求，自定义一个 springboot starter，用户只需要引入这个 maven 包，做一些简单的配置和适配，就可以做到关键节点的自动重试呢？

很幸运，我在网上找了一些类似的教程和代码，结合得物、转转的技术博客，实现了这个组件。

我们可以基于SpringAOP来实现，将需要重试的逻辑抽取成 public 修饰的方法，在这个方法上加上一致性注解。
拦截所有加了一致性注解的方法，封装为一个重试任务，持久化到数据库中，再通过反射去执行这个任务。

![](https://github.com/PansonPanson/supply-chain/blob/main/%E4%BE%9B%E5%BA%94%E9%93%BE%E5%AD%90%E7%B3%BB%E7%BB%9F/%E6%99%BA%E8%83%BD%E4%BB%93%E5%82%A8%E7%B3%BB%E7%BB%9F/%E9%87%8D%E6%9E%84%E7%AF%87/image/7/chongshizujian.png?raw=true)
## 四、如何设计自定义注解

我们如果想基于反射来做，那注解必须要有反射相关的信息，另外还要有执行间隔、延迟时间、告警相关
降级相关。
1. 任务名称：默认取方法全限定名，因为想使用反射来执行
2. 执行间隔：任务执行间隔
3. 初始延迟时间
4. 告警表达式
5. 告警类
6. 降级类
7. ……


## 五、重试任务执行流程

如下图所示：
![](https://github.com/PansonPanson/Argo/blob/main/doc/image/%E6%B5%81%E7%A8%8B%E8%AE%BE%E8%AE%A1.png?raw=true)

## 六、如何自定义任务查询逻辑
任务失败重试是通过定时任务调⽤ `taskScheduleManager.performanceTask()` ⽅法来实现 的，底层逻辑就是根据条件从数据库中查询出来失败的任务，然后判 断是否需要重试，执行后续逻辑。

在这个过程中，根据条件查询失败的任务，这⾥的条件允许⼀定程度的⾃定义。默认情况下⾏为 是： **每次查询当前时间 - 1⼩时 时间范围内的1000条失败的记录**。

如果想要更改此逻辑，可以自定义查询类名并继承查询接口，然后在 yml 中配置全路径类名，组件接入启动时会根据自定义配置类来反射获取自定义查询配置信息。
![](https://github.com/PansonPanson/supply-chain/blob/main/%E4%BE%9B%E5%BA%94%E9%93%BE%E5%AD%90%E7%B3%BB%E7%BB%9F/%E6%99%BA%E8%83%BD%E4%BB%93%E5%82%A8%E7%B3%BB%E7%BB%9F/%E9%87%8D%E6%9E%84%E7%AF%87/image/7/file-20251129204320270.png?raw=true)

## 七、指数退避重试
指数退避重试是一种智能的容错机制，其核心思想是当操作失败后，重试的等待时间随着重试次数的增加而呈指数级增长，并通常会引入随机扰动。它能有效防止因频繁重试导致的系统压力激增，是构建稳定分布式系统和网络应用的关键策略。

|重试次数|基础延迟计算（示例）|实际等待时间（含抖动）|说明|
|---|---|---|---|
|第1次重试|`base_delay * (2^0) = 1s`|1秒 ± 随机时间|初始快速重试，希望问题已瞬时恢复|
|第2次重试|`base_delay * (2^1) = 2s`|2秒 ± 随机时间|延迟加倍，给系统更多恢复时间|
|第3次重试|`base_delay * (2^2) = 4s`|4秒 ± 随机时间|继续指数增长，进一步退让|
|第n次重试|`base_delay * (2^(n-1))`|计算结果 ± 随机时间，但不超过 `max_delay`|避免等待时间无限增长|
在计算下一次执行时间时，可以按照这个指数退避重试，但一般我们设置的重试次数都很小，所以与线性重试差距不大。
![](https://github.com/PansonPanson/supply-chain/blob/main/%E4%BE%9B%E5%BA%94%E9%93%BE%E5%AD%90%E7%B3%BB%E7%BB%9F/%E6%99%BA%E8%83%BD%E4%BB%93%E5%82%A8%E7%B3%BB%E7%BB%9F/%E9%87%8D%E6%9E%84%E7%AF%87/image/7/file-20251129204403321.png?raw=true)
## 八、降级逻辑设计
有时候我们希望重试失败之后进行降级处理，所以注解中要支持定义降级类。
如果配置了降级类，并且超过了重试次数阈值，就调用降级逻辑。
具体的实现方式就是通过反射调用指定降级类的同名方法，方法参数要与原方法一致。

## 九、告警逻辑
告警逻辑可以自定义，在注解上可以配置自定义告警类，如果触发告警规则，则通过反射调用告警类的告警方法。
因为告警可能会比较耗时，所以做了异步化，避免影响主线程。

## 十、如何设计重试任务表

**数据模型：argo_task（任务表）**

|字段名 (Field Name)|数据类型 (Data Type)|允许空值 (Nullable)|默认值 (Default)|注释 (Comment)|
|---|---|---|---|---|
|**id**​|bigint|NOT NULL|AUTO_INCREMENT|主键自增|
|**task_id**​|varchar(500)|NOT NULL|-|用户自定义的任务名称，如果没有则使用方法签名|
|**task_status**​|int|NOT NULL|0|执行状态|
|**execute_times**​|int|NOT NULL|-|执行次数|
|**execute_time**​|bigint|NOT NULL|-|执行时间|
|**parameter_types**​|varchar(255)|NOT NULL|-|参数的类路径名称|
|**method_name**​|varchar(100)|NOT NULL|-|方法名|
|**method_sign_name**​|varchar(200)|NOT NULL|''|方法签名|
|**execute_interval_sec**​|int|NOT NULL|60|执行间隔秒|
|**delay_time**​|int|NOT NULL|60|延迟时间：单位秒|
|**task_parameter**​|varchar(200)|NOT NULL|''|任务参数|
|**performance_way**​|int|NOT NULL|-|执行模式：1、立即执行 2、调度执行|
|**thread_way**​|int|NOT NULL|-|线程模型 1、异步 2、同步|
|**error_msg**​|varchar(200)|NOT NULL|''|执行的error信息|
|**alert_expression**​|varchar(100)|YES|NULL|告警表达式|
|**alert_action_bean_name**​|varchar(255)|YES|NULL|告警逻辑的执行beanName|
|**fallback_class_name**​|varchar(255)|YES|NULL|降级逻辑的类路径|
|**fallback_error_msg**​|varchar(200)|YES|NULL|降级失败时的错误信息|
|**shard_key**​|bigint|YES|0|任务分片键|
|**gmt_create**​|datetime|NOT NULL|-|创建时间|
|**gmt_modified**​|datetime|NOT NULL|-|修改时间|

- **主键 (Primary Key)**：`PRIMARY KEY (id)`。
    
- **唯一索引 (Unique Key)**：`UNIQUE KEY uk_id_shard_key (id, shard_key) USING BTREE`。这是一个复合唯一索引，确保了 `id`和 `shard_key`组合的唯一性，常用于分库分表场景。
    
- **存储引擎与字符集 (Storage Engine & Character Set)**：`ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci`。使用InnoDB引擎，字符集为支持更广范围字符（如emoji）的utf8mb4。











