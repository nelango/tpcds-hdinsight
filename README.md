# tpcds-hdinsight

Goal of this project is to help generate TPCDS data with hive and create your own HDInsight benchmarks for various engines 

1. Hive
2. Interactive Hive(LLAP)
3. Spark
4. Presto


## How to use with Hive CLI

1. Clone this repo.

    ```shell
    git clone https://github.com/nelango/tpcds-hdinsight && cd tpcds-hdinsight
    ```
2. Run TPCDSDataGen.hql with settings.hql file and set the required config variables.
    ```shell
    /usr/bin/hive -i settings.hql -f TPCDSDataGen.hql -hiveconf SCALE=10 -hiveconf PARTS=10 -hiveconf LOCATION=/HiveTPCDS/ -hiveconf TPCHBIN=resources 
    ```
    Here, 
    
    `SCALE` is a scale factor for TPCDS. Scale factor 10 roughly generates 10 GB data, Scale factor 1000 generates 1 TB of data and so on.
    
    `PARTS` is a number of task to use for datagen (parrellelization). This should be set to the same value as `SCALE`. 
    
    `LOCATION` is the directory where the data will be stored on HDFS. 
    
    `TPCHBIN` is where the resources are found. You can specify specific settings in settings.hql file.

3. Now you can create tables on the generated data.
    ```shell
    /usr/bin/hive -i settings.hql -f ddl/createAllExternalTables.hql -hiveconf LOCATION=/HiveTPCDS/ -hiveconf DBNAME=tpcds
    ```
    Generate ORC tables and analyze
    ```shell
    hive -i settings.hql -f ddl/createAllORCTables.hql -hiveconf ORCDBNAME=tpcds_orc -hiveconf SOURCE=tpcds
    hive -i settings.hql -f ddl/analyze.hql -hiveconf ORCDBNAME=tpcds_orc 
    ```

4. Run the queries !
    ```shell
    /usr/bin/hive -database tpcds_orc -i settings.hql -f queries/query12.sql 
    ```

## How to use with Beeline CLI

1. Clone this repo.

    ```shell
    git clone https://github.com/nelango/tpcds-hdinsight && cd tpcds-hdinsight
    ```

2. Upload the resources to DFS.
    ```shell
    hdfs dfs -copyFromLocal resources /tmp
    ```    
    
3. Run TPCDSDataGen.hql with settings.hql file and set the required config variables.
    ```shell
    beeline -u "jdbc:hive2://`hostname -f`:10001/;transportMode=http" -n "" -p "" -i settings.hql -f TPCDSDataGen.hql -hiveconf SCALE=10 -hiveconf PARTS=10 -hiveconf LOCATION=/HiveTPCDS/ -hiveconf TPCHBIN=`grep -A 1 "fs.defaultFS" /etc/hadoop/conf/core-site.xml | grep -o "wasb[^<]*"`/tmp/resources  
    ```
       Here, 
    
    `SCALE` is a scale factor for TPCDS. Scale factor 10 roughly generates 10 GB data, Scale factor 1000 generates 1 TB of data and so on.
    
    `PARTS` is a number of task to use for datagen (parrellelization). This should be set to the same value as `SCALE`. 
    
    `LOCATION` is the directory where the data will be stored on HDFS. 
    
    `TPCHBIN` is where the resources are found. You can specify specific settings in settings.hql file.

4. Now you can create tables on the generated data.
    ```shell
    beeline -u "jdbc:hive2://`hostname -f`:10001/;transportMode=http" -n "" -p "" -i settings.hql -f ddl/createAllExternalTables.hql -hiveconf LOCATION=/HiveTPCDS/ -hiveconf DBNAME=tpcds
    ```
    Generate ORC tables and analyze
    ```shell
    beeline -u "jdbc:hive2://`hostname -f`:10001/;transportMode=http" -n "" -p "" -i settings.hql -f ddl/createAllORCTables.hql -hiveconf ORCDBNAME=tpcds_orc -hiveconf SOURCE=tpcds
    beeline -u "jdbc:hive2://`hostname -f`:10001/;transportMode=http" -n "" -p "" -i settings.hql -f ddl/analyze.hql -hiveconf ORCDBNAME=tpcds_orc 
    ```

5. Run the queries !
    ```shell
    beeline -u "jdbc:hive2://`hostname -f`:10001/tpcds_orc;transportMode=http" -n "" -p "" -i settings.hql -f queries/query12.sql 
    ```

If you want to run all the queries 10 times and measure the times it takes, you can use the following command:

    for f in queries/*.sql; do for i in {1..10} ; do STARTTIME="`date +%s`";  beeline -u "jdbc:hive2://`hostname -f`:10001/tpcds_orc;transportMode=http" -i settings.hql -f $f  > $f.run_$i.out 2>&1 ; SUCCESS=$? ; ENDTIME="`date +%s`"; echo "$f,$i,$SUCCESS,$STARTTIME,$ENDTIME,$(($ENDTIME-$STARTTIME))" >> times_orc.csv; done; done;


## FAQ

#### Does it work with scale factor 1?

    No. The parrellel data generation assumes that scale > 1. If you are just starting out, I would suggest you start with 10 and then move to standard higher scale factors (100, 1000, 10000,..)

#### Do I have to specify PARTS=SCALE ?

    Yes.

#### How do I avoid my session getting killed due to network errors while long running benchmark?
    
   Use byobu. Type byobu which will start a new session and then run the command. It will be there when you come back even if your network connection is broken. 
   
#### How do I generate partitioned text tables ?   
   After generating raw data(step 3a), use the following command:
    
   ```
   hive -i settings.hql -f ddl/createAllTextTables.hql -hiveconf TEXTDBNAME=tpcds_text -hiveconf SOURCE=tpcds
   ```
    
   This will generate tpcds_text database with all the tables in text format.
   
#### How do I generate Parquet data?

   After generating raw data(step 3a), use the following command:
    
   ```
   hive -i settings.hql -f ddl/createAllParquetTables.hql -hiveconf PARQUETDBNAME=tpcds_pqt -hiveconf SOURCE=tpcds
   ```
    
   This will generate tpcds_pqt database with all the tables in parquet format.

#### How do I run the queries with Spark?
   
   Spark thriftserver listens on 10002 instead of hive thrift server listening on 10001. So replace the connection url appropriately. For example, running the all the queries 10 times with Spark,
   
   ```
for f in queries/*.sql; do for i in {1..10} ; do STARTTIME="`date +%s`";  beeline -u "jdbc:hive2://`hostname -f`:10002/tpcds_orc;transportMode=http" -i sparksettings.hql -f $f  > $f.run_$i.out 2>&1 ; SUCCESS=$? ; ENDTIME="`date +%s`"; echo "$f,$i,$SUCCESS,$STARTTIME,$ENDTIME,$(($ENDTIME-$STARTTIME))" >> times_orc.csv; done; done;
   ```

#### How do I run the queries with [Presto](https://github.com/hdinsight/presto-hdinsight)?
   
   ```
   presto --schema tpcds_orc -f queries/query12.sql
   ```
   
   You can run all the queries 10 times with presto with the following command,
   
   ```
   for f in queries/*.sql; do for i in {1..10} ; do STARTTIME="`date +%s`"; presto --schema tpcds_orc -f $f  > $f.run_$i.out 2>&1 ; SUCCESS=$? ; ENDTIME="`date +%s`"; echo "$f,$i,$SUCCESS,$STARTTIME,$ENDTIME,$(($ENDTIME-$STARTTIME))" >> times_orc.csv; done; done;
   ```
