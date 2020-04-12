---
title: "Airflow中利用Variable生成动态DAG"
subtitle: "Using variable to generate dynamic DAG in airflow"
layout: post
author: "jessychen"
header-style: text
tags:
  - Airflow
  - CS
---

> In Airflow, a **DAG** – or a Directed Acyclic Graph – is a collection of all the tasks you want to run, organized in a way that reflects their relationships and dependencies.
>
> **Variables** are a generic way to store and retrieve arbitrary content or settings as a simple key value store within Airflow. 

**DAG**作为Airflow中至关重要的概念，相信你一定不会陌生。而**Variables**，则是Airflow在DB中以KV形式存储的一些变量，方便DAG在执行过程中使用。可能有人会觉的**Variables**和**Xcom**很相似，实际上确实也是。它们的区别在于：XCom设计的初衷是用于DAG内部task之间的数据共享（当然会有一些方法可以获取到其他DAG的XCom，但不推荐这么做），而Variables则是跨DAG的存在。关于XCom的介绍可以看上篇文章[Airflow中利用xcom实现task间信息传递](/2020/04/06/xcom)。更多Airflow的基础概念，参考官方文档[Airflow Concepts](https://airflow.apache.org/docs/stable/concepts.html)。

下面进入主题，今天向大家介绍一种利用Variable生成动态DAG的方法。为什么要动态生成DAG呢？在实际项目中，我们常常会遇到下面几种情况：

1. 需要生成多个类似的DAG。可能它们之间只是schedule时间，或某些配置不同，因而我们不想为每个DAG重复写类似的python文件。
2. 某个DAG，内部task的数量或类型会随外部条件变化。

那么对于上面两种情况，有没有可行的实现方案呢？既然有这篇文章，答案自然是肯定的。这里给出一种解决手段：利用Variables来存储DAG生成需要的参数，从而实现DAG的动态创建。

----

### 生成多个类似DAG

> **场景**： 新时代以HARD模式诞生的Sam同学，每天的学习安排的满满当当的：
>
> **周一到周五**：（7:00开始）
>
> <div class="mermaid">
> graph LR
> A("Go(School)")-->B("Go(InterestClass)");
> B-->C(DoHomework);
> </div>
> **周六到周日**：（8:00开始）
>
> <div class="mermaid">
> graph LR
> A("Go(Park)")-->B("Go(InterestClass)");
> B-->C(DoHomework);
> </div>

根据上面场景，如果想为每天的学习安排生成一个DAG，可以怎么做呢？最简单的方法，当然是写七个python文件来实现。不过这样会有大量重复代码，肯定是不推荐的，你也不会那么做。

第二种方法，是为周一到周五和周六日分别以循环的方式构建出DAG：

```python
for workday in range(1,6):
    dag_id = "weekday_%d" % workday
    dag = DAG(
        dag_id = dag_id,
        default_args = args,
        schedule_interval = '0 7 * * %d' % workday
    )
    globals()[dag_id] = dag
    
    go_school_op = PythonOperator(
        task_id='go_school',
        python_callable=go_study,
        op_kwargs={'target':"school"},
        dag=dag,
    )
    #Simplify Other Operators generate
    go_school_op >> go_interest_op >> do_homework_op

for holiday in [6,0]:
    dag_id = "weekday_%d" % holiday
    dag = DAG(
        dag_id = dag_id,
        default_args = args,
        schedule_interval = '0 8 * * %d' % holiday
    )
    globals()[dag_id] = dag
    
    go_park_op = PythonOperator(
        task_id='go_park',
        python_callable=go_study,
        op_kwargs={'target':"park"},
        dag=dag,
    )
    #Simplify Other Operators generate
    go_park_op >> go_interest_op >> do_homework_op

```

但是，考虑到周中和周末的安排其实是非常相似的，那么我们可不可以把它们进一步合并呢？看下面：

```python
for weekday in range(0,7):
    start = 8 if weekday in [0,6] else 7
    dag_id = "weekday_%d" % weekday
    dag = DAG(
        dag_id = dag_id,
        default_args = args,
        schedule_interval = '0 %d * * %d' % (start, weekday)
    )
    globals()[dag_id] = dag
    
    target = "park" if weekday in [0,6] else "school"
    go_study_op = PythonOperator(
        task_id='go_%s' % target,
        python_callable=go_study,
        op_kwargs={'target':target},
        dag=dag,
    )
    #Simplify Other Operators generate
    go_study_op >> go_interest_op >> do_homework_op
```

再进一步看，如果到了初三或高三，可能学校会要求周末也去学校补课，或者调整到校时间什么的。那样我们就需要去改code了。这时候可以用上Variable的配置功能，只用在UI上改下配置就达到调整的目的：

```json
// use 'weekday_schedule' as Variable key, value is a json:
{
  "0":{"start":8, "target":"park"},
  "1":{"start":7, "target":"school"},
  "2":{"start":7, "target":"school"},
  "3":{"start":7, "target":"school"},
  "4":{"start":7, "target":"school"},
  "5":{"start":7, "target":"school"},
  "6":{"start":8, "target":"park"}
}
```

修改后的DAG最终如下：

```python
weekday_schedule = Variable.get("weekday_schedule", deserialize_json=True)

for weekday in weekday_schedule:
    schedule = weekday_schedule[weekday]
    dag_id = "weekday_%s" % weekday
    dag = DAG(
        dag_id = dag_id,
        default_args = args,
        schedule_interval = '0 %d * * %s' % (schedule["start"], weekday)
    )
    globals()[dag_id] = dag
    
    go_study_op = PythonOperator(
        task_id='go_%s' % schedule["target"],
        python_callable=go_for_study,
        op_kwargs={'target':schedule["target"]},
        dag=dag,
    )
    #Simplify Other Operators generate
    go_study_op >> go_interest_op >> do_homework_op
```



----

### 动态控制DAG内tasks

> **场景**：想更细化一下Sam每天上课的科目和数量，每天根据课程表为每节课构建一个Task，把上第一个Go(***) task 替换成具体的课程，类似：
>
> **周一**：（7:00开始）
>
> <div class="mermaid">
> graph LR
> A(Chinese)-->B(Art);
> B-->C(English);
> C-->D(PEclass);
> </div>
>
> **周二**：（7:00开始）
>
> <div class="mermaid">
> graph LR
> A(Math)-->B(PEclass);
> B-->C(Chinese);
> C-->D(Art);
> </div>
>
> **周六**：（8:00开始）
>
> <div class="mermaid">
> graph LR
> A(PEclass)-->B(Math);
> B-->C(English);
> </div>

同样，我们需要为每天的课程构建一个DAG。这些DAG的某些组成来自一群相同的tasks（科目），但每个DAG中的tasks实例及数目又各不相同。更恼火的是，这些tasks的组合（课程表）可能时不时（每学期）会变化。如果直接固定的写在Code里，绝对会是场灾难。所以，更好的方法是把这种组合关系写到配置（Variable）里，方便随时更新。

Variable里的内容：

```json
// use 'weekday_schedule' as Variable key, value is a json:
{
  "0":{"start":8, "target":"park", "classes":["PEclass", "Chinese", "Art"]},
  "1":{"start":7, "target":"school", "classes":["Chinese", "Art", "English", "PEclass"]},
  "2":{"start":7, "target":"school", "classes":["Math", "PEclass", "Chinese", "Art"]},
  ......
  "6":{"start":8, "target":"park", "classes":["PEclass", "Math", "English"]}
}

```

DAG实现Code：

```python
weekday_schedule = Variable.get("weekday_schedule", deserialize_json=True)

for weekday in weekday_schedule:
    schedule = weekday_schedule[weekday]
    dag_id = "weekday_%s" % weekday
    dag = DAG(
        dag_id = dag_id,
        default_args = args,
        schedule_interval = '0 %d * * %s' % (schedule["start"], weekday)
    )
    globals()[dag_id] = dag
    
    classes = schedule["classes"]
    last_study_op = None
    for c in classes:
        go_study_op = PythonOperator(
            task_id='%s_class_at_%s' % (c, schedule["target"]),
            python_callable=go_study,
            op_kwargs={'target':schedule["target"], 'class_type':c},
            dag=dag,
        )
        if last_study_op:
            last_study_op >> go_study_op
        last_study_op = go_study_op

    #Simplify Other Operators generate
    last_study_op >> go_interest_op >> do_homework_op
```

通过这种写法，不管上课的时间、地点、内容怎样变化，只要一份code就可以很方便的按需生成新的DAG。不过，这么写的话Variable的配置会稍稍复杂了点。如果觉的手动更新Variable容易出错的话，建议写成脚本，在脚本中拼装json数据，再调用Airflow Cli来实现更新。



