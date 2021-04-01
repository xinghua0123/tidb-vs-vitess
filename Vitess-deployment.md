# Vitess Deployment on EKS

## Deploy an EKS Cluster
1. Install ```eksctl``` – A command line tool for working with EKS clusters that automates many individual tasks. Taking macOS as an example:
   ```shell
   brew install weaveworks/tap/eksctl
   ```

2. Prepare a cluster setup YAML file. Sample ```vitess-eks-cluster.yaml```:
   ```yaml
    apiVersion: eksctl.io/v1alpha5
    kind: ClusterConfig
    metadata:
      name: vitess-eks
      region: ap-southeast-1
    nodeGroups:
      - name: admin
        desiredCapacity: 1
        instanceType: m5.xlarge # replace with your instance type
        privateNetworking: true
        availabilityZones: ["ap-southeast-1a"] # replace with your preferred AWS region
      - name: vitess-1a
        desiredCapacity: 9
        instanceType: m5.2xlarge # replace with your instance type
        privateNetworking: true
        availabilityZones: ["ap-southeast-1a"] # replace with your preferred AWS region
   ```
3. Deploy the EKS cluster:
   ```shell
   eksctl create cluster -f vitess-eks-cluster.yaml
   ```
   Wait for the deployment of EKS cluster completes.

## Deploy EBS CSI Driver and set up EBS gp3 storageclass

### Set up driver permission </br>
The driver requires IAM permission to talk to Amazon EBS to manage the volume on user's behalf. Using secret object - create an IAM user with proper permission, put that user's credentials in secret manifest then deploy the secret.
   ```shell
   curl https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/deploy/kubernetes/secret.yaml > secret.yaml
   # Edit the secret with user credentials
   kubectl apply -f secret.yaml
   ```
### Create IAM Policy </br>
1. Download the IAM policy document from GitHub.
   ```shell
   curl -o example-iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/v0.9.0/docs/example-iam-policy.json
   ```
2. Create the policy. You can change ```AmazonEKS_EBS_CSI_Driver_Policy``` to a different name, but please remain consistent for the rest of the steps. And make sure ```aws``` client is installed.
   ```shell
   aws iam create-policy --policy-name AmazonEKS_EBS_CSI_Driver_Policy \
   --policy-document file://example-iam-policy.json
   ```
### Create an IAM role and attach the IAM policy</br>
1. View your cluster's OIDC provider URL. Replace <cluster_name> (including <>) with your cluster name.
   ```shell
   aws eks describe-cluster --name <cluster_name> --query "cluster.identity.oidc.issuer" --output text
   ```
   Output:
   ```
   https://oidc.eks.ap-southeast-1.amazonaws.com/id/C6DXXXXXXXXXXXXXXX60CC266F1078C
   ```
2. Create IAM role </br>
   Copy the following contents to a file named trust-policy.json. Replace <AWS_ACCOUNT_ID> (including <>) with your account ID and <XXXXXXXXXX45D83924220DC4815XXXXX> with the value returned in the previous step.
   ```json
      {
        "Version": "2012-10-17",
        "Statement": [
          {
            "Effect": "Allow",
            "Principal": {
              "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/oidc.eks.us-west-2.amazonaws.com/id/<XXXXXXXXXX45D83924220DC4815XXXXX>"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
              "StringEquals": {
                "oidc.eks.us-west-2.amazonaws.com/id/<XXXXXXXXXX45D83924220DC4815XXXXX>:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
              }
            }
          }
        ]
      }
   ```
   </br>

   Create the role. You can change```AmazonEKS_EBS_CSI_DriverRole```to a different name, but please remain consistent for the rest of the steps.</br>
   
   ```shell
   aws iam create-role \
   --role-name AmazonEKS_EBS_CSI_DriverRole \
   --assume-role-policy-document file://"trust-policy.json"
   ```
3. Attach the IAM policy to the role. </br>
   Replace <AWS_ACCOUNT_ID> (including <>) with your account ID.</br>
   ```shell
   aws iam attach-role-policy \
   --policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AmazonEKS_EBS_CSI_Driver_Policy \
   --role-name AmazonEKS_EBS_CSI_DriverRole
   ```
