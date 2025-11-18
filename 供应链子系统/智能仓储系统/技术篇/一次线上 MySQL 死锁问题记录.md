
## 一、现象

海外日本仓现场遇到一个问题，现场反馈某个工作站在做入库业务，但调度来的车辆不离站了。

## 二、问题分析

此前关于入库车辆不离站的问题已经发生过许多次了，但多数时候发生在开仓阶段，由于现场配置的问题，导致的车辆调度问题。

但是这一次有点不一样，我查看了现场所有的配置，都是正常的。

只能从业务流程分析了，我捋了入库业务逻辑链路：
1. 入库车辆到站，调度系统给仓储执行系统发送到站消息
2. 储执行系统进行业务处理，封装成实操任务推送给工作站系统
3. 工作站系统任务引擎调度实操任务，按照工作流形式提示仓储人员绑箱、绑库位
4. 绑箱、绑库位后，工作站系统通知仓储执行系统进行实操反馈
5. 仓储执行系统进行业务处理，通知下游车辆调度系统，车辆离站

现在现象是车不走，我按链路流程逐一检查，排除了 1、2、3，定位到问题出现在步骤 4 的实操反馈上。

## 三、关键信息

根据上下文排查到有死锁报错日志，所以立刻查看数据库死锁日志：
> SHOW ENGINE INNODB STATUS\G


捞出来死锁日志，日志很长，重点看：

```java
------------------------

LATEST DETECTED DEADLOCK

------------------------


*** (1) TRANSACTION:

TRANSACTION 994952163, ACTIVE 0 sec starting index read

mysql tables in use 1, locked 1

LOCK WAIT 11 lock struct(s), heap size 1136, 6 row lock(s), undo log entries 6

MySQL thread id 1380714, OS thread handle 140592418154240, query id 12281738268 172.16.12.200 root updating

update 入库单明细表 d

        set d.combined_quantity  = IF((IFNULL(d.combined_quantity, 0) + -240) >0 , (IFNULL(d.combined_quantity, 0) + -240), 0)

        where d.id = 18075

        and d.warehouse_id = 1

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 2153 page no 410 n bits 128 index PRIMARY of table `入库单明细表` trx id 994952163 lock_mode X locks rec but not gap waiting

Record lock, heap no 55 PHYSICAL RECORD: n_fields 46; compact format; info bits 0

 



*** (2) TRANSACTION:

TRANSACTION 994952206, ACTIVE 0 sec starting index read, thread declared inside InnoDB 5000

mysql tables in use 1, locked 1

6 lock struct(s), heap size 1136, 3 row lock(s), undo log entries 2

MySQL thread id 1381375, OS thread handle 140592409773824, query id 12281738303 172.16.12.200 root updating

update evo_wes_replenish.replenish_work_detail

        set fulfill_quantity = IFNULL(fulfill_quantity, 0) + 240

        where warehouse_id = 1

          and id = 271602

*** (2) HOLDS THE LOCK(S):

RECORD LOCKS space id 2153 page no 410 n bits 128 index PRIMARY of table `入库单明细表` trx id 994952206 lock_mode X locks rec but not gap

Record lock, heap no 55 PHYSICAL RECORD: n_fields 46; compact format; info bits 0


*** (2) WAITING FOR THIS LOCK TO BE GRANTED:

RECORD LOCKS space id 2161 page no 1142 n bits 136 index PRIMARY of table `入库作业单明细表` trx id 994952206 lock_mode X locks rec but not gap waiting

Record lock, heap no 65 PHYSICAL RECORD: n_fields 33; compact format; info bits 0

*** WE ROLL BACK TRANSACTION (2)

------------

TRANSACTIONS

------------

```

## 四、信息梳理

从死锁日志中，我们结合日志，定位到了报错的代码，是取消组箱与整箱上架同一个入库单明细时，两个逻辑加锁顺序不一致导致了死锁。

```yaml
┌──────────────────────────────┐
│        事务 (1)               │
│  TRANSACTION 994952163        │
│  SQL: 更新 入库单明细表 d     │
│  WHERE id = 18075             │
└─────────────┬────────────────┘
              │
              │ 持有锁：入库作业单明细表 (space id 2161, heap no 65)
              │ 等待锁：入库单明细表 (space id 2153, heap no 55)
              ▼
┌──────────────────────────────┐
│        事务 (2)               │
│  TRANSACTION 994952206        │
│  SQL: 更新 replenish_work_detail│
│  WHERE id = 271602            │
└─────────────┬────────────────┘
              │
              │ 持有锁：入库单明细表 (space id 2153, heap no 55)
              │ 等待锁：入库作业单明细表 (space id 2161, heap no 65)
              ▼
          [死锁形成，事务(2)回滚]

```

 死锁链路解释

1. **事务 (1)**
    - 正在更新 `入库单明细表`（id=18075），需要获取 `PRIMARY` 索引行锁（space id 2153, heap no 55）。
    - 已经持有 `入库作业单明细表`（space id 2161, heap no 65）的行锁。
2. **事务 (2)**
    - 正在更新 `replenish_work_detail`，但在执行过程中持有了 `入库单明细表`（space id 2153, heap no 55）的行锁。
    - 同时想获取 `入库作业单明细表`（space id 2161, heap no 65）的行锁。
3. **循环等待**
    - 事务 (1) 等事务 (2) 释放 **入库单明细表** 锁。
    - 事务 (2) 等事务 (1) 释放 **入库作业单明细表** 锁。
    - MySQL 检测到循环等待 → 回滚事务 (2)。

## 五、问题解决

其实知道知道了具体的问题，还蛮好解决死锁的，无非是破坏死锁的 4 个必要条件：
1. 互斥条件

资源在同一时刻只能被一个事务（或线程）占用，其他事务必须等待。

> 在 MySQL 中，行锁、表锁等都满足互斥性。


 2. 请求与保持条件

事务已经持有了至少一个资源（锁），同时又去申请新的资源，并且在等待过程中不释放已有的资源。

> 
> - 事务 (1) 持有 `入库作业单明细表` 的锁，还要申请 `入库单明细表` 的锁。
>     
> - 事务 (2) 持有 `入库单明细表` 的锁，还要申请 `入库作业单明细表` 的锁。
>     



 3. 不可剥夺条件

资源（锁）一旦被事务持有，在事务自己释放之前，其他事务不能强行夺走。

> MySQL 不会强制中断一个持锁事务去抢锁。

---

 4. 循环等待条件

存在一个事务等待链，链上的事务相互等待对方持有的资源，形成一个环路。

> 
> - 事务 (1) 等 事务 (2) 的锁
>     
> - 事务 (2) 等 事务 (1) 的锁  
>     → 环形等待
>     

我们修改了代码，将申请锁的顺序保持一致即可：
所有业务都先更新入库单明细，再更新入库作业单明细。

