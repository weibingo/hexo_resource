---
title: airflow使用指南
date: 2018-11-28 23:31:06
tags: airflow
categories: 数据工程
---

airflow 是一个编排、调度和监控workflow的平台，由Airbnb开源。airflow 将workflow编排为tasks组成的DAGs，调度器在一组workers上按照指定的依赖关系执行tasks。同时，airflow 提供了丰富的命令行工具和简单易用的用户界面以便用户查看和操作，并且airflow提供了监控和报警系统。
### airflow 核心概念
* DAGs：即有向无环图(Directed Acyclic Graph)，将所有需要运行的tasks按照依赖关系组织起来，描述的是所有tasks执行的顺序。
* Operators：可以简单理解为一个class，描述了DAG中一个具体的task具体要做的事。其中，airflow内置了很多operators，如BashOperator 执行一个bash 命令，PythonOperator 调用任意的Python 函数，EmailOperator 用于发送邮件，HTTPOperator 用于发送HTTP请求， SqlOperator 用于执行SQL命令...同时，用户可以自定义Operator，这给用户提供了极大的便利性。
* Tasks：Task 是 Operator的一个实例，也就是DAGs中的一个node。
* Task Instance：task的一次运行。task instance 有自己的状态，包括"running", "success", "failed", "skipped", "up for retry"等。
* Task Relationships：DAGs中的不同Tasks之间可以有依赖关系，如 TaskA >> TaskB，表明TaskB依赖于TaskA。

通过将DAGs和Operators结合起来，用户就可以创建各种复杂的 workflow了。

### operators
下面讲解下常见的operator，以及如何使用，注意点。
<!--more-->
#### BashOperator
```python
run_this = BashOperator(
    task_id='run_after_loop', 
    bash_command='echo 1', 
    dag=dag)
```
很简单的一个operator，执行bash命令。其中bash命令返回状态码0时，表示此task成功；否则，失败。

未完待续~~~