### Deploy the driver
1. Deploy the driver.
   ```shell
   # Don't use the master branch, use the version 0.9
   kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-0.9"
   ```
2. Annotate the ebs-csi-controller-sa Kubernetes service account with the ARN of the IAM role that you created previously. Replace the <AWS_ACCOUNT_ID> (including <>) with your account ID. </br>
   
   ```
   kubectl annotate serviceaccount ebs-csi-controller-sa \
   -n kube-system \
   eks.amazonaws.com/role-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:role/AmazonEKS_EBS_CSI_DriverRole
   ```
   </br>
   
3. Delete the driver pods. They're automatically redeployed with the IAM permissions from the IAM policy assigned to the role. </br>
   
   ```shell
   kubectl delete pods \
   -n kube-system \
   -l=app=ebs-csi-controller
   ```
### Deploy gp3 storageclass
1. Prepare the storagleclass YAML ```gp3.yaml```file.
   ```yaml
   kind: StorageClass
   apiVersion: storage.k8s.io/v1
   metadata:
   name: gp3
   provisioner: ebs.csi.aws.com
   volumeBindingMode: WaitForFirstConsumer
   parameters:
   csi.storage.k8s.io/fstype: xfs # must be xfs
   type: gp3
   iops: "10000" # up to 16000
   throughput: "500" # up to 1000 MB/s
   ```
2. Deploy gp3 storageclass
   ```
   kubectl apply -f gp3.yaml
   ```
3. Verify that new storageclass is successfully created.
   ```
   kubectl get sc
   NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
   gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  28h
   gp3             ebs.csi.aws.com         Delete          WaitForFirstConsumer   false                  24h
   ```
## Deploy Vitess Cluster on EKS

### Deploy Vitess Operator and Cluster
1. Change to the operator example directory
   ```shell
   git clone git@github.com:vitessio/vitess.git
   cd vitess/examples/operator
   ```

2. Install the operator
   ```shell
   kubectl apply -f operator.yaml
   ```

3. Installation of ```vtctlclient``` locally
   Download the file and ```untar``` it.
   ```
   wget https://github.com/vitessio/vitess/releases/download/v9.0.0/vitess-9.0.0-daa6085.tar.gz
   ```
   Create a softlink for the ```vtctlclient```
   ```
   ln -s vitess-9.0.0-daa608/bin/vtctlclient /usr/local/bin/vtctlclient
   ```
   And make sure the executable is working as expected:
   ```shell
   vtctlclient
   E0330 15:55:13.723108   45320 main.go:57] please specify -server <vtctld_host:vtctld_port> to specify the vtctld server to connect to
   ```
