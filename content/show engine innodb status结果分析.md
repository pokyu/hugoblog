---
title: "My First Post"
date: 2017-12-14T11:18:15+08:00
weight: 70
keywords: ["hugo"]
description: "第一篇文章"
tags: ["hugo", "pages"]
categories: ["pages"]
author: ""
---


# show engine innodb status结果分析

https://blog.csdn.net/github_26672553/article/details/52931263
https://blog.csdn.net/happy_life123/article/details/45971349


## 头部状态信息
```
=====================================
2018-06-28 10:54:29 7f25431fe700 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 27 seconds
```
2018-06-28 10:54:29 : 当前时间
per second averages calculated from the last 27 seconds : 意思是说接下来的信息，是对过去27秒时间内innodb状态的输出。


## 后台进程
Innodb存储引擎室多线程的模型，因此其后台有多个不同的后台线程负责处理不同的任务。Master thread是一个非常核心的后台线程，主要负责缓冲池中的数据异步刷新到磁盘，保证数据的一致性。Master thread具有最高的线程优先级别，其内部由多个循环(loop)组成:主循环，后台循环，刷新循环，暂停循环。
```
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 6249261 srv_active, 0 srv_shutdown, 1693104 srv_idle
srv_master_thread log flush and writes: 7942146
```

 - srv_master_thread loops : Master线程的循环次数，master线程-在每次loop过程中都会sleep，sleep的时间为1秒。而在每次loop的过程中会选择active、shutdown、idle中一种状态执行。Master线程在不停循环，所以其值是随时间递增的。
 - srv_active : Master线程选择的active状态执行。Active数量增加与数据表、数据库更新操作有关，与查询无关，例如：插入数据、更新数据、修改表等。
 - srv_shutdown : 这个参数的值一直为0，因为srv_shutdown只有在mysql服务关闭的时候才会增加。
 - srv_idle : 这个参数是在master线程空闲的时候增加，即没有任何数据库改动操作时。
 - Log_flush_and_write : Master线程在后台会定期刷新日志，日志刷新是由参数innodb_flush_log_at_timeout参数控制前后刷新时间差。
注:Background thread部分信息为统计信息，即mysql服务启动之后该部分值会一直递增，因为它显示的是自mysqld服务启动之后master线程所有的loop和log刷新操作。通过对比active和idle的值，可以获知系统整体负载情况。Active的值越大，证明服务越繁忙。


## SEMAPHORES
第一部分是当前的等待，这部分只是包含了在高并发环境下的全部记录(这些记录不包括sleep的)。
```
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 1577328
OS WAIT ARRAY INFO: reservation count 1577329
OS WAIT ARRAY INFO: reservation count 1577329
OS WAIT ARRAY INFO: reservation count 1577329
OS WAIT ARRAY INFO: reservation count 1577329
OS WAIT ARRAY INFO: reservation count 1577329
OS WAIT ARRAY INFO: reservation count 1577328
OS WAIT ARRAY INFO: reservation count 1577328
OS WAIT ARRAY INFO: reservation count 1577328
OS WAIT ARRAY INFO: reservation count 1577328
OS WAIT ARRAY INFO: reservation count 1577328
OS WAIT ARRAY INFO: reservation count 1577328
OS WAIT ARRAY INFO: reservation count 1577328
OS WAIT ARRAY INFO: reservation count 1577328
OS WAIT ARRAY INFO: reservation count 1577328
OS WAIT ARRAY INFO: reservation count 1577328
OS WAIT ARRAY INFO: reservation count 1577328
OS WAIT ARRAY INFO: reservation count 1577328
OS WAIT ARRAY INFO: reservation count 1577328
OS WAIT ARRAY INFO: reservation count 1577328
OS WAIT ARRAY INFO: reservation count 1577328
OS WAIT ARRAY INFO: reservation count 1577328
OS WAIT ARRAY INFO: reservation count 1577328
OS WAIT ARRAY INFO: reservation count 1577328
OS WAIT ARRAY INFO: signal count 1170527971
```

 - OS WAIT ARRAY INFO : 系统等待队列信息
 - Reservation count : 线程尝试访问os wait array的次数，大于等于线程进入os wait状态的线程数(因为尝试放入os wait array可能不成功，不成功的时候reservation count也会++)。
 - Signal count : 线程被唤醒的次数，进入os wait的线程，在占用资源线程释放mutex的时候会通过signal唤醒等待线程。

第二部分是事件统计。相比系统等待，自旋锁的开销较小，但是它是活跃的等待，会浪费CPU资源，如果有大量的自旋等待和自旋轮转，则会浪费大量的CPU资源。
```
Mutex spin waits 2412082675, rounds 6217918137, OS waits 8718751
RW-shared spins 284558481, rounds 1200039502, OS waits 23135175
RW-excl spins 46662334, rounds 1902497845, OS waits 3293683
Spin rounds per wait: -3.30 mutex, 4.22 RW-shared, 40.77 RW-excl
```

 - Mutex spin wait : 线程自旋等待次数，线程在获取mutex过程中如果没有获取到mutex，则首先进入自旋状态，这个时候mutex spin wait值++。
 - Mutex Rounds : 进入spin wait的线程，通过不断循环来等待获取mutex，循环的次数称为round(理解可以参照图 4)。
 - Mutex Os wait : 线程进入系统等待的次数，现在在获取mutex过程中，如果没有在第一时间内获取，则进入自旋，自旋达到设定的时间需求后依旧没能获取到mutex，这个时候线程进入系统等待。Os wait的值增加。
Rw-share和rw-excl：对应参数含义类似。

 - Spin rounds per waits mutex : 每次mutex自旋等待中round的次数，值＝mutex rounds/mutex spin wait
 - Spin rounds per waits rw-share(读锁) : 每次rw-sahre自旋等待中round的次数，值＝rw-share rounds/rw-share spin wait
 - Spin rounds per waits rw-excl(写锁) : 每次rw-excl自旋等待中round的次数,值＝rw-excl rounds/rw-excl spin wait

