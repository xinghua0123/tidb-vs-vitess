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
