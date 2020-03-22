---
title: "Airflow中的路径选择"
subtitle: "DAG path select in Airflow"
layout: post
author: "jessychen"
header-style: text
tags:
  - Airflow
  - CS
---

> Airflow 是 Airbnb 公司开源的任务调度系统, 通过使用 Python 开发 DAG, 非常方便的调度计算任务。

Airflow之前，当多个上下游的任务间存在依赖时，需要我们自己来写一些调度的code，常用的有几种：

* 上游任务完成后置个done标记。需要下游隔段时间检查上游任务有没有完成。
* 上游完成任务后通过某个api来call下游。你说call 失败了？retry大法用起来呀​​。。。

模块太多，依赖太复杂，记不住怎么办？写个文档吧。。。上游改了？？改文档吧。。。下游又变了？？？再改。。。哎呀，上线来不及了，过几天更新吧。。。。那个文档很久没更新了，还是看code吧。。。

而Airflow使用DAG，可以很方便的通过定义**Operator**，及Operator间的关系（**upstream, downstream**）来解决任务之间的各种依赖。同时，它还提供了UI来查看任务依赖的DAG，任务运行状态。简单的说，它可以使我们更多的关注到业务的实现上而不是各种复杂的模块调用及繁琐的文档维护。

当然，作为一条完整的业务流，根据上游的执行结果来决定下游的行为是最常见不过的事了。比如👇的例子：

有个淘气的Sam小盆友，每天按时 **完成作业** 是个大难题。于是，妈妈想了个办法：

![branch-dag](/img/in-post/Airflow/branch.png)

真是个不错的法子👍！这里，`8点前写完`在Airflow里就是个判断用的task，由它来决定整个DAG的下一步操作。那么，怎么实现呢？Airflow提供了**BranchPythonOperator**，用来支持分支, 通过函数返回要执行的分支。

```python
check_hour_op = BranchPythonOperator(
  task_id = "check_hour",
  python_callable = check_hour,
  dag = dag
)
def check_hour(){
  if current_hour > 8:
    return "sleep"
  else:
    return "watch_animation"
}
do_homework_op >> check_hour_op >> [watch_animation_op, sleep_op]
watch_animation_op >> sleep_op
```

Ok👏 简直太简单了！执行一下试试，然后你会发现，当`check_hour`返回`watch_animation`时，`sleep`这个task的状态永远是skip的？？？让我们翻一下官方的解释：

> Note that using tasks with `depends_on_past=True` downstream from `BranchPythonOperator` is logically unsound as `skipped` status will invariably lead to block tasks that depend on their past successes. `skipped` states propagates where all directly upstream tasks are `skipped`.

就是说，`sleep`这个task在`check_hour`选择了`watch_animation`的时候就被skip掉了，即便这里它是`watch_animation`的下游任务，仍然不会被执行😭。原因呢？官方说是downstream设了*depends_on_past=True*。要知道，operator中还有个属性叫*trigger_rule*，默认值是*all_success*。这两个组合在一起就要上游的task状态都为*success*才能继续往下执行。而这里`sleep`的上游任务`watch_animation`状态是*skipped*, 所以`sleep`就没有被执行到了。那么，如果这里我们把`sleep`的*trigger_rule*置成*none_failed*的话，整个过程就能按我们预期的执行下去了:

```python
sleep_op = PythonOperator(
  task_id = "sleep",
  trigger_rule = "none_failed",
  python_callable = "sleep",
  dag = dag
)
```

再试一下，果然可以啦:satisfied:

Airflow现在支持的Trigger Rules:

> ALL_SUCCESS = 'all_success'
> ALL_FAILED = 'all_failed' 
> ALL_DONE = 'all_done' 
> ONE_SUCCESS = 'one_success' 
> ONE_FAILED = 'one_failed' 
> NONE_FAILED = 'none_failed' 
> NONE_SKIPPED = 'none_skipped' 
> DUMMY = 'dummy'

除了上面这个方法，我们也可以利用**DummyOperator**来达到想要的效果。也就是在`watch_animation`平行的分支上加一个`dummy`task，它啥活都不干：

```python
dummp_op = DummyOperator(
	task_id = "dummy",
  dag = dag
)
do_homework_op >> check_hour_op >> [watch_animation_op, dummp_op]
watch_animation_op >> sleep_op
dummp_op >> sleep_op
```

这种方式更简单易懂，代价就是会稍稍增加些代码，喜欢哪个就仁者见仁了:smirk:。