# 0 Presentation 

Based on the documentation available at :
- Apache Airflow : https://airflow.apache.org/docs/apache-airflow/stable/index.html
- Astronomer : https://www.astronomer.io/ebooks/dags-definitive-guide/?utm_term=airflow%20learn&utm_content=airflow-tutorial-lg&utm_campaign=airflow-guides-row&utm_source=adwords&utm_medium=ppc&hsa_acc=4274135664&hsa_cam=22503860639&hsa_grp=177600554054&hsa_ad=749926177586&hsa_src=g&hsa_tgt=kwd-1477911351967&hsa_kw=airflow%20learn&hsa_mt=b&hsa_net=adwords&hsa_ver=3&gad_source=1&gad_campaignid=22503860639&gbraid=0AAAAADP7Y9i3JCVAGdxjsxyCd2tNZl6fu&gclid=Cj0KCQjwl5jHBhDHARIsAB0Yqjwhx2vCQoVTdFrlzn2WDcmhXp7pz0AGZDKGvUt6aLNR4dDMXuW4MJIaAi24EALw_wcB

Airflow is an open-source platform to orchestrate and monitor batch-oriented workflows. It can run in any configurations, local laptop to distributed systems.  

# 1 Airflow versus pipelines in Fabric 

It is a workflow as code approach that enables flexible framework. As it is developed in Python, it enables : 
- Version Control & multiple edits/collaboration
- Automated testing
- Extensibility

Apache Airflow is not meant to design event driven workflows/streaming/continuous runs. It needs to has a start and an end. 

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
- scheduled based on times or events 
