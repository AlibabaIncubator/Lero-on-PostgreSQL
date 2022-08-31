# Lero: A Learning-to-Rank Query Optimizer
Query optimizer is the core part, as well as the most challenging problem, in DBMS. 
We witness that the relative order (or rank) of plans actually matters to optimizer. Learning the rough rank scores is much easier than the unique latency value. To this end, we design Lero, a new learned query optimizer system following the rank-based paradigm.  
And this repository is a naive demo of Lero based on PostgreSQL. In the future, we will release a development framework that facilitates the integration of machine learning algorithms on the database while avoiding direct modifications to the source code, and then formally do the code refactoring of this project on it. 

---
## Setup

This demo is based on a modified version of PostgreSQL 13.1, these are some related installation processes.

```bash
# 1. download the PostgreSQL 13.1  
wget https://ftp.postgresql.org/pub/source/v13.1/postgresql-13.1.tar.bz2
tar -xvf postgresql-13.1.tar.bz2

# 2. apply some modifications on it
cd postgresql-13.1
git apply ../0001-init-lero.patch

# 3. install PostgreSQL
make
make install

# 4. modify the configuration of PostgreSQL in postgresql.conf
listen_addresses = '*'
geqo = off
max_parallel_workers = 0
max_parallel_workers_per_gather = 0
```
---

## Run the Demo
In this demo, we use an independent server to simulate most of the features of Lero.
Lero is mainly composed of two stages:   

1. generate different execution plans according to different policies  
2. use a model to select an optimal execution plan

For the convenience of demonstration, these two parts are put in the server. PostgreSQL completes the corresponding work by communicating with the server.


### Start Server
```bash
python server.py
```
The port and host of the server are configured in the server.conf. 
PostgreSQL uses the same preset port and host to communicate with the server.
If you want to modify these two configurations, you need to execute two additional commands in PSQL every time you execute a query.
```bash
SET lero_server_host TO "new_host";
SET lero_server_port TO new_port;
```


### Collect Plans and Re-train Model
Here we use TPC-H queries in 1GB data set for demonstration (You can refer to this work ([tpch-dbgen](https://github.com/electrum/tpch-dbgen)) to create corresponding data set).  
Relevant scripts are placed in "./test_script". 
**And please modify the configurations in "config.py" before execution to ensure the script can correctly connect to the server and control the database.**

Use the following command to start demo. 
This script will load the training queries and test queries, and the model will be retrained every 'query_num_per_chunk' queries. Lero will collect performance on the test queries after each training phase.

```bash
# output_query_latency_file: the final executed plan will be output to this file
# model_prefix: prefix of model name
# topK: the number of plans that can be explored by each query
python train_model.py --query_path tpch_train.txt --test_query_path tpch_test.txt --algo lero --query_num_per_chunk 20 --output_query_latency_file lero_tpch.log --model_prefix tpch_test_model --topK 3
```
Four kinds of files will be generated gradually during the execution of this script:
1. lero_tpch.log  
    The best plan considered by model in Lero will be executed and the results will be output to this file.
2. lero_tpch.log_exploratory  
    Other plans for pairwise training of each query will be executed and output to this file.
3. lero_tpch.log.training  
    Integrate the results of "lero_tpch.log" and "lero_tpch.log_exploratory" for model training.
4. lero_tpch.log_tpch_test_model_i  
    The performance of the model after i-th training.

In order to compare the results, after Lero executes all the queries, we use PostgreSQL to execute them again.
The plans of the training set and the test set will be saved in "pg_tpch_train.log" and "pg_tpch_test.log" respectively.
```bash
python train_model.py --query_path tpch_train.txt  --algo pg --output_query_latency_file pg_tpch_train.log
python train_model.py --query_path tpch_test.txt --algo pg --output_query_latency_file pg_tpch_test.log
```

### Result Visualization
Here we can use jupyter to easily visualize the results of the experiment (see visualization.ipynb for details).  
In the training set, Lero will gradually surpass PostgreSQL after the first training.
And in the test set, Lero runs 1.2x faster than PostgreSQL.
<center class="half">
    <img src="lero/test_script/train.jpg" width="200"/>
    <img src="lero/test_script/test.jpg" width="200"/>
</center>


<!-- 
---
## Paper Citation
```bash
TODO
```
-->

---
## Re-produce
We also put some files here to help you reproduce the result of some experiments in paper. You can fine them in "lero/reproduce".  

1. Create database  
We dump the data of STATS to facilitate you to rebuild the database. (You can rebuild JOB by [join-order-benchmark](https://github.com/gregrahn/join-order-benchmark))
```bash
psql > CREATE DATABASE stats;
psql -d stats -f stats_db.sql
```

2. Load Model  
There are three trained models (imdb_pw, stats_pw and tpch_pw) available in the folder. You can modify the configuration file "server.conf" to load the corresponding model. We choose to load "stats_pw" here.
```bash
# in server.conf
ModelPath = ./reproduce/stats_pw
```

3. Test the effect of the model  
As in the previous example, we first need to start the server and set the corresponding script configuration.
And then execute the script to collect the results of the model on the test set.
```bash
# start server
python server.py
# in conf.py
...
DB = stats
...
# execute the script
python test.py --query_path ../reproduce/test/stats.txt --output_query_latency_file stats.test
```


---
Thanks for the implementation of ["Tree Convolution"](https://github.com/RyanMarcus/TreeConvolution).