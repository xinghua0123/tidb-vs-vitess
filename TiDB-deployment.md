# TiDB Deployment on EKS

## Deploy an EKS Cluster
1. Install ```eksctl``` – A command line tool for working with EKS clusters that automates many individual tasks. Taking macOS as an example:
   ```shell
   brew install weaveworks/tap/eksctl
   ```
2. Install ```kubectl``` :
   ```shell
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
   kubectl version --client
   ```
3. Install ```helm```:
   ```shell
   curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
   chmod 700 get_helm.sh
   ./get_helm.sh  
   ```
4. Prepare a cluster setup YAML file. Sample ```tidb-eks-cluster.yaml``` creates a TiDB cluster consisting of 3 TiDB server, 1 PD server and 5 TiKV servers:
   ```yaml
    apiVersion: eksctl.io/v1alpha5
    kind: ClusterConfig
    metadata:
    name: tidb-cluster
    region: ap-southeast-1
    nodeGroups:
    - name: admin
        desiredCapacity: 1
        instanceType: m5.xlarge
        privateNetworking: true
        availabilityZones: ["ap-southeast-1a"] 
        labels:
        dedicated: admin
    - name: tidb-1a
        desiredCapacity: 3
        instanceType: c5.2xlarge
        privateNetworking: true
        availabilityZones: ["ap-southeast-1a"]
        labels:
        dedicated: tidb
        taints:
        dedicated: tidb:NoSchedule
    - name: pd-1a
        desiredCapacity: 1
        instanceType: c5.2xlarge
        privateNetworking: true
        availabilityZones: ["ap-southeast-1a"]
        labels:
        dedicated: pd
        taints:
        dedicated: pd:NoSchedule
    - name: tikv-1a
        desiredCapacity: 5
        instanceType: r5.2xlarge
        privateNetworking: true
        availabilityZones: ["ap-southeast-1a"]
        labels:
        dedicated: tikv
        taints:
        dedicated: tikv:NoSchedule
   ```
