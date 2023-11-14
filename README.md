# eks-setup
EKS Setup Instructions and Demos

Disclaimer : Samples in this repo are for training purposes only. Please refer to official documentation and best practices when deploying to production


Install AWS CLI v2


```
sudo yum remove awscli

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
```




Install Kubectl


```
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.28.2/2023-10-17/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH
echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
```


Install eksctl

https://eksctl.io/

```
# for ARM systems, set ARCH to: `arm64`, `armv6` or `armv7`
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# (Optional) Verify checksum
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz

sudo mv /tmp/eksctl /usr/local/bin
```


Create Cluster


config.yaml
```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: demo-eks-cluster
  region: us-east-1
  version: "1.28"


vpc:
  id: "vpc-0e1234567890"  
  subnets:
    private:
      us-east-1a:
        id: "subnet-0a1234567890"
      us-east-1b:
        id: "subnet-0b1234567890"
      us-east-1c:
        id: "subnet-0b1234567890"

managedNodeGroups:
  - name: data-plane
    instanceType: t3.medium
    minSize: 2
    maxSize: 4
    desiredCapacity: 3
    volumeSize: 50
    privateNetworking: true
    labels: {role: worker}
    tags:
      nodegroup-role: worker
    iam:
      withAddonPolicies:
        externalDNS: true
        certManager: true

cloudWatch:
  clusterLogging:
    # enable specific types of cluster control plane logs
    enableTypes: [ "*" ]
    # all supported types: "api", "audit", "authenticator", "controllerManager", "scheduler"
    # supported special values: "*" and "all"

    # Sets the number of days to retain the logs for (see [CloudWatch docs](https://docs.aws.amazon.com/AmazonCloudWatchLogs/latest/APIReference/API_PutRetentionPolicy.html#API_PutRetentionPolicy_RequestSyntax)).
    # By default, log data is stored in CloudWatch Logs indefinitely.
    logRetentionInDays: 30

iam:
  withOIDC: true

addons:
- name: vpc-cni
  # all below properties are optional
  version: v1.15.1-eksbuild.1
  attachPolicyARNs:
  - arn:aws:iam::account:policy/AmazonEKS_CNI_Policy
- name: coredns
  version: latest # auto discovers the latest available
- name: kube-proxy
  version: latest
- name: aws-ebs-csi-driver
  wellKnownPolicies:      # add IAM and service account
    ebsCSIController: true
  
```

```
eksctl create cluster -f config.yaml
```

Install HELM


If you're using Linux, install the binaries with the following commands.

```
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
```




Set up an Application Load Balancer using the AWS Load Balancer Controller



Tag your subnets to allow auto discovery


Tag the Amazon VPC subnets in your Amazon EKS cluster to allow your AWS Load Balancer Controller to autodiscover subnets when the Application Load Balancer resource is created.

The Kubernetes Cloud Controller Manager (cloud-controller-manager) and AWS Load Balancer Controller (aws-load-balancer-controller) query a cluster's subnets to identify them. This query uses the following tag as a filter:
```
Key: kubernetes.io/cluster/cluster-name 
value: cluster-name

key: kubernetes.io/cluster/<ClusterName>
value: owned
```

For public Application Load Balancers, you must have at least two public subnets in your cluster's VPC with the following tags:
```
key: kubernetes.io/role/elb
value: 1
```

For internal Application Load Balancers, you must have at least two private subnets in your cluster's VPC with the following tags:
```
key: kubernetes.io/role/internal-elb
value: 1
```

Create an IAM policy for the AWS Load Balancer Controller

The Amazon EKS policy that you create allows the AWS Load Balancer Controller to make calls to AWS APIs on your behalf. It's a best practice to use AWS IAM roles for service accounts when you grant access to AWS APIs.

1.    To download an IAM policy document for the AWS Load Balancer Controller from AWS GitHub, run one of the following commands based on your Region.

All Regions other than China Regions:

```
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

2.    To create an IAM policy named AWSLoadBalancerControllerIAMPolicy for your worker node instance profile, run the following command:

```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json
```

3.    Create an IAM role. Create a Kubernetes service account named aws-load-balancer-controller in the kube-system namespace for the AWS Load Balancer Controller and annotate the Kubernetes service account with the name of the IAM role.

Replace my-clusterwith the name of your cluster, 111122223333 with your account ID, and then run the command. If your cluster is in the AWS GovCloud (US-East) or AWS GovCloud (US-West) AWS Regions, then replace arn:aws:with arn:aws-us-gov:.
```
$ eksctl create iamserviceaccount \
  --cluster=demo-eks-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::111122223333:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

4.    Install the AWS Load Balancer Controller using Helm V3 or later or by applying a Kubernetes manifest. If you want to deploy the controller on Fargate, use the Helm procedure. The Helm procedure doesn't depend on cert-manager because it generates a self-signed certificate.

a.   Add the eks-charts repository
```
helm repo add eks https://aws.github.io/eks-charts
```

b.   Update your local repo to make sure that you have the most recent charts.
```
helm repo update eks
```

c.   Install the AWS Load Balancer Controller. If you're deploying the controller to Amazon EC2 nodes that have restricted access to the Amazon EC2 instance metadata service (IMDS), or if you're deploying to Fargate, then add the following flags to the helm command that follows:

```
--set region=region-code
--set vpcId=vpc-xxxxxxxx
```

Replace my-cluster with the name of your cluster. In the following command, aws-load-balancer-controller is the Kubernetes service account that you created in a previous step.

```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-eks-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```


Troubleshoot AWS Loadbalancer Controller
https://repost.aws/knowledge-center/eks-load-balancer-controller-subnets

Create namespace

Create Service Account


Create Policy to Read Secret

Policy Document
```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "ReadSecretFromEKS",
			"Effect": "Allow",
			"Action": "secretsmanager:GetSecretValue",
			"Resource": "arn:aws:secretsmanager:us-east-1:11122334455:secret:prod/nginx-svc/token-xxxxx"
		}
	]
}
```

Create Role for iamserviceaccount

Edit Trust Relationship
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::1122334455:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/123ABCDE2B3406649B5643F1ECABCVD"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.us-east-1.amazonaws.com/id/123ABCDE2B3406649B5643F1ECABCVD:sub": "system:serviceaccount:production:nginx"
                }
            }
        }
    ]
}
```


Use AWS Secrets Manager secrets in Amazon Elastic Kubernetes Service

https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_csi_driver.html

```
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install -n kube-system --set syncSecret.enabled=true csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver

helm repo add aws-secrets-manager https://aws.github.io/secrets-store-csi-driver-provider-aws
helm install -n kube-system secrets-provider-aws aws-secrets-manager/secrets-store-csi-driver-provider-aws
```
https://github.com/kubernetes-sigs/secrets-store-csi-driver/blob/221ef44211750661b2407e608ff707517d7a940a/charts/secrets-store-csi-driver/values.yaml