4. Deploy the initial cluster </br>
   Edit 101_initial_cluster.yaml for a desired topology. Example below shows a vitess cluster with 3 shards and each shard has 3 replicas including master.
   ```yaml
   # The following example is minimalist. The security policies
   # and resource specifications are not meant to be used in production.
   # Please refer to the operator documentation for recommendations on
   # production settings.
   apiVersion: planetscale.com/v2
   kind: VitessCluster
   metadata:
   name: my-vitess
   spec:
   images:
      vtctld: vitess/lite:latest
      vtgate: vitess/lite:latest
      vttablet: vitess/lite:latest
      vtbackup: vitess/lite:latest
      mysqld:
         mysql56Compatible: vitess/lite:latest
      mysqldExporter: prom/mysqld-exporter:v0.11.0
   cells:
   - name: zone1
      gateway:
         authentication:
         static:
            secret:
               name: example-cluster-config
               key: users.json
         replicas: 1
         resources:
         requests:
            cpu: 100m
            memory: 256Mi
         limits:
            memory: 256Mi
   vitessDashboard:
      cells:
      - zone1
      extraFlags:
         security_policy: read-only
      replicas: 1
      resources:
         limits:
         memory: 128Mi
         requests:
         cpu: 100m
         memory: 128Mi
   keyspaces:
   - name: commerce # a keyspace(database) named commerce will be created
      turndownPolicy: Immediate
      partitionings:
      - equal:
         parts: 3 # number of shards
         shardTemplate:
            databaseInitScriptSecret:
               name: example-cluster-config
               key: init_db.sql
            replication:
               enforceSemiSync: true
            tabletPools:
            - cell: zone1
               type: replica
               replicas: 3 # total number of replicas including master
               vttablet:
               extraFlags:
                  db_charset: utf8mb4
               resources:
                     limits:
                     cpu: 1
                     memory: 6Gi
               mysqld:
               resources:
                     cpu: 6
                     memory: 22Gi
               dataVolumeClaimTemplate:
               accessModes: ["ReadWriteOnce"]
               storageClassName: gp3 # Set the mysql storageclass to be gp3. etcd storageclass will use gp2 
               resources:
                  requests:
                     storage: 100Gi
   updateStrategy:
      type: Immediate
   ---
   apiVersion: v1
   kind: Secret
   metadata:
   name: example-cluster-config
   type: Opaque
   stringData:
   users.json: |
      {
         "user": [{
         "UserData": "user",
         "Password": ""
         }]
      }
   init_db.sql: |
      # This file is executed immediately after mysql_install_db,
      # to initialize a fresh data directory.

      ###############################################################################
      # Equivalent of mysql_secure_installation
      ###############################################################################

      # Changes during the init db should not make it to the binlog.
      # They could potentially create errant transactions on replicas.
      SET sql_log_bin = 0;
      # Remove anonymous users.
      DELETE FROM mysql.user WHERE User = '';

      # Disable remote root access (only allow UNIX socket).
      DELETE FROM mysql.user WHERE User = 'root' AND Host != 'localhost';

      # Remove test database.
      DROP DATABASE IF EXISTS test;

      ###############################################################################
      # Vitess defaults
      ###############################################################################

      # Vitess-internal database.
      CREATE DATABASE IF NOT EXISTS _vt;
      # Note that definitions of local_metadata and shard_metadata should be the same
      # as in production which is defined in go/vt/mysqlctl/metadata_tables.go.
      CREATE TABLE IF NOT EXISTS _vt.local_metadata (
         name VARCHAR(255) NOT NULL,
         value VARCHAR(255) NOT NULL,
         db_name VARBINARY(255) NOT NULL,
         PRIMARY KEY (db_name, name)
         ) ENGINE=InnoDB;
      CREATE TABLE IF NOT EXISTS _vt.shard_metadata (
         name VARCHAR(255) NOT NULL,
         value MEDIUMBLOB NOT NULL,
         db_name VARBINARY(255) NOT NULL,
         PRIMARY KEY (db_name, name)
         ) ENGINE=InnoDB;

      # Admin user with all privileges.
      CREATE USER 'vt_dba'@'localhost';
      GRANT ALL ON *.* TO 'vt_dba'@'localhost';
      GRANT GRANT OPTION ON *.* TO 'vt_dba'@'localhost';

      # User for app traffic, with global read-write access.
      CREATE USER 'vt_app'@'localhost';
      GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, RELOAD, PROCESS, FILE,
         REFERENCES, INDEX, ALTER, SHOW DATABASES, CREATE TEMPORARY TABLES,
         LOCK TABLES, EXECUTE, REPLICATION SLAVE, REPLICATION CLIENT, CREATE VIEW,
         SHOW VIEW, CREATE ROUTINE, ALTER ROUTINE, CREATE USER, EVENT, TRIGGER
         ON *.* TO 'vt_app'@'localhost';

      # User for app debug traffic, with global read access.
      CREATE USER 'vt_appdebug'@'localhost';
      GRANT SELECT, SHOW DATABASES, PROCESS ON *.* TO 'vt_appdebug'@'localhost';

      # User for administrative operations that need to be executed as non-SUPER.
      # Same permissions as vt_app here.
      CREATE USER 'vt_allprivs'@'localhost';
      GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, RELOAD, PROCESS, FILE,
         REFERENCES, INDEX, ALTER, SHOW DATABASES, CREATE TEMPORARY TABLES,
         LOCK TABLES, EXECUTE, REPLICATION SLAVE, REPLICATION CLIENT, CREATE VIEW,
         SHOW VIEW, CREATE ROUTINE, ALTER ROUTINE, CREATE USER, EVENT, TRIGGER
         ON *.* TO 'vt_allprivs'@'localhost';

      # User for slave replication connections.
      # TODO: Should we set a password on this since it allows remote connections?
      CREATE USER 'vt_repl'@'%';
      GRANT REPLICATION SLAVE ON *.* TO 'vt_repl'@'%';

      # User for Vitess filtered replication (binlog player).
      # Same permissions as vt_app.
      CREATE USER 'vt_filtered'@'localhost';
      GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, RELOAD, PROCESS, FILE,
         REFERENCES, INDEX, ALTER, SHOW DATABASES, CREATE TEMPORARY TABLES,
         LOCK TABLES, EXECUTE, REPLICATION SLAVE, REPLICATION CLIENT, CREATE VIEW,
         SHOW VIEW, CREATE ROUTINE, ALTER ROUTINE, CREATE USER, EVENT, TRIGGER
         ON *.* TO 'vt_filtered'@'localhost';

      # User for Orchestrator (https://github.com/openark/orchestrator).
      # TODO: Reenable when the password is randomly generated.
      #CREATE USER 'orc_client_user'@'%' IDENTIFIED BY 'orc_client_user_password';
      #GRANT SUPER, PROCESS, REPLICATION SLAVE, RELOAD
      #  ON *.* TO 'orc_client_user'@'%';
      #GRANT SELECT
      #  ON _vt.* TO 'orc_client_user'@'%';

      FLUSH PRIVILEGES;

      RESET SLAVE ALL;
      RESET MASTER;
   ```

   ```shell
   kubectl apply -f 101_initial_cluster.yaml
   ```
