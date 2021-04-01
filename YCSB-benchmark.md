# tidb-vs-vitess

### Installation of YCSB
1. Java is a prerequisite and must be installed.
   ```shell
   sudo yum install java -y
   ```
2. Download the latest YCSB.
   ```shell
   curl -O --location https://github.com/brianfrankcooper/YCSB/releases/download/0.17.0/ycsb-0.17.0.tar.gz
   tar xfvz ycsb-0.17.0.tar.gz
   cd ycsb-0.17.0
   ```
3. Since there is no MySQL binding, we will use JDBC instead.</br>
   Download the MySQL JDBC connector and move it to ```lib``` directory:
   ```shell
   wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-8.0.23.tar.gz
   tar xzvf mysql-connector-java-8.0.23.tar.gz
   mv mysql-connector-java-8.0.23/mysql-connector-java-8.0.23.jar jdbc-binding/lib
   ```
4. Change the database configuration for JDBC.
   ```shell
   cd ycsb-0.17.0/jdbc-binding/conf
   vi db.properties

   db.driver=com.mysql.jdbc.Driver
   db.url=jdbc:mysql://<AWS-LB-DNS>:3306/commerce # For TiDB, use port 4000 instead, and respective database name
   db.user=user
   db.passwd=
   ```   
### Data Loading of YCSB
This would generate plenty of workloads but only of inserts. This does not simulate real-life workload which is composed of inserts/updates/deletes. This would happen in the run stage.</br>

1. Prepare the data loading.</br>
   ```shell
   ./bin/ycsb load jdbc -P ./jdbc-binding/conf/db.properties -P workloads/workloada -p recordcount=90000000 -p threadcount=200 -s 2> load_data_90M.log
   ```
   The standard workload parameter files create very small databases; for example, workloada creates only 1,000 records. This is useful while debugging your setup. However, to run an actual benchmark you'll want to generate a much larger database. For example, imagine you want to load 90 million records. You need to specify a new value of the ```recordcount``` property on the command line.</br></br>
   ```threadcount``` is the number of running threads. Think of it as the number of connecting clients.</br></br>
   The ```-s``` parameter will require the Client to produce status report on stderr. ```2> load_data_90M.log``` will output the stderr to the log file.</br>

2. When the load completes, the Client will report statistics about the performance of the load. 
   ```shell
   [OVERALL], RunTime(ms), 13536401
   [OVERALL], Throughput(ops/sec), 6648.7392032786265
   [TOTAL_GCS_G1_Young_Generation], Count, 4081
   [TOTAL_GC_TIME_G1_Young_Generation], Time(ms), 2999
   [TOTAL_GC_TIME_%_G1_Young_Generation], Time(%), 0.02215507652292511
   [TOTAL_GCS_G1_Old_Generation], Count, 0
   [TOTAL_GC_TIME_G1_Old_Generation], Time(ms), 0
   [TOTAL_GC_TIME_%_G1_Old_Generation], Time(%), 0.0
   [TOTAL_GCs], Count, 4081
   [TOTAL_GC_TIME], Time(ms), 2999
   [TOTAL_GC_TIME_%], Time(%), 0.02215507652292511
   [CLEANUP], Operations, 100
   [CLEANUP], AverageLatency(us), 250.19
   [CLEANUP], MinLatency(us), 53
   [CLEANUP], MaxLatency(us), 16023
   [CLEANUP], 95thPercentileLatency(us), 118
   [CLEANUP], 99thPercentileLatency(us), 131
   [INSERT], Operations, 90000000
   [INSERT], AverageLatency(us), 14986.335335733333
   [INSERT], MinLatency(us), 4516
   [INSERT], MaxLatency(us), 802815
   [INSERT], 95thPercentileLatency(us), 19055
   [INSERT], 99thPercentileLatency(us), 24847
   [INSERT], Return=OK, 90000000
   ```
   To do a quick cross verification:
   ```sql
   mysql> select count(YCSB_KEY) from usertable;
   +-----------------+
   | count(YCSB_KEY) |
   +-----------------+
   |       90000000 |
   +-----------------+
   1 row in set (8.46 sec)
   ```
### Running Workloads of YCSB
1. This would generate tons of inserts/update/deletes simulate real-life workload:
   ```shell
   ./bin/ycsb run jdbc -P ./jdbc-binding/conf/db.properties -P workloads/workloada -p operationcount=30000000 -p threadcount=20 -s 2> run_thread_20.log
   ```
   ```operationcount``` is the total number of operations including inserts, updates and deletes in each run.</br></br>
   ```threadcount``` is the number of running threads. Think of it as the number of connecting clients.</br></br>
