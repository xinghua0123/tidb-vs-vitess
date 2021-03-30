# Vitess Deployment on EKS

## Prepare an EKS Cluster

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

4. 