5. Verify the cluster is up and running.
   ```
   kubectl get pods
   NAME                                               READY   STATUS    RESTARTS   AGE
   my-vitess-etcd-40b4cbc2-1                          1/1     Running   0          36m
   my-vitess-etcd-40b4cbc2-2                          1/1     Running   0          36m
   my-vitess-etcd-40b4cbc2-3                          1/1     Running   0          36m
   my-vitess-vttablet-zone1-0069280532-3b978d46       3/3     Running   1          36m
   my-vitess-vttablet-zone1-0406690258-39fb446b       3/3     Running   1          36m
   my-vitess-vttablet-zone1-0598971861-9dc5be68       3/3     Running   1          36m
   my-vitess-vttablet-zone1-1124725830-145759d8       3/3     Running   1          36m
   my-vitess-vttablet-zone1-1320336861-1b8e7787       3/3     Running   1          36m
   my-vitess-vttablet-zone1-2070837339-62bca317       3/3     Running   1          36m
   my-vitess-vttablet-zone1-2195186634-51d48922       3/3     Running   1          36m
   my-vitess-vttablet-zone1-3157751654-76a5f21a       3/3     Running   1          36m
   my-vitess-vttablet-zone1-4067224458-48d0dcdc       3/3     Running   1          36m
   my-vitess-zone1-vtctld-63f67451-7975b46f9d-xnz88   1/1     Running   2          36m
   my-vitess-zone1-vtgate-0ca8a771-548dc45d54-f7q57   1/1     Running   2          36m
   vitess-operator-9874b4ff-9tpbl                     1/1     Running   0          24h
   ```
### Create Vitess Database and Shards for YCSB
1. Setup Port-forward
   For ease-of-use, Vitess provides a script to port-forward from Kubernetes to your local machine. This script also recommends setting up aliases for ```mysql``` and ```vtctlclient```:
   ```
   ./pf.sh &
   alias vtctlclient="vtctlclient -server=localhost:15999"
   alias mysql="mysql -h 127.0.0.1 -P 15306 -u user"
   ```
