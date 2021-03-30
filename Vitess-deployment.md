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
   </br>
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
   kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"
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

3. Deploy the initial cluster </br>
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
   - name: commerce
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

### Create Vitess Database and Shards for YCSB




