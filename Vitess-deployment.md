# Vitess Deployment on EKS

## Preparation

### Deploy an EKS Cluster
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

### Deploy EBS CSI Driver and set up EBS gp3 storageclass

#### Set up driver permission </br>
The driver requires IAM permission to talk to Amazon EBS to manage the volume on user's behalf. Using secret object - create an IAM user with proper permission, put that user's credentials in secret manifest then deploy the secret.
   ```shell
   curl https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/deploy/kubernetes/secret.yaml > secret.yaml
   # Edit the secret with user credentials
   kubectl apply -f secret.yaml
   ```
#### Create IAM Policy </br>
1. Download the IAM policy document from GitHub.
   ```shell
   curl -o example-iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/v0.9.0/docs/example-iam-policy.json
   ```
2. Create the policy. You can change ```AmazonEKS_EBS_CSI_Driver_Policy``` to a different name, but please remain consistent for the rest of the steps. And make sure ```aws``` client is installed.
   ```shell
   aws iam create-policy --policy-name AmazonEKS_EBS_CSI_Driver_Policy \
   --policy-document file://example-iam-policy.json
   ```
#### Create an IAM role and attach the IAM policy</br>
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
#### Deploy the driver
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
#### Deploy gp3 storageclass
1. Prepare the storagleclass YAML file.
   ```yaml
   kind: StorageClass
   apiVersion: storage.k8s.io/v1
   metadata:
   name: gp3
   annotations:
      storageclass.kubernetes.io/is-default-class: "true"
   provisioner: ebs.csi.aws.com
   volumeBindingMode: WaitForFirstConsumer
   parameters:
   csi.storage.k8s.io/fstype: xfs # must be xfs
   type: gp3
   iops: "10000" # up to 16000
   throughput: "500" # up to 1000
   ```




