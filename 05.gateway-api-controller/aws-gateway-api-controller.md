# AWS Gateway API Controller 설치



```

~/environment/vpclattice/cloud9/eks-cluster1.sh

```

VPC Lattice 네트워크에서 트래픽을 수신하도록 Security Group을 구성합니다.&#x20;

VPC Lattice와 통신하는 모든 포드가 VPC Lattice 관리형 PrefixList의 트래픽을 허용하도록 Security Group을 설정해야 합니다. [자세한 내용은 Security Group을 사용하여 자원에 대한 트래픽 제어 방법을 참조할 수 있습니다.](https://docs.aws.amazon.com/vpc/latest/userguide/VPC\_SecurityGroups.html)Lattice에는 IPv4 및 IPv6 PrefixList가 모두 있습니다.

```
CLUSTER1_SG=$(aws eks describe-cluster --name $CLUSTER1_NAME --output json| jq -r '.cluster.resourcesVpcConfig.clusterSecurityGroupId')
PREFIX_LIST_ID1=$(aws ec2 describe-managed-prefix-lists --query "PrefixLists[?PrefixListName=="\'com.amazonaws.$AWS_REGION.vpc-lattice\'"].PrefixListId" | jq -r '.[]')
aws ec2 authorize-security-group-ingress --group-id $CLUSTER1_SG --ip-permissions "PrefixListIds=[{PrefixListId=${PREFIX_LIST_ID1}}],IpProtocol=-1"
PREFIX_LIST_ID1_IPV6=$(aws ec2 describe-managed-prefix-lists --query "PrefixLists[?PrefixListName=="\'com.amazonaws.$AWS_REGION.ipv6.vpc-lattice\'"].PrefixListId" | jq -r '.[]')
aws ec2 authorize-security-group-ingress --group-id $CLUSTER1_SG --ip-permissions "PrefixListIds=[{PrefixListId=${PREFIX_LIST_ID1_IPV6}}],IpProtocol=-1"

```



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



```
aws iam create-policy \
   --policy-name VPCLatticeControllerIAMPolicy \
   --policy-document file://~/environment/vpclattice/recommended-inline-policy.json

```



aws-application-networking-system 네임스페이스를 생성합니다.

```
kubectl create namespace aws-application-networking-system

```



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