2. As specified in the ```101_initial_cluster.yaml```, a keyspace(database) ```commerce``` will be created.
   ```sql
   [ec2-user@ip-192-168-85-148 ycsb-0.17.0]$ mysql
   Handling connection for 15306
   Welcome to the MariaDB monitor.  Commands end with ; or \g.
   Your MySQL connection id is 375
   Server version: 5.7.9-vitess-10.0.0-SNAPSHOT Version: 10.0.0-SNAPSHOT (Git revision 92fd95324 branch 'master') built on Tue Mar 30 02:32:03 UTC 2021 by vitess@d9474db7708a using go1.15.6 linux/amd64

   Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

   Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

   MySQL [(none)]> show databases;
   +--------------------+
   | Database           |
   +--------------------+
   | information_schema |
   | mysql              |
   | sys                |
   | performance_schema |
   | commerce           |
   +--------------------+
   5 rows in set (0.00 sec)
   ```
3. Create YCSB schemas
   ```usertable```table structure as defined in ```create_commerce_schema.sql```:
   ```sql
   create table usertable(
   YCSB_KEY varchar(64)  NOT NULL,
   FIELD0 varchar(100)  ,
   FIELD1 varchar(100)  ,
   FIELD2 varchar(100)  ,
   FIELD3 varchar(100)  ,
   FIELD4 varchar(100)  ,
   FIELD5 varchar(100)  ,
   FIELD6 varchar(100)  ,
   FIELD7 varchar(100)  ,
   FIELD8 varchar(100)  ,
   FIELD9 varchar(100)  ,
   primary key(YCSB_KEY)
   ) ENGINE=InnoDB;
   ```
   And table shards as defined in ```vschema_commerce_sharded.json```:
   ```json
   {
      "sharded": true,
      "vindexes": {
         "binary_md5": {
               "type": "binary_md5"
         }
      },
      "tables": {
         "usertable": {
               "column_vindexes": [
                  {
                     "column": "YCSB_KEY",
                     "name": "binary_md5"
                  }
               ]
         }
      }
   }
   ```
   Load initial schemas:
   ```shell
   vtctlclient ApplySchema -sql="$(cat create_commerce_schema.sql)" commerce
   vtctlclient ApplyVSchema -vschema="$(cat vschema_commerce_sharded.json)" commerce
   ```
4. At this point, you should be able to see the table schema in MySQL
   ```sql
   MySQL [(none)]> use commerce
   Reading table information for completion of table and column names
   You can turn off this feature to get a quicker startup with -A

   Database changed
   MySQL [commerce]> desc usertable;
   +----------+--------------+------+-----+---------+-------+
   | Field    | Type         | Null | Key | Default | Extra |
   +----------+--------------+------+-----+---------+-------+
   | YCSB_KEY | varchar(64)  | NO   | PRI | NULL    |       |
   | FIELD0   | varchar(100) | YES  |     | NULL    |       |
   | FIELD1   | varchar(100) | YES  |     | NULL    |       |
   | FIELD2   | varchar(100) | YES  |     | NULL    |       |
   | FIELD3   | varchar(100) | YES  |     | NULL    |       |
   | FIELD4   | varchar(100) | YES  |     | NULL    |       |
   | FIELD5   | varchar(100) | YES  |     | NULL    |       |
   | FIELD6   | varchar(100) | YES  |     | NULL    |       |
   | FIELD7   | varchar(100) | YES  |     | NULL    |       |
   | FIELD8   | varchar(100) | YES  |     | NULL    |       |
   | FIELD9   | varchar(100) | YES  |     | NULL    |       |
   +----------+--------------+------+-----+---------+-------+
   11 rows in set (0.03 sec)
   ```
5. Create a dedicated loadbalancer for Vitess MySQL service.
   ```shell
   kubectl expose deployment $( kubectl get deployment --selector="planetscale.com/component=vtgate" -o=jsonpath="{.items..metadata.name}" ) --type=LoadBalancer --name=test-vtgate --port 3306 --target-port 3306
   
   kubectl get service test-vtgate

   NAME          TYPE           CLUSTER-IP      EXTERNAL-IP                                                                   PORT(S)          AGE
   test-vtgate   LoadBalancer   10.100.238.40   a7e70567f99354f5885968987fe62e6c-478783606.ap-southeast-1.elb.amazonaws.com   3306:30717/TCP   7m
   ```
