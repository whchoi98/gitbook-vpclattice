---
description: 'Update : 2024.06.07'
---

# AWS Gateway API Controller 설치

아래와 같은 단계로 설치를 진행합니다.

* Step1. Client VPC에 EKS Cluster를 설치
* Step2. VPC Lattice와 AWS GW API Controller간의 통신을 위한 구성

<figure><img src="../.gitbook/assets/image (42).png" alt=""><figcaption></figcaption></figure>

### Step1. EKS Cluster 설치

#### EKS Cluster 설치 준비

[01.사전 준비 및 소개 - 2. 사전준비](../01./) 에서 구성한 Client VPC에 EKS Clsuter를 설치하기 위해 eksctl yaml을 생성합니다.

```
# eksctl 기반의 cluster 생성을 위한 yaml을 만듭니다.
# ~/environment/vpclattice/cloud9/lattice_eks01.yaml 에 파일이 생성됩니다.
~/environment/vpclattice/cloud9/eks-cluster1.sh

```

#### EKS Cluster 설치

정상적으로 eksctl yaml이 생성되었으면, 아래 명령을 통해 eks clsuter "c1"을 구성합니다.

```
cd ~/environment/vpclattice/cloud9/
eksctl create cluster -f lattice_eks01.yaml

```

#### 네트워크 보안 환경 구성

**VPC Lattice 네트워크에서 트래픽을 수신하도록 Security Group을 구성합니다.**&#x20;

