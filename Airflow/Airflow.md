# 0 Presentation 

Based on the documentation available at :
- [Apache Airflow](https://airflow.apache.org/docs/apache-airflow/stable/index.html)
- [Astronomer](https://www.astronomer.io/ebooks/dags-definitive-guide/?utm_term=airflow%20learn&utm_content=airflow-tutorial-lg&utm_campaign=airflow-guides-row&utm_source=adwords&utm_medium=ppc&hsa_acc=4274135664&hsa_cam=22503860639&hsa_grp=177600554054&hsa_ad=749926177586&hsa_src=g&hsa_tgt=kwd-1477911351967&hsa_kw=airflow%20learn&hsa_mt=b&hsa_net=adwords&hsa_ver=3&gad_source=1&gad_campaignid=22503860639&gbraid=0AAAAADP7Y9i3JCVAGdxjsxyCd2tNZl6fu&gclid=Cj0KCQjwl5jHBhDHARIsAB0Yqjwhx2vCQoVTdFrlzn2WDcmhXp7pz0AGZDKGvUt6aLNR4dDMXuW4MJIaAi24EALw_wcB)

Airflow is an open-source platform to orchestrate and monitor batch-oriented workflows. It can run in any configurations, local laptop to distributed systems. Based on Python Code, it can execute tasks based on different components, like Python, Bash, Docker images in Kubernates Pods ... 

# 1 Airflow versus pipelines in Fabric 

It is a workflow as code approach that enables flexible framework. As it is developed in Python, it enables : 
- Version Control & multiple edits/collaboration
- Automated testing
- Extensibility

**Warning :** Apache Airflow is not meant to design event driven workflows/streaming/continuous runs. It needs to has a start and an end.

# 2 Introduction to DAGs 

DAG = Directed Acyclic Graph. It is an individual **pipeline + schedule**, written in **Python**. When launched, it is a **DAG Run**. 
DAGs are made of **tasks**. When task is executed at a specific time point it is a **task instance**. 
DAGs can have parallel/sequential tasks + conditionnal branches. 

Two requirements in a DAG structure : 
- No clear direction between tasks (directed)
- No circular dependencies (acyclic)

Tasks can be simple Python functions, complex data transformations, or external data service calls. 

Composition of DAGs : 
- an operator : a python class
- dependencies between items
- scheduled items

Triggers : 
Scheduled/Manual/Dataset triggered/Backfill

Statuses : 
Queued/Running/Success/Failed 

# 3 Compositions of DAGs 

## Operator :
Contains the logic of how the data is processed in the pipeline. It is a Python class that encapsulte the logic to do work units, like a wrapper. When an operator is instanciated, it becomes a Task. It can be generic of very specific.
  - All operators inherit from the abstract class ```BaseOperator``` that contains logic to execute work. 
  - PythonOperator is most common operator, and can execute Python Function.
  - Some operators are included in the core Airflow package. Some others, specific, may need to be installed separately.
  - If an operator already exists, prefer using it instead of using own Python one, to ensure maintenability & readability.
  - Any operator that inteacts with exernal services require authentication to external services. 

```
 # from airflow.operators.python import PythonOperator
 def _say_hello():
 return “hello”
 say_hello = PythonOperator(
 task_id=”say_hello”,
 )
```
  
  - BashOperator can execute bash command or script. 

```
# from airflow.operators.bash import BashOperator
 bash_task = BashOperator(
 task_id=”bash_task”,
 bash_command=”echo $MY_VAR”,
 env={“MY_VAR”: “Hello World”}
 )
```
 
## Decorators 

In Python, decorator is a function that takes another one as argument to extend it. 
Decorators are Part of TaskFlowAPI : an API to easily define DAGS & tasks, that simplifies how to pass data from one task to the other. 

```
 # definition of the decorator function
 def multiply_by_100_decorator(decorated_function):
 def wrapper(num1, num2):
 result = decorated_function(num1, num2) * 100
 return result
 return wrapper
 # definition of the `add` function decorated with the `multiply_by_100_decorator`
 @multiply_by_100_decorator
 def add(num1, num2):
 return num1 + num2
 @multiply_by_100_decorator
 def subtract(num1, num2):
 return num1 - num2
 # definition of the `subtract` function decorated with the `multiply_by_100_decorator`
 # calling the decorated functions
 print(add(1, 9))  # prints 1000
 print(subtract(4, 2))  # prints 200
```

# TBD

_TaskFlow/TaskFlowAPI_
_Sensors_
_Deferables operators_

_Deferrable Operators are a type of operator that releases their worker slot while waiting for 
their work to be completed. This can result in cost savings and greater scalability. Astron
omer recommends using deferrable operators whenever one exists for your use case and 
your task takes longer than a minute. You must have a triggerer running to use deferrable 
operators._
