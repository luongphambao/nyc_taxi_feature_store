# MLE2
## **Description**: 

+ In this repository, there is a constructed data pipeline featuring distinct flows tailored for batch and streaming data processing. Different services are utilized to meet the specific needs of each flow. Pyspark, PostgreSQL, Flink, Kafka, DBT, and Airflow are prominent among the services employed for these purposes. Moreover, monitoring tools like Prometheus, Grafana, and LogStash are integrated to ensure effective performance tracking.

## Overall data architecture

![](imgs/final1.png)


## Note:
+ **stream_processing** folder: contain pyflink scripts to process streaming data
+ **jars** folder: contain used jars file for data pipeline 
+ **airflow** folder: enviroment to run airflow service
+ **utils** folder: helper funtions
+ **This repo is implemented on nyc taxi data**
![](imgs/gcs.png)
## 1. Installation
+ Tested on Python 3.9.12 (recommended to use a virtual environment such as Conda)
 ```bash
    conda create -n mle python=3.9
    pip install -r requirements.txt
 ```

+ Data: You can dowload and use this dataset in here: https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page. The format data i use in this Project is Parquet file

+ Docker engine
## How to guide 

You can use list command in `Makefile` to run service

For example: Run all service by command

 ```bash
    make run_all
 ```
 ### Moniotring 
 Access at http://localhost:3000/ to for Grafana for tracking resource usage 
  ![](imgs/grafana.png)
  
 ### Airflow


 You can see task in `airflow/dags` in  `data1.py` and `data2.py`

 ```bash
    make airflow_up
 ```

 Accesss at http://localhost:8080/ to for Airflow UI to run dag
 ![](imgs/airflow.png)
 You create connection `postgre_default` 
 ![](imgs/airflow1.png)

 data1: Download data ->Create streamming data -> Transform data

 data2: Create Datawarehous->Insert data-> Check expectations

### 2.3 Batch processing

+Pyspark helps efficiently handle big data, speeding up data reading and writing, and processing much faster as data grows.

+In this problem, we leverage Pyspark to transform and store data into a data warehouse, as well as quickly validate data.
#### How to guide

+ ```python pyspark/batch_processing.py``` 
+ ```python pyspark/spark_insert.py```
+ ```python pyspark/validation.py```
![](imgs/monitoring_architecture.png)
### 2.5. Streaming data source
+ Besides csv files data, there is also a streaming diabetes data
+ A new sample will be stored in a table in PostgreSQL
+ Then Debezium, which is a connector for PostgreSQL, will scan the table to check whenever the database has new data
+ The detected new sample will be push to the defined topic in Kafka
+ Any consumer can get messages from topic for any purposes
#### How to guide
+ ```cd postgresql_utils```
+ ```./run.sh register_connector ../configs/postgresql-cdc.json``` to send PostgreSQL config to Debezium
![](imgs/debezium.png)
+ ```python create_table.py``` to create a new table on PostgreSQL
+ ```python insert_table.py``` to insert data to the table
+ We can access Kafka at port 9021 to check the results
![](imgs/kafka.png)
+ Then click **Topics** bar to get all existing topics on Kafka
![](imgs/kafka2.png)
    + **dunghc.public.diabetes_new** is my created topic
+ Choose **Messages** to observe streaming messages
![](imgs/kafka1.png)

### 2.6. Streaming processing
+ To handle this streaming datasource, Pyflink is a good option to do this task
#### How to guide
+ ```cd stream_processing```
+ ```python datastream_api.py```
    + This script will check necessary key in messages as well as filter redundant keys and merge data for later use
![](imgs/flink2.png)
        + Only information columns are kept
    + Processed samples will be stored back to Kafka in the defined sink
![](imgs/sink.png)
        + **dunghc.public.sink_diabetes** is the defined sink in my case
+ ```python kafka_consumer.py```
    + Messages from sink will be fed into diabetes service to get predictions
    + From then, New data is created
    + Notice, we have to validate predictions from the diabetes model to ensure labels are correct before using that data to train new models.

 
### Deploy data pipeline on Google Compute Engine
### 3.1. Spin up your instance
Create your [service account](https://console.cloud.google.com/), and select [Compute Admin](https://cloud.google.com/compute/docs/access/iam#compute.admin) role (Full control of all Compute Engine resources) for your service account.

Create new key as json type for your service account. Download this json file and save it in `infra/ansible/secrets` directory. Update your `project` and `service_account_file` in `infra/ansible/create_compute_instance.yaml`.

![](gifs/create_svc_acc_out.gif)

Go back to your terminal, please execute the following commands to create the Compute Engine instance:
```bash
cd infra/ansible
ansible-playbook create_compute_instance.yaml
```

![](gifs/create_compute_instance.gif)

Go to Settings, select [Metadata](https://console.cloud.google.com/compute/metadata) and add your SSH key.

Update the IP address of the newly created instance and the SSH key for connecting to the Compute Engine in the inventory file.

![](gifs/ssh_key_out.gif)

+ ```cd ansible```
+ To initialize a compute engine, json key file of service account on google cloud is located at **secrets folder**
+ ```ansible-playbook create_compute_instance.yaml``` to create virtual machine instance using ansible. Configuration of machine was defined in file create_compute_instance.yaml
![](imgs/gcp.png)
    + Virtual machine is ready to run
    + Before moving to next step, subtitute **External IP** of created compute engine to **inventory file** in **ansible folder**
![](imgs/gcp1.png) 
+ ```ansible-playbook -i ../inventory deploy.yml``` to deploy data pipeline on cloud.