5. Deploy the EKS cluster:
   ```shell
   eksctl create cluster -f tidb-eks-cluster.yaml
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
3. Verify that new storageclass is successfully created.
   ```
   kubectl get sc
   NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
   gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  28h
   gp3             ebs.csi.aws.com         Delete          WaitForFirstConsumer   false                  24h
   ```
4. Make gp3 as default storageclass.
   ```shell
   kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
   kubectl patch storageclass gp3 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
   ```
## Deploy TiDB Cluster on EKS

### Deploy TiDB Operator 
1. Install CRDs into the EKS cluster
   ```shell
   kubectl apply -f https://raw.githubusercontent.com/pingcap/tidb-operator/v1.1.11/manifests/crd.yaml
   ```

2. Install the operator
   ```shell
   helm repo add pingcap https://charts.pingcap.org/
   kubectl create namespace tidb-admin
   helm install --namespace tidb-admin tidb-operator pingcap/tidb-operator --version v1.1.11
   ```
   To confirm that the TiDB Operator components are running, execute the following command:
   ```shell
   kubectl get pods --namespace tidb-admin -l app.kubernetes.io/instance=tidb-operator
   ```
   Expected Output:
   ```shell
    NAME                                       READY   STATUS    RESTARTS   AGE
    tidb-controller-manager-6d8d5c6d64-b8lv4   1/1     Running   0          2m22s
    tidb-scheduler-644d59b46f-4f6sb            2/2     Running   0          2m22s   
   ```

### Deploy TiDB Cluster and Monitor 
1. Deploy TiDB cluster.
   ```shell
   kubectl create namespace tidb-cluster && \
   kubectl -n tidb-cluster apply -f https://raw.githubusercontent.com/pingcap/tidb-operator/master/examples/basic/tidb-cluster.yaml
   ```
   Feel free to edit the TiDB Cluster CR yaml:
   ```yaml
    apiVersion: pingcap.com/v1alpha1
    kind: TidbCluster
    metadata:
    name: tidb-test
    namespace: tidb-cluster
    spec:
    version: v4.0.10
    timezone: UTC
    configUpdateStrategy: RollingUpdate
    pvReclaimPolicy: Retain
    schedulerName: tidb-scheduler
    enableDynamicConfiguration: true
    pd:
        baseImage: pingcap/pd
        version: "v4.0.11"
        replicas: 1
        requests:
        storage: "50Gi"
        config:
        log:
            level: info
        replication:
            location-labels:
            - zone
            max-replicas: 1
        nodeSelector:
        dedicated: pd
        tolerations:
        - effect: NoSchedule
        key: dedicated
        operator: Equal
        value: pd
        service:
        annotations:
            service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: 'true'
            # service.beta.kubernetes.io/aws-load-balancer-internal: '0.0.0.0/0'
            service.beta.kubernetes.io/aws-load-balancer-type: nlb 
        exposeStatus: true
        externalTrafficPolicy: Local
        type: LoadBalancer
    tikv:
        baseImage: pingcap/tikv
        version: "v4.0.11"
        replicas: 5
        requests:
        storage: "150Gi"
        config: {}
        nodeSelector:
        dedicated: tikv
        tolerations:
        - effect: NoSchedule
        key: dedicated
        operator: Equal
        value: tikv
    tidb:
        baseImage: pingcap/tidb
        version: "v4.0.11"
        replicas: 3
        service:
        annotations:
            service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: 'true'
            # service.beta.kubernetes.io/aws-load-balancer-internal: '0.0.0.0/0'
            service.beta.kubernetes.io/aws-load-balancer-type: nlb 
        exposeStatus: true
        externalTrafficPolicy: Local
        type: LoadBalancer
        config:
        log:
            level: info
        performance:
            max-procs: 0
            tcp-keep-alive: true
        annotations:
        tidb.pingcap.com/sysctl-init: "true"
        podSecurityContext:
        sysctls:
        - name: net.ipv4.tcp_keepalive_time
            value: "300"
        - name: net.ipv4.tcp_keepalive_intvl
            value: "75"
        - name: net.core.somaxconn
            value: "32768"
        separateSlowLog: true
        nodeSelector:
        dedicated: tidb
        tolerations:
        - effect: NoSchedule
        key: dedicated
        operator: Equal
        value: tidb
   ```
   
2. Deploy TiDB monitor </br>
   ```yaml
    apiVersion: pingcap.com/v1alpha1
    kind: TidbMonitor
    metadata:
    name: tidb-monitor
    namespace: tidb-cluster
    spec:
    alertmanagerURL: ""
    annotations: {}
    clusters:
    - name: tidb-test
    grafana:
        baseImage: grafana/grafana
        envs:
        # Configure Grafana using environment variables except GF_PATHS_DATA, GF_SECURITY_ADMIN_USER and GF_SECURITY_ADMIN_PASSWORD
        # Ref https://grafana.com/docs/installation/configuration/#using-environment-variables
        GF_AUTH_ANONYMOUS_ENABLED: "true"
        GF_AUTH_ANONYMOUS_ORG_NAME: "Main Org."
        GF_AUTH_ANONYMOUS_ORG_ROLE: "Viewer"
        # if grafana is running behind a reverse proxy with subpath http://foo.bar/grafana
        # GF_SERVER_DOMAIN: foo.bar
        # GF_SERVER_ROOT_URL: "%(protocol)s://%(domain)s/grafana/"
        imagePullPolicy: IfNotPresent
        logLevel: info
        password: admin
        service:
        portName: http-grafana
        type: LoadBalancer
        username: admin
        version: 6.1.6
    imagePullPolicy: IfNotPresent
    initializer:
        baseImage: pingcap/tidb-monitor-initializer
        imagePullPolicy: IfNotPresent
        version: v4.0.11
    kubePrometheusURL: ""
    persistent: true
    prometheus:
        baseImage: prom/prometheus
        imagePullPolicy: IfNotPresent
        logLevel: info
        reserveDays: 12
        service:
        portName: http-prometheus
        type: NodePort
        version: v2.18.1
    reloader:
        baseImage: pingcap/tidb-monitor-reloader
        imagePullPolicy: IfNotPresent
        service:
        portName: tcp-reloader
        type: NodePort
        version: v1.0.1
    storage: 50Gi
   ```
3. Verify the cluster is up and running.
   ```shell
    kubectl get pods -n tidb-cluster
    NAME                                    READY   STATUS    RESTARTS   AGE
    tidb-monitor-monitor-777c9675b7-d2dj5   3/3     Running   0          19h
    tidb-test-discovery-6c45f88cbd-hhnt2    1/1     Running   0          19h
    tidb-test-pd-0                          1/1     Running   0          19h
    tidb-test-tidb-0                        2/2     Running   0          19h
    tidb-test-tidb-1                        2/2     Running   0          19h
    tidb-test-tidb-2                        2/2     Running   0          19h
    tidb-test-tikv-0                        1/1     Running   0          19h
    tidb-test-tikv-1                        1/1     Running   0          19h
    tidb-test-tikv-2                        1/1     Running   0          19h
    tidb-test-tikv-3                        1/1     Running   0          19h
    tidb-test-tikv-4                        1/1     Running   0          19h
   ```
### Create Database and Shards for YCSB

1. Create YCSB schemas
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
2. At this point, you should be able to see the table schema in MySQL
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