VPC Lattice와 통신하는 모든 Pod가 VPC Lattice 관리형 PrefixList의 트래픽을 허용하도록 Security Group을 설정해야 합니다. [자세한 내용은 Security Group을 사용하여 자원에 대한 트래픽 제어 방법을 참조할 수 있습니다.](https://docs.aws.amazon.com/vpc/latest/userguide/VPC\_SecurityGroups.html)Lattice에는 IPv4 및 IPv6 PrefixList가 모두 있습니다.

<figure><img src="../.gitbook/assets/image (50).png" alt=""><figcaption></figcaption></figure>

```
CLUSTER1_SG=$(aws eks describe-cluster --name $CLUSTER1_NAME --output json| jq -r '.cluster.resourcesVpcConfig.clusterSecurityGroupId')
PREFIX_LIST_ID1=$(aws ec2 describe-managed-prefix-lists --query "PrefixLists[?PrefixListName=="\'com.amazonaws.$AWS_REGION.vpc-lattice\'"].PrefixListId" | jq -r '.[]')
aws ec2 authorize-security-group-ingress --group-id $CLUSTER1_SG --ip-permissions "PrefixListIds=[{PrefixListId=${PREFIX_LIST_ID1}}],IpProtocol=-1"
PREFIX_LIST_ID1_IPV6=$(aws ec2 describe-managed-prefix-lists --query "PrefixLists[?PrefixListName=="\'com.amazonaws.$AWS_REGION.ipv6.vpc-lattice\'"].PrefixListId" | jq -r '.[]')
aws ec2 authorize-security-group-ingress --group-id $CLUSTER1_SG --ip-permissions "PrefixListIds=[{PrefixListId=${PREFIX_LIST_ID1_IPV6}}],IpProtocol=-1"

```

#### IRSA 구성을 위한 환경 준비

Gateway API Controller가  AWS VPC Lattice에 접근해서, 구성이 가능하도록 정책을 설정합니다.

Gateway API를 "Invoke"할 수 있는 아래 Policy로 IAM에서 Policy(recommended-inline-policy.json)를 생성하고 나중에 사용할 수 있도록 Policy ARN을 복사합니다.

```
cat <<EoF > ~/environment/vpclattice/recommended-inline-policy.json
{
 "Version": "2012-10-17",
 "Statement": [
     {
         "Effect": "Allow",
         "Action": [
             "vpc-lattice:*",
             "ec2:DescribeVpcs",
             "ec2:DescribeSubnets",
             "ec2:DescribeTags",
             "ec2:DescribeSecurityGroups",
             "logs:CreateLogDelivery",
             "logs:GetLogDelivery",
             "logs:DescribeLogGroups",
             "logs:PutResourcePolicy",
             "logs:DescribeResourcePolicies",
             "logs:UpdateLogDelivery",
             "logs:DeleteLogDelivery",
             "logs:ListLogDeliveries",
             "tag:GetResources",
             "firehose:TagDeliveryStream",
             "s3:GetBucketPolicy",
             "s3:PutBucketPolicy"
         ],
         "Resource": "*"
     },
     {
         "Effect" : "Allow",
         "Action" : "iam:CreateServiceLinkedRole",
         "Resource" : "arn:aws:iam::*:role/aws-service-role/vpc-lattice.amazonaws.com/AWSServiceRoleForVpcLattice",
         "Condition" : {
             "StringLike" : {
                 "iam:AWSServiceName" : "vpc-lattice.amazonaws.com"
             }
         }
     },
     {
         "Effect" : "Allow",
         "Action" : "iam:CreateServiceLinkedRole",
         "Resource" : "arn:aws:iam::*:role/aws-service-role/delivery.logs.amazonaws.com/AWSServiceRoleForLogDelivery",
         "Condition" : {
             "StringLike" : {
                 "iam:AWSServiceName" : "delivery.logs.amazonaws.com"
             }
         }
     }
   ]
}
EoF

```

앞서 구성한 Json 파일을 IAM 정책으로 생성합니다.

```
aws iam create-policy \
   --policy-name VPCLatticeControllerIAMPolicy \
   --policy-document file://~/environment/vpclattice/recommended-inline-policy.json

```



### Step2. Gateway API Controller 설치 구성 준비

aws-application-networking-system 네임스페이스를 생성합니다.

```
kubectl create namespace aws-application-networking-system

```

이 랩에서 사용할 Sample Code들이 있는 "aws-application-networking-k8s" Git을 Clone 합니다.

```
cd ~/environment
git clone https://github.com/aws/aws-application-networking-k8s.git

```

앞서 생성한 policy ARN에 대한 변수를 설정합니다.

```
export VPCLatticeControllerIAMPolicyArn=$(aws iam list-policies --query 'Policies[?PolicyName==`VPCLatticeControllerIAMPolicy`].Arn' --output text)

```

Pod Level의 권한을 위해서 "iamserviceaccount" 명령어를 수행합니다.

```
eksctl create iamserviceaccount \
   --cluster=$CLUSTER1_NAME \
   --namespace=aws-application-networking-system \
   --name=gateway-api-controller \
   --attach-policy-arn=$VPCLatticeControllerIAMPolicyArn \
   --override-existing-serviceaccounts \
   --region $AWS_REGION \
   --approve
   
```

Gateway Controller를 아래와 같이 ECR에 로그인 한 이후, Helm을 통해 설치합니다.

```
# login to ECR
aws ecr-public get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin public.ecr.aws
# Run helm with either install or upgrade
helm install gateway-api-controller \
   oci://public.ecr.aws/aws-application-networking-k8s/aws-gateway-controller-chart\
   --version=v1.0.3 \
   --set=serviceAccount.create=false --namespace aws-application-networking-system \
   --set=log.level=info \
   --set=awsRegion=$AWS_REGION \
   --set=clusterVpcId=$CLUSTER_VPC_ID \
   --set=awsAccountId=$ACCOUNT_ID \
   --set=clusterName=$CLUSTER1_NAME
   
```

정상적으로 Gateway API Controller가 설치되었는지 확인합니다.

```
kubectl --namespace aws-application-networking-system get pods -l "app.kubernetes.io/instance=gateway-api-controller"

```

아래 명령어를 통해서 Gateway Class를 설치합니다.

```
kubectl apply -f ~/environment/aws-application-networking-k8s/examples/gatewayclass.yaml 

```

Gateway Class는 아래와 같은 manifest 포맷을 가지고 있습니다.

Gateway Class는 인프라 제공자가 정의하는 클러스터 범위의 resource로서 생성 가능한 gateway를 정의합니다.

VPC Lattice가 GatewayClass의 인프라 제공자가 됩니다.

```
# AWS VPC lattice provider 를 위한 Gateway Class
# istio를 사용할 경우 metadata, spec이 istio로 변경됨
apiVersion: gateway.networking.k8s.io/v1beta1
kind: GatewayClass
metadata:
  name: amazon-vpc-lattice
spec:
  controllerName: application-networking.k8s.aws/gateway-api-controller

```
