# 0️⃣ Presentation 

Based on the documentation available at :
- [Apache Airflow](https://airflow.apache.org/docs/apache-airflow/stable/index.html)
- [Astronomer](https://www.astronomer.io/ebooks/dags-definitive-guide/?utm_term=airflow%20learn&utm_content=airflow-tutorial-lg&utm_campaign=airflow-guides-row&utm_source=adwords&utm_medium=ppc&hsa_acc=4274135664&hsa_cam=22503860639&hsa_grp=177600554054&hsa_ad=749926177586&hsa_src=g&hsa_tgt=kwd-1477911351967&hsa_kw=airflow%20learn&hsa_mt=b&hsa_net=adwords&hsa_ver=3&gad_source=1&gad_campaignid=22503860639&gbraid=0AAAAADP7Y9i3JCVAGdxjsxyCd2tNZl6fu&gclid=Cj0KCQjwl5jHBhDHARIsAB0Yqjwhx2vCQoVTdFrlzn2WDcmhXp7pz0AGZDKGvUt6aLNR4dDMXuW4MJIaAi24EALw_wcB)

Airflow is an open-source platform to orchestrate and monitor batch-oriented workflows. It can run in any configurations, local laptop to distributed systems. Based on Python Code, it can execute tasks based on different components, like Python, Bash, Docker images in Kubernates Pods ... 

API = Application Programming Interface. It is a developped interface as a service, to connect computers/programs together, in opposition with user interfaces. It is not necessarly a web API (such as REST API), that means that the API is available via Internet (privately or publically). 

# :one: Airflow versus pipelines in Fabric 

It is a workflow as code approach that enables flexible framework. As it is developed in Python, it enables : 
- Version Control & multiple edits/collaboration
- Automated testing
- Extensibility

**Warning :** Apache Airflow is not meant to design event driven workflows/streaming/continuous runs. It needs to has a start and an end.

# :two: Introduction to DAGs 

DAG = Directed Acyclic Graph. It is an individual **pipeline + schedule**, written in **Python**. When launched, it is a **DAG Run**. 
DAGs are made of **tasks**. When task is executed at a specific time point it is a **task instance**. 
DAGs can have parallel/sequential tasks + conditionnal branches. 

Two requirements in a DAG structure : 
- No clear direction between tasks (directed)
- No circular dependencies (acyclic)
e
Tasks can be simple Python functions, complex data transformations, or external data service calls. 

Composition of DAGs : 
- an operator : a python class
- dependencies between items
- scheduled items

Triggers : 
Scheduled/Manual/Dataset triggered/Backfill

Statuses : 
Queued/Running/Success/Failed 

# :three: Compositions of DAGs 

DAGs are made of different tasks. These tasks can be of different kinds, such as : 
- **Operators** : Items that holds the logic of data processing. 
- **Deferrable operators** : Asyncrhonous operators, to reorganize tasks waiting for external resources. 
- **Sensors** : Items that waits for signals to organize pipelines.
- **Hooks** : Abstraction of an external API to enable Airflow interactions. 

## Operator :
Contains the logic of how the data is processed in the pipeline. It is a Python class that encapsulte the logic to do work units, like a wrapper. When an operator is instanciated, it becomes a Task. It can be generic of very specific.
  - All operators inherit from the abstract class ```BaseOperator``` that contains logic to execute work. 
  - PythonOperator is most common operator, and can execute Python Function.
  - Some operators are included in the core Airflow package. Some others, specific, may need to be installed separately.
  - If an operator already exists, prefer using it instead of using own Python one, to ensure maintenability & readability.
  - Any operator that inteacts with exernal services require authentication to external services. 

```python
 # from airflow.operators.python import PythonOperator
 def _say_hello():
 return “hello”
 say_hello = PythonOperator(
 task_id=”say_hello”,
 )
```
  
  - BashOperator can execute bash command or script. 

```python
# from airflow.operators.bash import BashOperator
 bash_task = BashOperator(
 task_id=”bash_task”,
 bash_command=”echo $MY_VAR”,
 env={“MY_VAR”: “Hello World”}
 )
```
 
## Decorators 

In Python, decorator is a function that takes another one as argument to extend it. For instance : ```multiply_by_100_decorator``` is the decorator. ```@multiply_by_100_decorator``` enables to use it. 

```python
 # definition of the decorator function
 def multiply_by_100_decorator(decorated_function):
   def wrapper(num1, num2):
     result = decorated_function(num1, num2) * 100
     return result
   return wrapper

# definition of the add and substract function decorated with the `multiply_by_100_decorator`
@multiply_by_100_decorator
  def add(num1, num2):
     return num1 + num2

@multiply_by_100_decorator
  def subtract(num1, num2):
    return num1 - num2

# calling the decorated functions
print(add(1, 9))  # prints 1000
print(subtract(4, 2))  # prints 200
```

Decorators are Part of TaskFlowAPI : an API to easily define DAGS & tasks, that simplifies how to pass data from one task to the other. Otherwise, every function needs to come with it's corresponding Operator, and set dependencies explicitly.  

 ```python
# from airflow.decorators import task
 @task
 def say_hello():
   return “hello“
```

 replaces 

 ```python
# from airflow.operators.python import PythonOperator
def _say_hello():
  return “hello”

say_hello = PythonOperator(
  task_id=”say_hello”,
   python_callable=_say_hello
)
```

**Common decorators** :

- DAG decorator (@dag()), which creates a DAG.
- TaskGroup decorator (@task_group()), which creates a task group.
- Task decorator (@task()), which creates a Python task.
- Bash decorator (@task.bash()) which creates a BashOperator task.
- Python Virtual Env decorator (@task.virtualenv()), which runs your Python task in a virtual environment.
- Branch decorator (@task.branch()), which creates a branch in your DAG based on an evaluated condition.
- Kubernetes pod decorator (@task.kubernetes()), which runs a KubernetesPodOperator task.
- Sensor decorator (@task.sensor()), which turns a Python function into a sensor.

## Sensors 

Sensors are operators that are waiting for something. It checks if a condition is met and then is in success. Sensor takes following parameters : 
- mode :
  - Poke (default mode) : good for short run time. Occupies a worker during all execution and sleeps betwen checks.
  - Reschedule : good for long run time. if criteria is not met, the sensor releases its worker slot and reschedule for the next check.  
- poke_interval : default is 60 sec. It defines the wait interval between checks. 
- exponential_backoff : (boolean) extends exponentially the wait time between pokes. 
- timeout : (in sec.) max time to check condition before the task fails. 
- soft_fail : (boolean) if not met it is skipped once timed out.  

Failure modes : 
- soft_fail : (boolean) If an exception is raised, task is marked as skipped instead of fail. 
- silent_fail : (boolean) Log only but no error if the error is not a common mikstake. 
- never_fail : (boolean) 

**Common sensors** :

- S3KeySensor: Waits for a key (file) to appear in an Amazon S3 bucket. This sensor is useful if you want your DAG to process files from Amazon S3 as they arrive.
- HttpSensor: Waits for an API to be available. This sensor is useful if you want to ensure your API requests are successful.
- SqlSensor: Waits for data to be present in a SQL table. This sensor is useful if you want your DAG to process data as it arrives in your database.
- @task.sensor.decorator: Allows you to turn any Python function that returns a PokeReturn Value into an instance of the BaseSensorOperator class. This way of creating a sensor is useful when checking for complex logic or if you are connecting to a tool via an API that has no specific sensor available.

Sensor decorator enables to create your own sensor. It returns PokeReturnValue. 

**Best Practices** : 

- Define timeouts that meets integration logic. Default is 7 days and in most case it is longer than integration timeframe. Do not check state every 60 seconds if the average runtime is 30 minutes. 
- For long running sensors, the risk is that it never releases a worker, and provoke dead locks. Prefer deferrable operators, and if not available, prefer sensors in reschedule mode.
- If the poke_interval is shorter than 5 minutes, prefer poke_mode. 

## Deferrable operators (asyncrhonous operators) : 

Deferrable operators are operators that runs asyncrhonously, usign the asyncio library from Python. It allows releasing workers to manager resources efficiently.  

It is made of : 
- Trigger : asynchronous python code sections living in a single process, called a Triggerer
- Triggerer : rescheduler or worker that holds a asyncio event loop.
- Deferred state : a tasks indicates that it is deferred when it paused it's execution, relased a worker slot, and submited a trigger to be catched by the triggerer.   

Operators are in three steps : 
- Submit & prepare a job to a service ==> Wait for the execution of the job ==> Receive the status of the job and continue.  

In the case of a classic operator, the wait time would occupy a worker slot even if action is processed outside of the airflow run. When using the deferrable operators, a slot would be released during the wait time, and a triggerer would wait for the completion. Triggerer can hold multiple asyncrhonous polling tasks, preventing individual workers for each polling tasks.   

Everytime you are using an external system, deferrable operators should be prefered, providing efficency and reducing costs. Operators can support deferrable mode or not.  

## Hooks : 

Hooks are abstractions of specific APIs allowing interactions. Standardization of API usage via Hooks makes code cleaner and improves readability. 

## Connect to resources in Airflow : 

Airflow connection are configuration to access to external tools. It is shared to modules via Connection ID to be used. 

# TBD

_XCom_

_Deferrable Operators are a type of operator that releases their worker slot while waiting for 
their work to be completed. This can result in cost savings and greater scalability. Astron
omer recommends using deferrable operators whenever one exists for your use case and 
your task takes longer than a minute. You must have a triggerer running to use deferrable 
operators._
