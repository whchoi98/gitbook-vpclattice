---
description: 'Update : 2024-06-09'
---

# MultiCluster 구성

### Step1. EKS Cluster 설치 <a href="#step1.-eks-cluster" id="step1.-eks-cluster"></a>

**EKS Cluster 설치 준비**

[01.사전 준비 및 소개 - 2. 사전준비](https://whchoi98.gitbook.io/amazon-vpc-lattice/01.) 에서 구성한 Client VPC에 EKS Clsuter를 설치하기 위해 eksctl yaml을 생성합니다.

Copy

```
# eksctl 기반의 cluster 생성을 위한 yaml을 만듭니다.
# ~/environment/vpclattice/cloud9/lattice_eks02.yaml 에 파일이 생성됩니다.
~/environment/vpclattice/cloud9/eks-cluster2.sh
```

**EKS Cluster 설치**

정상적으로 eksctl yaml이 생성되었으면, 아래 명령을 통해 eks clsuter "c2"를 구성합니다.

```
cd ~/environment/vpclattice/cloud9/
eksctl create cluster -f lattice_eks02.yaml

```

### **네트워크 보안 환경 구성**

**VPC Lattice 네트워크에서 트래픽을 수신하도록 Security Group을 구성합니다.**

VPC Lattice와 통신하는 모든 Pod가 VPC Lattice 관리형 PrefixList의 트래픽을 허용하도록 Security Group을 설정해야 합니다. [자세한 내용은 Security Group을 사용하여 자원에 대한 트래픽 제어 방법을 참조할 수 있습니다.](https://docs.aws.amazon.com/vpc/latest/userguide/VPC\_SecurityGroups.html)Lattice에는 IPv4 및 IPv6 PrefixList가 모두 있습니다.

```

CLUSTER2_SG=$(aws eks describe-cluster --name $CLUSTER2_NAME --output json| jq -r '.cluster.resourcesVpcConfig.clusterSecurityGroupId')
PREFIX_LIST_ID2=$(aws ec2 describe-managed-prefix-lists --query "PrefixLists[?PrefixListName=="\'com.amazonaws.$AWS_REGION.vpc-lattice\'"].PrefixListId" | jq -r '.[]')
aws ec2 authorize-security-group-ingress --group-id $CLUSTER2_SG --ip-permissions "PrefixListIds=[{PrefixListId=${PREFIX_LIST_ID2}}],IpProtocol=-1"
PREFIX_LIST_ID2_IPV6=$(aws ec2 describe-managed-prefix-lists --query "PrefixLists[?PrefixListName=="\'com.amazonaws.$AWS_REGION.ipv6.vpc-lattice\'"].PrefixListId" | jq -r '.[]')
aws ec2 authorize-security-group-ingress --group-id $CLUSTER2_SG --ip-permissions "PrefixListIds=[{PrefixListId=${PREFIX_LIST_ID1_IPV6}}],IpProtocol=-1"

```

### AWS Gateway API Controller 생성

새로운 컨트롤러를 두 번째 Kubernetes 클러스터 "c2"에 생성합니다 (첫 번째 클러스터를 생성하고 컨트롤러를 설치하는 것과 동일한 방식으로 설정합니다.).

편리하게 구성하도록 Kubernetes 구성 파일에 클러스터의 alias를 설정합니다.

```
#Cluster Alias 설정
aws eks update-kubeconfig --name c1 --region $AWS_REGION --alias c1
aws eks update-kubeconfig --name c2 --region $AWS_REGION --alias c2

#설정된 Cluster와 alias 확인
kubectl config get-contexts

```

두 번째 클러스터의 kubectl 컨텍스트를 사용 중인지 확인합니다.

```
kubectl config current-context

```

"c2"가 아닐 경우 아래와 같이, "c2" Cluster로 Switching 합니다.

```
kubectl config use-context c2

```

aws-application-networking-system 네임스페이스를 생성합니다.

```
kubectl create namespace aws-application-networking-system

```



```
aws ecr-public get-login-password --region us-east-1 | helm registry login --username AWS --password-stdin public.ecr.aws
helm upgrade gateway-api-controller \
oci://public.ecr.aws/aws-application-networking-k8s/aws-gateway-controller-chart \
--version=v1.0.5 \
--reuse-values \
--namespace aws-application-networking-system \
--set=defaultServiceNetwork=my-hotel

```
