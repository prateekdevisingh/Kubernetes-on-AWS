## Chapter 2 Part1 : Create Kubernetes Cluster Using Kops

Chapter 2 will cover below topics:

* Configure client
* Create AWS resources
* Create Cluster with one master and two nodes
* Delete Cluster 
* Delete AWS resources

### Configure Client on Mac

Install Kops: 
```
brew update
brew install kops 
```

Install kubectl on Mac (https://kubernetes.io/docs/tasks/tools/install-kubectl/)

```
brew install kubectl
```
Check Kops version kops version. Make sure its 1.6.2 at least.

http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html[Configure AWS CLI]

### Create AWS Resources

Create IAM group: 
``` 
aws iam create-group --group-name kops 
```

Attach policy:

```
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops
```

Create IAM user: 
```
aws iam create-user --user-name kops
```
Add user to group: 
```
aws iam add-user-to-group --user-name kops --group-name kops
```
Create keys for the user: 
```
aws iam create-access-key --user-name kops
```
Note down `SecretAccessKey` and `AccessKeyId`
Configure AWS CLI: 
```aws configure
```
Use `SecretAccessKey` and `AccessKeyId`

Export keys:
```
export AWS_ACCESS_KEY_ID=<KEY>
export AWS_SECRET_ACCESS_KEY=<KEY>
```

Create S3 bucket: 
```
aws s3api create-bucket --bucket kubernetes-devops-me --region us-east-1
```
Enable bucket versioning: 
```
aws s3api put-bucket-versioning --bucket kubernetes-devops-me --region us-east-1 --versioning-configuration Status=Enabled
```
Set S3 bucket: 
```
export KOPS_STATE_STORE=s3://kubernetes-devops-me
```

### Configure One master and two nodes Cluster on AWS

Set cluster name: `export NAME=cluster.k8s.local`
Start Kubernetes cluster on AWS

```
kops create cluster \
${NAME} \
--zones us-east-1a \
--yes
```

It shows the output as:
```
I0703 12:10:25.700774   36281 create_cluster.go:655] Inferred --cloud=aws from zone "us-east-1a"
I0703 12:10:25.701240   36281 create_cluster.go:841] Using SSH public key: /Users/argu/.ssh/id_rsa.pub
I0703 12:10:26.175659   36281 subnets.go:183] Assigned CIDR 172.20.32.0/19 to subnet us-east-1a
I0703 12:10:28.930005   36281 apply_cluster.go:396] Gossip DNS: skipping DNS validation
I0703 12:10:29.709277   36281 executor.go:91] Tasks: 0 done / 67 total; 32 can run
I0703 12:10:30.619598   36281 vfs_castore.go:422] Issuing new certificate: "kops"
I0703 12:10:30.637415   36281 vfs_castore.go:422] Issuing new certificate: "kube-scheduler"
I0703 12:10:30.961460   36281 vfs_castore.go:422] Issuing new certificate: "kube-proxy"
I0703 12:10:31.088121   36281 vfs_castore.go:422] Issuing new certificate: "kube-controller-manager"
I0703 12:10:31.198301   36281 vfs_castore.go:422] Issuing new certificate: "kubecfg"
I0703 12:10:31.371058   36281 vfs_castore.go:422] Issuing new certificate: "kubelet"
I0703 12:10:32.717984   36281 executor.go:91] Tasks: 32 done / 67 total; 13 can run
I0703 12:10:34.007905   36281 executor.go:91] Tasks: 45 done / 67 total; 18 can run
I0703 12:10:35.182359   36281 launchconfiguration.go:320] waiting for IAM instance profile "masters.cluster.k8s.local" to be ready
I0703 12:10:35.226575   36281 launchconfiguration.go:320] waiting for IAM instance profile "nodes.cluster.k8s.local" to be ready
I0703 12:10:45.933390   36281 executor.go:91] Tasks: 63 done / 67 total; 3 can run
I0703 12:10:47.189627   36281 vfs_castore.go:422] Issuing new certificate: "master"
I0703 12:10:47.527929   36281 executor.go:91] Tasks: 66 done / 67 total; 1 can run
I0703 12:10:47.888263   36281 executor.go:91] Tasks: 67 done / 67 total; 0 can run
I0703 12:10:48.289931   36281 update_cluster.go:229] Exporting kubecfg for cluster
Kops has set your kubectl context to cluster.k8s.local

Cluster is starting.  It should be ready in a few minutes.

Suggestions:
 * validate cluster: kops validate cluster
 * list nodes: kubectl get nodes --show-labels
 * ssh to the master: ssh -i ~/.ssh/id_rsa admin@api.cluster.k8s.local
The admin user is specific to Debian. If not using Debian please use the appropriate user based on your OS.
 * read about installing addons: https://github.com/kubernetes/kops/blob/master/docs/addons.md
```

Wait for a few minutes and then validate the cluster: `kops validate cluster`:

```
Using cluster from kubectl context: cluster.k8s.local

Validating cluster cluster.k8s.local

INSTANCE GROUPS
NAME      ROLE  MACHINETYPE MIN MAX SUBNETS
master-us-east-1a Master  m3.medium 1 1 us-east-1a
nodes     Node  t2.medium 2 2 us-east-1a

NODE STATUS
NAME        ROLE  READY
ip-172-20-49-105.ec2.internal node  True
ip-172-20-58-78.ec2.internal  node  True
ip-172-20-61-107.ec2.internal master  True

Your cluster cluster.k8s.local is ready
```

Get nodes in the cluster using `kubectl get nodes`:

```
NAME                            STATUS         AGE       VERSION
ip-172-20-49-105.ec2.internal   Ready,node     1m        v1.6.2
ip-172-20-58-78.ec2.internal    Ready,node     1m        v1.6.2
ip-172-20-61-107.ec2.internal   Ready,master   2m        v1.6.2
```

### Delete cluster

Use kops delete command to delete the cluster and its resources

```
kops delete cluster ${NAME} --yes
```