注：统计信息部分给出的是自mysqld服务启动以来的累计值，进入spin wait的线程不一定会进入os wait状态，但是进入os wait状态的线程必然经历过spin wait状态。如果出现spin wait的次数与os wait次数相等或者相差不大的情况，证明绝大多数线程在spin wait状态结束后最终进入os wait状态，这种情况与innodb mutex设计初衷相违背。如果出现这样的情况，第一证明系统处于高负载，资源竞争激烈；第二，可以通过改变参数innodb_spin_wait_delay和innodb_sync_spin_loops的值来减少资源浪费。Innodb_sync_spin_loops控制的是进入spin wait状态之后循环次数，innodb_spin_wait_delay是为了延长循环时间的。统计信息给的并非实时状态，而是历史状态，如果需要观察实时状态，则必须记录两个时间点差值，这样才能充分反映当前系统状态。非高并发情况下是看不到第一部分的等待信息的，所以当你看到有第一部分信息出现的时候证明你需要关注系统状态。


```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2018-06-28 10:44:12 7f25430d8700
*** (1) TRANSACTION:
TRANSACTION 286314896, ACTIVE 2 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 6 lock struct(s), heap size 1184, 3 row lock(s), undo log entries 2
MySQL thread id 34185587, OS thread handle 0x7f258696b700, query id 730892184 10.80.48.5 crm_user updating
update QL_ACTIVITIES_TASK set EXEC_STATUS=2, UPDATE_TIME='2018-06-28 10:44:10' where ID='e155f350240244879edf3ec2572c0b46' and DEL_STATUS=0
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 1155 page no 1421 n bits 112 index `PRIMARY` of table `tkcn`.`ql_activities_task` trx table locks 3 total table locks 2  trx id 286314896 lock_mode X locks rec but not gap waiting lock hold time 0 wait time before grant 0 
*** (2) TRANSACTION:
TRANSACTION 286314796, ACTIVE 3 sec starting index read, thread declared inside InnoDB 5000
mysql tables in use 1, locked 1
5 lock struct(s), heap size 1184, 2 row lock(s), undo log entries 2
MySQL thread id 34188405, OS thread handle 0x7f25430d8700, query id 730892204 10.80.48.4 crm_user updating
update QL_LIAISON_HISTORY set ACTIVE_CALL='01061048789', ACTIVITY_ID='ff808081605523f90160743d6c510923', AGENT_ID='38758', CUSTOMER_ID='69b61e268f3f4778b95cbb8cf1ebe5d7', CUSTOMER_NAME='谭成波', DATA_SOURCE=1, END_TIME='2018-06-28 10:44:09', LIAISON_HISTORY_INFO_ID='cef7dd7f40a74440a31b90a3b6f7d1a2', LIAISON_TYPE=9, ORGAN_CODE='tkzx-wdzx-bj--zh1z', ORGAN_ID='ff80808160551f8f016073f0354c049e', PASSIVE_CALL='13996697534', SERVICE_STATUS=1, START_TIME='2018-06-28 10:43:30', TASK_ID='e155f350240244879edf3ec2572c0b46', TRAFFIC_STATUS=0, USER_ID='ff808081612db13201622303707518bb', USER_NAME='刘志刚' where ID='A109C14D0EC2551018B2467C270AD3AD'
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 1155 page no 1421 n bits 112 index `PRIMARY` of table `tkcn`.`ql_activities_task` trx table locks 3 total table locks 2  trx id 286314796 lock_mode X locks rec but not gap lock hold time 1 wait time before grant 1 
*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 1165 page no 10841 n bits 120 index `PRIMARY` of table `tkcn`.`ql_liaison_history` trx table locks 3 total table locks 3  trx id 286314796 lock_mode X locks rec but not gap waiting lock hold time 0 wait time before grant 0 
*** WE ROLL BACK TRANSACTION (2)
------------
TRANSACTIONS
------------
Trx id counter 286365836
Purge done for trx's n:o < 286365611 undo n:o < 0 state: running but idle
History list length 994
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 0, not started
MySQL thread id 34191417, OS thread handle 0x7f25431fe700, query id 731026330 10.95.0.10 dba init
show engine innodb status
---TRANSACTION 286365659, not started
MySQL thread id 34191601, OS thread handle 0x7f2597476700, query id 731026029 10.80.48.4 crm_user cleaning up
---TRANSACTION 286365560, not started
MySQL thread id 34191603, OS thread handle 0x7f2586909700, query id 731026267 10.80.48.4 crm_user cleaning up
---TRANSACTION 286365640, not started
MySQL thread id 34191602, OS thread handle 0x7f25dac76700, query id 731026268 10.80.48.4 crm_user cleaning up
---TRANSACTION 286365387, not started
MySQL thread id 34191581, OS thread handle 0x7f25a81cd700, query id 731025165 10.80.48.4 crm_user cleaning up
---TRANSACTION 286365480, not started
MySQL thread id 34191570, OS thread handle 0x7f26051cc700, query id 731025980 10.80.48.4 crm_user cleaning up
---TRANSACTION 286365390, not started
MySQL thread id 34191556, OS thread handle 0x7f25c9ca7700, query id 731025310 10.80.48.5 crm_user cleaning up
---TRANSACTION 286364480, not started
MySQL thread id 34191542, OS thread handle 0x7f25975cd700, query id 731022779 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365490, not started
MySQL thread id 34191511, OS thread handle 0x7f2532109700, query id 731025851 10.80.48.4 crm_user cleaning up
---TRANSACTION 286364987, not started
MySQL thread id 34191512, OS thread handle 0x7f2553d9c700, query id 731024369 10.80.48.4 crm_user cleaning up
---TRANSACTION 286364432, not started
MySQL thread id 34191509, OS thread handle 0x7f25b9076700, query id 731022661 10.80.48.4 crm_user cleaning up
---TRANSACTION 286365641, not started
MySQL thread id 34191510, OS thread handle 0x7f25430d8700, query id 731025918 10.80.48.4 crm_user cleaning up
---TRANSACTION 286364433, not started
MySQL thread id 34191477, OS thread handle 0x7f25974d8700, query id 731025309 10.80.48.5 crm_user cleaning up
---TRANSACTION 286363647, not started
MySQL thread id 34191463, OS thread handle 0x7f2532109700, query id 731024370 10.80.48.4 crm_user cleaning up
---TRANSACTION 286363328, not started
MySQL thread id 34191311, OS thread handle 0x7f2553d09700, query id 731019476 10.80.48.5 crm_user cleaning up
---TRANSACTION 286363327, not started
MySQL thread id 34191289, OS thread handle 0x7f25c9d6b700, query id 731019405 10.80.48.4 crm_user cleaning up
---TRANSACTION 286365642, not started
MySQL thread id 34191420, OS thread handle 0x7f25b91cd700, query id 731026237 10.80.48.5 crm_user cleaning up
---TRANSACTION 286363307, not started
MySQL thread id 34191403, OS thread handle 0x7f25b9109700, query id 731025311 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365615, not started
MySQL thread id 34191413, OS thread handle 0x7f25975fe700, query id 731025837 10.80.48.4 crm_user cleaning up
---TRANSACTION 286363851, not started
MySQL thread id 34191144, OS thread handle 0x7f253219c700, query id 731024245 10.80.48.5 crm_user cleaning up
---TRANSACTION 286363988, not started
MySQL thread id 34191303, OS thread handle 0x7f25c9c76700, query id 731021405 10.80.48.4 crm_user cleaning up
---TRANSACTION 286361975, not started
MySQL thread id 34191351, OS thread handle 0x7f25c9c76700, query id 731025314 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365473, not started
MySQL thread id 34191340, OS thread handle 0x7f2543045700, query id 731025448 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365776, not started
MySQL thread id 34191338, OS thread handle 0x7f25dac76700, query id 731026232 10.80.48.4 crm_user cleaning up
---TRANSACTION 286363306, not started
MySQL thread id 34191315, OS thread handle 0x7f2586909700, query id 731024368 10.80.48.4 crm_user cleaning up
---TRANSACTION 286365785, not started
MySQL thread id 34191304, OS thread handle 0x7f2597509700, query id 731026247 10.80.48.4 crm_user cleaning up
---TRANSACTION 286364556, not started
MySQL thread id 34191322, OS thread handle 0x7f25dac45700, query id 731022998 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365529, not started
MySQL thread id 34191318, OS thread handle 0x7f2553c76700, query id 731026122 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365831, not started
MySQL thread id 34191313, OS thread handle 0x7f2597476700, query id 731026312 10.80.48.5 crm_user cleaning up
---TRANSACTION 286360855, not started
MySQL thread id 34191287, OS thread handle 0x7f2553d09700, query id 731024371 10.80.48.4 crm_user cleaning up
---TRANSACTION 286361176, not started
MySQL thread id 34191267, OS thread handle 0x7f2586909700, query id 731025313 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365682, not started
MySQL thread id 34191245, OS thread handle 0x7f25975fe700, query id 731026062 10.80.48.4 crm_user cleaning up
---TRANSACTION 286365834, not started
MySQL thread id 34191150, OS thread handle 0x7f2553c76700, query id 731026328 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365827, not started
MySQL thread id 34191149, OS thread handle 0x7f25b9076700, query id 731026317 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365618, not started
MySQL thread id 34191141, OS thread handle 0x7f258696b700, query id 731025843 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365645, not started
MySQL thread id 34191117, OS thread handle 0x7f258696b700, query id 731026265 10.80.48.4 crm_user cleaning up
---TRANSACTION 286364526, not started
MySQL thread id 34191148, OS thread handle 0x7f25c9ca7700, query id 731022897 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365481, not started
MySQL thread id 34191147, OS thread handle 0x7f2586909700, query id 731025472 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365815, not started
MySQL thread id 34191119, OS thread handle 0x7f2597445700, query id 731026318 10.80.48.4 crm_user cleaning up
---TRANSACTION 286364567, not started
MySQL thread id 34191081, OS thread handle 0x7f2543109700, query id 731023023 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365646, not started
MySQL thread id 34191076, OS thread handle 0x7f25c9ca7700, query id 731025931 10.80.48.4 crm_user cleaning up
---TRANSACTION 286365729, not started
MySQL thread id 34191077, OS thread handle 0x7f25b9076700, query id 731026170 10.80.48.4 crm_user cleaning up
---TRANSACTION 286364278, not started
MySQL thread id 34191048, OS thread handle 0x7f253219c700, query id 731022168 10.80.48.4 crm_user cleaning up
---TRANSACTION 286365398, not started
MySQL thread id 34191058, OS thread handle 0x7f25dac45700, query id 731025212 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365608, not started
MySQL thread id 34191057, OS thread handle 0x7f25431fe700, query id 731025847 10.80.48.5 crm_user cleaning up
---TRANSACTION 286362432, not started
MySQL thread id 34190936, OS thread handle 0x7f2553d9c700, query id 731024247 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365698, not started
MySQL thread id 34190879, OS thread handle 0x7f2597445700, query id 731026102 10.80.48.4 crm_user cleaning up
---TRANSACTION 286360653, not started
MySQL thread id 34190901, OS thread handle 0x7f25974d8700, query id 731024638 10.80.48.4 crm_user cleaning up
---TRANSACTION 286365824, not started
MySQL thread id 34190881, OS thread handle 0x7f25c9d6b700, query id 731026304 10.80.48.4 crm_user cleaning up
---TRANSACTION 286365649, not started
MySQL thread id 34190826, OS thread handle 0x7f26051cc700, query id 731025938 10.80.48.4 crm_user cleaning up
---TRANSACTION 286363827, not started
MySQL thread id 34190806, OS thread handle 0x7f25a81fe700, query id 731024244 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365399, not started
MySQL thread id 34190802, OS thread handle 0x7f25b916b700, query id 731025214 10.80.48.5 crm_user cleaning up
---TRANSACTION 286364549, not started
MySQL thread id 34190755, OS thread handle 0x7f25b90a7700, query id 731022968 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365675, not started
MySQL thread id 34190421, OS thread handle 0x7f258696b700, query id 731026040 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365598, not started
MySQL thread id 34190508, OS thread handle 0x7f25a816b700, query id 731025844 10.80.48.4 crm_user cleaning up
---TRANSACTION 286365603, not started
MySQL thread id 34190450, OS thread handle 0x7f25dac76700, query id 731025809 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365708, not started
MySQL thread id 34190418, OS thread handle 0x7f25b916b700, query id 731026117 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365391, not started
MySQL thread id 34190397, OS thread handle 0x7f258696b700, query id 731025182 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365537, not started
MySQL thread id 34190252, OS thread handle 0x7f25b9045700, query id 731025991 10.80.48.4 crm_user cleaning up
---TRANSACTION 286365653, not started
MySQL thread id 34190323, OS thread handle 0x7f25b9109700, query id 731026266 10.80.48.4 crm_user cleaning up
---TRANSACTION 286364306, not started
MySQL thread id 34190245, OS thread handle 0x7f25320d8700, query id 731026321 10.80.48.4 crm_user cleaning up
---TRANSACTION 286365757, not started
MySQL thread id 34189735, OS thread handle 0x7f25b922f700, query id 731026201 10.80.48.4 crm_user cleaning up
---TRANSACTION 286365719, not started
MySQL thread id 34189253, OS thread handle 0x7f25b9076700, query id 731026151 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365789, not started
MySQL thread id 34189003, OS thread handle 0x7f25b9109700, query id 731026254 10.80.48.4 crm_user cleaning up
---TRANSACTION 286365546, not started
MySQL thread id 34188727, OS thread handle 0x7f2532109700, query id 731025947 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365717, not started
MySQL thread id 34188554, OS thread handle 0x7f25dac76700, query id 731026149 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365775, not started
MySQL thread id 34187583, OS thread handle 0x7f25c9c76700, query id 731026229 10.80.48.5 crm_user cleaning up
---TRANSACTION 286365835, not started
MySQL thread id 34187359, OS thread handle 0x7f25b922f700, query id 731026329 10.80.48.4 crm_user cleaning up
---TRANSACTION 286364440, not started
MySQL thread id 34184793, OS thread handle 0x7f2543109700, query id 731022688 10.80.48.4 crm_user cleaning up
---TRANSACTION 286320051, not started
MySQL thread id 34156276, OS thread handle 0x7f254322f700, query id 730904620 10.80.16.6 os_user cleaning up
---TRANSACTION 286054504, not started
MySQL thread id 33332904, OS thread handle 0x7f25b919c700, query id 730174758 10.80.16.29 os_user cleaning up
---TRANSACTION 286365402, not started
MySQL thread id 22909707, OS thread handle 0x7f25430a7700, query id 731025237 localhost tdsqlsys_agent cleaning up
---TRANSACTION 286365128, not started
MySQL thread id 22909706, OS thread handle 0x7f2575a2f700, query id 731024530 localhost tdsqlsys_agent cleaning up
---TRANSACTION 286365668, not started
MySQL thread id 22909704, OS thread handle 0x7f25974a7700, query id 731026013 localhost tdsqlsys_agent cleaning up
---TRANSACTION 286365669, not started
MySQL thread id 22909705, OS thread handle 0x7f253213a700, query id 731026014 localhost tdsqlsys_agent cleaning up
---TRANSACTION 286316893, not started
MySQL thread id 9541252, OS thread handle 0x7f21633de700, query id 730896720 10.95.0.9 dba cleaning up
---TRANSACTION 0, not started
MySQL thread id 62868, OS thread handle 0x7f25a816b700, query id 731019926 10.95.0.9 dba cleaning up
---TRANSACTION 107074306, not started
MySQL thread id 1, OS thread handle 0x7f2608185700, query id 0 Waiting for background binlog tasks
---TRANSACTION 286365833, ACTIVE 0 sec, thread declared inside InnoDB 3009
mysql tables in use 2, locked 0
MySQL thread id 34191589, OS thread handle 0x7f25320d8700, query id 731026324 10.80.48.5 crm_user Sending data
select remind0_.ID as ID1_120_, remind0_.CREATE_ID as CREATE_I2_120_, remind0_.CREATE_NAME as CREATE_N3_120_, remind0_.CREATE_TIME as CREATE_T4_120_, remind0_.IS_SPREAD_REMIND as IS_SPREA5_120_, remind0_.OPT_ID as OPT_ID6_120_, remind0_.OPT_TIME as OPT_TIME7_120_, remind0_.OPT_TYPE as OPT_TYPE8_120_, remind0_.OPT_URL as OPT_URL9_120_, remind0_.ORGAN_CODE as ORGAN_C10_120_, remind0_.ORGAN_ID as ORGAN_I11_120_, remind0_.REMARK as REMARK12_120_, remind0_.REMIND_CONTENT as REMIND_13_120_, remind0_.REMIND_SETTING_ID as REMIND_23_120_, remind0_.REMIND_STATUS as REMIND_14_120_, remind0_.START_TIME as
Trx read view will not see trx with id >= 286365834, sees < 286365834
Trx #rec lock waits 0 #table lock waits 0
Trx total rec lock wait time 0 SEC
Trx total table lock wait time 0 SEC
--------
FILE I/O
--------
I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
I/O thread 1 state: waiting for completed aio requests (log thread)
I/O thread 2 state: waiting for completed aio requests (read thread)
I/O thread 3 state: waiting for completed aio requests (read thread)
I/O thread 4 state: waiting for completed aio requests (read thread)
I/O thread 5 state: waiting for completed aio requests (read thread)
I/O thread 6 state: waiting for completed aio requests (read thread)
I/O thread 7 state: waiting for completed aio requests (read thread)
I/O thread 8 state: waiting for completed aio requests (read thread)
I/O thread 9 state: waiting for completed aio requests (read thread)
I/O thread 10 state: waiting for completed aio requests (read thread)
I/O thread 11 state: waiting for completed aio requests (read thread)
I/O thread 12 state: waiting for completed aio requests (read thread)
I/O thread 13 state: waiting for completed aio requests (read thread)
I/O thread 14 state: waiting for completed aio requests (read thread)
I/O thread 15 state: waiting for completed aio requests (read thread)
I/O thread 16 state: waiting for completed aio requests (read thread)
I/O thread 17 state: waiting for completed aio requests (read thread)
I/O thread 18 state: waiting for completed aio requests (write thread)
I/O thread 19 state: waiting for completed aio requests (write thread)
I/O thread 20 state: waiting for completed aio requests (write thread)
I/O thread 21 state: waiting for completed aio requests (write thread)
I/O thread 22 state: waiting for completed aio requests (write thread)
I/O thread 23 state: waiting for completed aio requests (write thread)
I/O thread 24 state: waiting for completed aio requests (write thread)
I/O thread 25 state: waiting for completed aio requests (write thread)
I/O thread 26 state: waiting for completed aio requests (write thread)
I/O thread 27 state: waiting for completed aio requests (write thread)
I/O thread 28 state: waiting for completed aio requests (write thread)
I/O thread 29 state: waiting for completed aio requests (write thread)
I/O thread 30 state: waiting for completed aio requests (write thread)
I/O thread 31 state: waiting for completed aio requests (write thread)
I/O thread 32 state: waiting for completed aio requests (write thread)
I/O thread 33 state: waiting for completed aio requests (write thread)
Pending normal aio reads: 0 [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0] , aio writes: 0 [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0] ,
 ibuf aio reads: 0, log i/o's: 0, sync i/o's: 0
Pending flushes (fsync) log: 0; buffer pool: 0
7340014081 OS file reads, 112989989 OS file writes, 56478419 OS fsyncs
18.07 reads/s, 16384 avg bytes/read, 24.52 writes/s, 12.07 fsyncs/s
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 1002, seg size 1004, 10071129 merges
merged operations:
 insert 11753878, delete mark 694448, delete 59677
discarded operations:
 insert 0, delete mark 0, delete 0
962542.87 hash searches/s, 39274.88 non-hash searches/s
---
LOG
---
Log sequence number 106246324645
Log flushed up to   106246324629
Pages flushed up to 106234677289
Last checkpoint at  106234556811
Max checkpoint age    3474434213
Checkpoint age target 3365858144
Modified age          11647356
Checkpoint age        11767834
0 pending log writes, 0 pending chkp writes
29930706 log i/o's done, 5.52 log i/o's/second
----------------------
BUFFER POOL AND MEMORY
----------------------
Total memory allocated 17301504000; in additional pool allocated 0
Total memory allocated by read views 10280
Internal hash tables (constant factor + variable factor)
    Adaptive hash index 461651072   (295663936 + 165987136)
    Page hash           289928 (buffer pool 0 only)
    Dictionary cache    89073555    (73915984 + 15157571)
    File system         1346288     (812272 + 534016)
    Lock system         41533288    (41504488 + 28800)
    Recovery system     0   (0 + 0)
Dictionary memory allocated 15157571
Buffer pool size        1023936
Buffer pool size, bytes 16776167424
Free buffers            65531
Database pages          948276
Old database pages      348739
Modified db pages       13077
Percent of dirty pages(LRU & free pages): 1.290
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 15983297984, not young 26338471396
13.30 youngs/s, 188.25 non-youngs/s
Pages read 7340008025, created 2816670, written 73947516
18.07 reads/s, 1.59 creates/s, 17.93 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 948276, unzip_LRU len: 0
I/O sum[86464]:cur[4672], unzip sum[0]:cur[0]
----------------------
INDIVIDUAL BUFFER POOL INFO
----------------------
---BUFFER POOL 0
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14814
Old database pages      5448
Modified db pages       237
Percent of dirty pages(LRU & free pages): 1.496
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 281880723, not young 453744688
0.15 youngs/s, 1.56 non-youngs/s
Pages read 120442656, created 43470, written 4639737
0.37 reads/s, 0.00 creates/s, 0.70 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14814, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 1
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14815
Old database pages      5448
Modified db pages       203
Percent of dirty pages(LRU & free pages): 1.282
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 288033171, not young 462540165
0.00 youngs/s, 0.19 non-youngs/s
Pages read 126854505, created 43630, written 693516
0.22 reads/s, 0.00 creates/s, 0.19 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14815, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 2
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1023
Database pages          14818
Old database pages      5450
Modified db pages       208
Percent of dirty pages(LRU & free pages): 1.313
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 262624762, not young 380645714
0.37 youngs/s, 0.19 non-youngs/s
Pages read 115862855, created 43354, written 644415
0.15 reads/s, 0.00 creates/s, 0.56 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14818, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 3
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14818
Old database pages      5449
Modified db pages       326
Percent of dirty pages(LRU & free pages): 2.058
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 265895801, not young 538996027
0.07 youngs/s, 1.22 non-youngs/s
Pages read 128068080, created 43040, written 6577830
0.26 reads/s, 0.00 creates/s, 0.07 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14818, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 4
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14826
Old database pages      5452
Modified db pages       235
Percent of dirty pages(LRU & free pages): 1.483
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 281989343, not young 409741852
0.07 youngs/s, 0.15 non-youngs/s
Pages read 117203921, created 44913, written 6459964
0.19 reads/s, 0.04 creates/s, 0.33 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14826, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 5
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14826
Old database pages      5452
Modified db pages       248
Percent of dirty pages(LRU & free pages): 1.565
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 305868370, not young 473430312
0.41 youngs/s, 4.59 non-youngs/s
Pages read 120875999, created 51220, written 3212204
0.26 reads/s, 0.07 creates/s, 0.30 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14826, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 6
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14829
Old database pages      5453
Modified db pages       222
Percent of dirty pages(LRU & free pages): 1.400
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 257197128, not young 585842343
0.07 youngs/s, 15.52 non-youngs/s
Pages read 123889971, created 51329, written 2028120
0.52 reads/s, 0.00 creates/s, 0.22 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14829, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 7
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1023
Database pages          14825
Old database pages      5453
Modified db pages       202
Percent of dirty pages(LRU & free pages): 1.275
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 274323337, not young 406558868
0.30 youngs/s, 0.48 non-youngs/s
Pages read 122777658, created 50433, written 1869000
0.41 reads/s, 0.00 creates/s, 0.52 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14825, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 8
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14827
Old database pages      5453
Modified db pages       247
Percent of dirty pages(LRU & free pages): 1.558
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 270448684, not young 436633806
0.41 youngs/s, 0.07 non-youngs/s
Pages read 116586705, created 51520, written 2102845
0.11 reads/s, 0.00 creates/s, 0.26 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14827, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 9
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14819
Old database pages      5450
Modified db pages       224
Percent of dirty pages(LRU & free pages): 1.414
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 275286343, not young 461475711
0.19 youngs/s, 5.56 non-youngs/s
Pages read 110276992, created 48776, written 1632668
0.33 reads/s, 0.00 creates/s, 0.70 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14819, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 10
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14827
Old database pages      5453
Modified db pages       291
Percent of dirty pages(LRU & free pages): 1.836
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 260995815, not young 318851129
0.11 youngs/s, 0.63 non-youngs/s
Pages read 95132508, created 53689, written 2364590
0.33 reads/s, 0.00 creates/s, 0.74 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14827, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 11
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14830
Old database pages      5454
Modified db pages       284
Percent of dirty pages(LRU & free pages): 1.791
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 240760004, not young 395550876
0.07 youngs/s, 0.07 non-youngs/s
Pages read 110428138, created 52321, written 2080367
0.07 reads/s, 0.15 creates/s, 0.41 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14830, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 12
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1023
Database pages          14820
Old database pages      5451
Modified db pages       249
Percent of dirty pages(LRU & free pages): 1.572
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 228342727, not young 444827862
0.33 youngs/s, 0.33 non-youngs/s
Pages read 119325993, created 51459, written 2049231
0.30 reads/s, 0.22 creates/s, 0.33 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14820, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 13
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1023
Database pages          14819
Old database pages      5450
Modified db pages       262
Percent of dirty pages(LRU & free pages): 1.654
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 252268293, not young 430129711
0.48 youngs/s, 0.81 non-youngs/s
Pages read 127260741, created 42982, written 712337
0.33 reads/s, 0.00 creates/s, 0.37 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14819, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 14
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14806
Old database pages      5445
Modified db pages       200
Percent of dirty pages(LRU & free pages): 1.263
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 226872589, not young 501857663
0.19 youngs/s, 2.89 non-youngs/s
Pages read 127703930, created 51121, written 1861858
0.56 reads/s, 0.00 creates/s, 0.30 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14806, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 15
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14810
Old database pages      5446
Modified db pages       194
Percent of dirty pages(LRU & free pages): 1.225
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 223481470, not young 416994862
0.26 youngs/s, 0.33 non-youngs/s
Pages read 113782182, created 42273, written 729379
0.37 reads/s, 0.00 creates/s, 0.30 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14810, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 16
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14810
Old database pages      5446
Modified db pages       245
Percent of dirty pages(LRU & free pages): 1.547
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 230831954, not young 334643823
0.07 youngs/s, 0.11 non-youngs/s
Pages read 109502253, created 43093, written 1943852
0.15 reads/s, 0.00 creates/s, 0.22 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14810, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 17
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14808
Old database pages      5446
Modified db pages       211
Percent of dirty pages(LRU & free pages): 1.333
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 221059092, not young 362592353
0.22 youngs/s, 0.19 non-youngs/s
Pages read 103477232, created 42917, written 646304
0.11 reads/s, 0.19 creates/s, 0.52 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14808, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 18
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14806
Old database pages      5445
Modified db pages       264
Percent of dirty pages(LRU & free pages): 1.668
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 276288350, not young 359916596
0.15 youngs/s, 0.00 non-youngs/s
Pages read 97385287, created 42892, written 672820
0.15 reads/s, 0.07 creates/s, 0.15 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14806, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 19
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14810
Old database pages      5446
Modified db pages       177
Percent of dirty pages(LRU & free pages): 1.118
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 242284972, not young 352162073
0.04 youngs/s, 1.19 non-youngs/s
Pages read 107143239, created 41966, written 673911
0.15 reads/s, 0.04 creates/s, 0.11 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14810, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 20
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14817
Old database pages      5449
Modified db pages       137
Percent of dirty pages(LRU & free pages): 0.865
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 274347059, not young 411137292
0.11 youngs/s, 0.07 non-youngs/s
Pages read 113824557, created 42245, written 656044
0.11 reads/s, 0.00 creates/s, 0.37 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14817, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 21
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14816
Old database pages      5449
Modified db pages       206
Percent of dirty pages(LRU & free pages): 1.300
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 239928811, not young 369790970
0.15 youngs/s, 0.37 non-youngs/s
Pages read 118997494, created 42646, written 676565
0.26 reads/s, 0.11 creates/s, 0.22 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14816, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 22
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1025
Database pages          14825
Old database pages      5452
Modified db pages       170
Percent of dirty pages(LRU & free pages): 1.072
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 239928004, not young 404871246
0.37 youngs/s, 0.07 non-youngs/s
Pages read 114127832, created 42898, written 708926
0.15 reads/s, 0.00 creates/s, 0.26 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14825, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 23
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1025
Database pages          14823
Old database pages      5451
Modified db pages       195
Percent of dirty pages(LRU & free pages): 1.230
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 244193336, not young 328717736
0.11 youngs/s, 13.70 non-youngs/s
Pages read 100026292, created 42589, written 571700
0.33 reads/s, 0.00 creates/s, 0.26 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14823, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 24
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1025
Database pages          14815
Old database pages      5448
Modified db pages       159
Percent of dirty pages(LRU & free pages): 1.004
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 220376940, not young 349963522
0.04 youngs/s, 0.07 non-youngs/s
Pages read 107281038, created 42489, written 728253
0.15 reads/s, 0.00 creates/s, 0.33 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14815, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 25
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1023
Database pages          14817
Old database pages      5450
Modified db pages       157
Percent of dirty pages(LRU & free pages): 0.991
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 238583348, not young 422472829
0.26 youngs/s, 0.37 non-youngs/s
Pages read 107075448, created 42491, written 577496
0.33 reads/s, 0.00 creates/s, 0.15 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14817, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 26
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1023
Database pages          14805
Old database pages      5445
Modified db pages       201
Percent of dirty pages(LRU & free pages): 1.270
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 279733460, not young 440508744
0.15 youngs/s, 0.74 non-youngs/s
Pages read 111548801, created 42915, written 679745
0.48 reads/s, 0.00 creates/s, 0.19 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14805, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 27
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14807
Old database pages      5445
Modified db pages       268
Percent of dirty pages(LRU & free pages): 1.693
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 252306945, not young 379827365
0.11 youngs/s, 1.19 non-youngs/s
Pages read 114328852, created 43369, written 625660
0.19 reads/s, 0.00 creates/s, 0.19 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14807, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 28
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14806
Old database pages      5445
Modified db pages       184
Percent of dirty pages(LRU & free pages): 1.162
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 246509786, not young 407828656
0.15 youngs/s, 1.41 non-youngs/s
Pages read 112383009, created 42547, written 693876
0.26 reads/s, 0.00 creates/s, 0.11 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14806, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 29
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14811
Old database pages      5447
Modified db pages       189
Percent of dirty pages(LRU & free pages): 1.193
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 236907983, not young 449214323
0.15 youngs/s, 8.78 non-youngs/s
Pages read 119839783, created 43161, written 670112
0.44 reads/s, 0.00 creates/s, 0.30 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14811, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 30
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14809
Old database pages      5446
Modified db pages       133
Percent of dirty pages(LRU & free pages): 0.840
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 237975002, not young 394186696
0.33 youngs/s, 0.33 non-youngs/s
Pages read 122413605, created 42896, written 630295
0.11 reads/s, 0.00 creates/s, 0.41 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14809, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 31
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14822
Old database pages      5451
Modified db pages       181
Percent of dirty pages(LRU & free pages): 1.142
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 237027270, not young 416313264
0.15 youngs/s, 0.52 non-youngs/s
Pages read 116439212, created 42838, written 674692
0.22 reads/s, 0.00 creates/s, 0.19 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14822, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 32
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14822
Old database pages      5451
Modified db pages       233
Percent of dirty pages(LRU & free pages): 1.470
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 218812504, not young 415101163
1.44 youngs/s, 6.52 non-youngs/s
Pages read 118469103, created 43071, written 671943
0.41 reads/s, 0.22 creates/s, 0.30 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14822, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 33
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14820
Old database pages      5450
Modified db pages       160
Percent of dirty pages(LRU & free pages): 1.010
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 227637479, not young 428533417
0.41 youngs/s, 0.22 non-youngs/s
Pages read 113265855, created 42306, written 606055
0.15 reads/s, 0.00 creates/s, 0.07 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14820, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 34
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14816
Old database pages      5449
Modified db pages       168
Percent of dirty pages(LRU & free pages): 1.061
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 240565913, not young 442213326
0.11 youngs/s, 12.33 non-youngs/s
Pages read 102035644, created 42348, written 725713
0.44 reads/s, 0.00 creates/s, 0.33 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14816, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 35
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14816
Old database pages      5449
Modified db pages       230
Percent of dirty pages(LRU & free pages): 1.452
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 305463237, not young 514347951
0.30 youngs/s, 10.15 non-youngs/s
Pages read 111378629, created 42379, written 653439
0.41 reads/s, 0.00 creates/s, 0.04 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14816, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 36
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14816
Old database pages      5449
Modified db pages       188
Percent of dirty pages(LRU & free pages): 1.187
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 274592844, not young 487499748
0.04 youngs/s, 0.93 non-youngs/s
Pages read 116782897, created 42805, written 677176
0.33 reads/s, 0.00 creates/s, 0.22 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14816, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 37
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14824
Old database pages      5452
Modified db pages       224
Percent of dirty pages(LRU & free pages): 1.413
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 217285815, not young 410096782
0.30 youngs/s, 4.52 non-youngs/s
Pages read 117554797, created 42739, written 612866
0.26 reads/s, 0.04 creates/s, 0.26 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14824, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 38
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14829
Old database pages      5453
Modified db pages       170
Percent of dirty pages(LRU & free pages): 1.072
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 232448664, not young 371285973
0.22 youngs/s, 4.44 non-youngs/s
Pages read 112885265, created 42915, written 638331
0.15 reads/s, 0.19 creates/s, 0.07 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14829, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 39
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14829
Old database pages      5453
Modified db pages       133
Percent of dirty pages(LRU & free pages): 0.839
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 222550300, not young 367735561
0.07 youngs/s, 1.63 non-youngs/s
Pages read 113356423, created 42401, written 623822
0.19 reads/s, 0.00 creates/s, 0.19 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14829, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 40
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14816
Old database pages      5449
Modified db pages       189
Percent of dirty pages(LRU & free pages): 1.193
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 184846937, not young 148973730
0.11 youngs/s, 0.11 non-youngs/s
Pages read 48768239, created 38722, written 663297
0.15 reads/s, 0.00 creates/s, 0.37 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14816, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 41
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14813
Old database pages      5448
Modified db pages       187
Percent of dirty pages(LRU & free pages): 1.181
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 225797187, not young 413040723
0.11 youngs/s, 0.74 non-youngs/s
Pages read 118099207, created 42612, written 694370
0.37 reads/s, 0.00 creates/s, 0.22 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14813, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 42
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14817
Old database pages      5449
Modified db pages       215
Percent of dirty pages(LRU & free pages): 1.357
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 262421082, not young 392772848
0.26 youngs/s, 0.74 non-youngs/s
Pages read 122374705, created 39007, written 738255
0.30 reads/s, 0.04 creates/s, 0.30 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14817, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 43
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14817
Old database pages      5449
Modified db pages       165
Percent of dirty pages(LRU & free pages): 1.042
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 253815848, not young 418208528
0.11 youngs/s, 1.11 non-youngs/s
Pages read 127806190, created 42985, written 668123
0.26 reads/s, 0.00 creates/s, 0.26 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14817, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 44
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14824
Old database pages      5452
Modified db pages       163
Percent of dirty pages(LRU & free pages): 1.028
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 229064547, not young 435430787
0.15 youngs/s, 0.78 non-youngs/s
Pages read 123949429, created 42665, written 550837
0.41 reads/s, 0.00 creates/s, 0.41 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14824, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 45
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14820
Old database pages      5450
Modified db pages       202
Percent of dirty pages(LRU & free pages): 1.275
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 226427619, not young 364481683
0.22 youngs/s, 24.22 non-youngs/s
Pages read 111612333, created 51340, written 2126438
0.56 reads/s, 0.07 creates/s, 0.26 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14820, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 46
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14822
Old database pages      5451
Modified db pages       206
Percent of dirty pages(LRU & free pages): 1.300
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 254322155, not young 427434686
0.11 youngs/s, 5.56 non-youngs/s
Pages read 115189599, created 42610, written 628374
0.11 reads/s, 0.00 creates/s, 0.26 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14822, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 47
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1023
Database pages          14814
Old database pages      5449
Modified db pages       195
Percent of dirty pages(LRU & free pages): 1.231
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 276858415, not young 423326968
0.11 youngs/s, 0.63 non-youngs/s
Pages read 127383470, created 42745, written 687097
0.59 reads/s, 0.00 creates/s, 0.11 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14814, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 48
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14816
Old database pages      5449
Modified db pages       180
Percent of dirty pages(LRU & free pages): 1.136
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 282694904, not young 376816038
0.19 youngs/s, 11.11 non-youngs/s
Pages read 108712117, created 42218, written 618740
0.37 reads/s, 0.00 creates/s, 0.26 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14816, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 49
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14817
Old database pages      5449
Modified db pages       127
Percent of dirty pages(LRU & free pages): 0.802
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 268554226, not young 392387480
0.11 youngs/s, 0.15 non-youngs/s
Pages read 110179086, created 41880, written 623156
0.11 reads/s, 0.00 creates/s, 0.07 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14817, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 50
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14822
Old database pages      5451
Modified db pages       174
Percent of dirty pages(LRU & free pages): 1.098
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 294239953, not young 435314968
0.37 youngs/s, 1.52 non-youngs/s
Pages read 129953477, created 43189, written 630770
0.19 reads/s, 0.00 creates/s, 0.11 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14822, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 51
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14828
Old database pages      5453
Modified db pages       197
Percent of dirty pages(LRU & free pages): 1.243
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 205287225, not young 386403412
0.15 youngs/s, 0.00 non-youngs/s
Pages read 111169053, created 43000, written 661846
0.15 reads/s, 0.00 creates/s, 0.19 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14828, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 52
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14809
Old database pages      5446
Modified db pages       214
Percent of dirty pages(LRU & free pages): 1.352
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 224058178, not young 474484851
0.11 youngs/s, 0.11 non-youngs/s
Pages read 127030978, created 43094, written 639036
0.11 reads/s, 0.11 creates/s, 0.41 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14809, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 53
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14813
Old database pages      5448
Modified db pages       193
Percent of dirty pages(LRU & free pages): 1.219
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 240605271, not young 412328005
0.11 youngs/s, 0.52 non-youngs/s
Pages read 119449647, created 42983, written 662222
0.41 reads/s, 0.00 creates/s, 0.30 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14813, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 54
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14811
Old database pages      5447
Modified db pages       215
Percent of dirty pages(LRU & free pages): 1.358
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 260274915, not young 451196992
0.30 youngs/s, 14.15 non-youngs/s
Pages read 115606820, created 43006, written 710187
0.44 reads/s, 0.00 creates/s, 0.37 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14811, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 55
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14807
Old database pages      5445
Modified db pages       283
Percent of dirty pages(LRU & free pages): 1.788
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 291878839, not young 320537243
0.19 youngs/s, 7.52 non-youngs/s
Pages read 100981128, created 42787, written 679620
0.22 reads/s, 0.00 creates/s, 0.37 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14807, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 56
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14814
Old database pages      5448
Modified db pages       186
Percent of dirty pages(LRU & free pages): 1.174
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 300259374, not young 428741863
0.07 youngs/s, 0.00 non-youngs/s
Pages read 114635805, created 42416, written 647097
0.11 reads/s, 0.00 creates/s, 0.11 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14814, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 57
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14804
Old database pages      5444
Modified db pages       236
Percent of dirty pages(LRU & free pages): 1.491
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 228713349, not young 398148568
0.11 youngs/s, 0.26 non-youngs/s
Pages read 116435130, created 42248, written 616185
0.30 reads/s, 0.04 creates/s, 0.37 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14804, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 58
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14804
Old database pages      5444
Modified db pages       167
Percent of dirty pages(LRU & free pages): 1.055
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 241472093, not young 482921580
0.00 youngs/s, 0.37 non-youngs/s
Pages read 129240053, created 43012, written 661726
0.37 reads/s, 0.00 creates/s, 0.33 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14804, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 59
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14815
Old database pages      5448
Modified db pages       234
Percent of dirty pages(LRU & free pages): 1.477
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 254524430, not young 410711595
0.85 youngs/s, 0.63 non-youngs/s
Pages read 118486360, created 42299, written 666927
0.52 reads/s, 0.00 creates/s, 0.22 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14815, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 60
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1023
Database pages          14815
Old database pages      5449
Modified db pages       159
Percent of dirty pages(LRU & free pages): 1.004
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 220250921, not young 389793880
0.15 youngs/s, 0.48 non-youngs/s
Pages read 111325938, created 42284, written 650765
0.33 reads/s, 0.00 creates/s, 0.11 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14815, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 61
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14814
Old database pages      5448
Modified db pages       190
Percent of dirty pages(LRU & free pages): 1.200
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 237789017, not young 454544024
0.04 youngs/s, 9.96 non-youngs/s
Pages read 119811330, created 42979, written 651312
0.41 reads/s, 0.00 creates/s, 0.26 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14814, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 62
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14816
Old database pages      5449
Modified db pages       140
Percent of dirty pages(LRU & free pages): 0.884
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 229743087, not young 377557236
0.22 youngs/s, 0.37 non-youngs/s
Pages read 115120561, created 43177, written 656787
0.26 reads/s, 0.00 creates/s, 0.22 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14816, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
---BUFFER POOL 63
Buffer pool size        15999
Buffer pool size, bytes 262127616
Free buffers            1024
Database pages          14820
Old database pages      5450
Modified db pages       245
Percent of dirty pages(LRU & free pages): 1.546
Max dirty pages percent: 70.000
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 235490784, not young 384130710
0.22 youngs/s, 2.78 non-youngs/s
Pages read 118691989, created 42966, written 716692
0.33 reads/s, 0.00 creates/s, 0.30 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 14820, unzip_LRU len: 0
I/O sum[1351]:cur[73], unzip sum[0]:cur[0]
--------------
ROW OPERATIONS
--------------
2 queries inside InnoDB, 0 queries in queue
2 read views open inside InnoDB
1 RW transactions active inside InnoDB
0 RO transactions active inside InnoDB
1 out of 1000 descriptors used
---OLDEST VIEW---
Normal read view
Read view low limit trx n:o 286365665
Read view up limit trx id 286365665
Read view low limit trx id 286365665
Read view individually stored trx ids:
-----------------
Main thread process no. 69463, id 139781619037952, state: sleeping
Number of rows inserted 16696938, updated 26228596, deleted 970970, read 965546661710
4.19 inserts/s, 4.22 updates/s, 0.00 deletes/s, 1248149.66 reads/s
Number of system rows inserted 226745, updated 0, deleted 226742, read 226743
0.00 inserts/s, 0.00 updates/s, 0.00 deletes/s, 0.00 reads/s
----------------------------
END OF INNODB MONITOR OUTPUT
============================

```
