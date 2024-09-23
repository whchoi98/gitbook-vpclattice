# MultiCluster 구성

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

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

### **Step2.네트워크 보안 환경 구성**

**VPC Lattice 네트워크에서 트래픽을 수신하도록 Security Group을 구성합니다.**

VPC Lattice와 통신하는 모든 Pod가 VPC Lattice 관리형 PrefixList의 트래픽을 허용하도록 Security Group을 설정해야 합니다. [자세한 내용은 Security Group을 사용하여 자원에 대한 트래픽 제어 방법을 참조할 수 있습니다.](https://docs.aws.amazon.com/vpc/latest/userguide/VPC\_SecurityGroups.html)Lattice에는 IPv4 및 IPv6 PrefixList가 모두 있습니다.

```

CLUSTER2_SG=$(aws eks describe-cluster --name $CLUSTER2_NAME --output json| jq -r '.cluster.resourcesVpcConfig.clusterSecurityGroupId')
PREFIX_LIST_ID2=$(aws ec2 describe-managed-prefix-lists --query "PrefixLists[?PrefixListName=="\'com.amazonaws.$AWS_REGION.vpc-lattice\'"].PrefixListId" | jq -r '.[]')
aws ec2 authorize-security-group-ingress --group-id $CLUSTER2_SG --ip-permissions "PrefixListIds=[{PrefixListId=${PREFIX_LIST_ID2}}],IpProtocol=-1"
PREFIX_LIST_ID2_IPV6=$(aws ec2 describe-managed-prefix-lists --query "PrefixLists[?PrefixListName=="\'com.amazonaws.$AWS_REGION.ipv6.vpc-lattice\'"].PrefixListId" | jq -r '.[]')
aws ec2 authorize-security-group-ingress --group-id $CLUSTER2_SG --ip-permissions "PrefixListIds=[{PrefixListId=${PREFIX_LIST_ID1_IPV6}}],IpProtocol=-1"

```

### Step3.AWS Gateway API Controller 권한 설정 준비

새로운 컨트롤러를 두 번째 Kubernetes 클러스터 "c2"에 생성합니다 (첫 번째 클러스터를 생성하고 컨트롤러를 설치하는 것과 동일한 방식으로 설정합니다.).

구성의 편의성을 위해 Kubernetes 구성 파일에 클러스터의 alias를 설정합니다.

* Cluster1 (기존 구성된 Cluster) - c1&#x20;
* Cluster2 (신규 구성된 Cluster) - c2

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

이제 aws-gateway-controller를 설치 하기 위해, aws-application-networking-system 네임스페이스를 생성합니다.

```
kubectl apply -f https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/deploy-namesystem.yaml

```



Pod Identity(권장) 또는 IRSA(IAM Role for Service Account)를 기반으로 설정할 수 있습니다.

Gateway API Controller는 Pod Identity를 권장합니다.

1. **Pod Identity Agent 설정**

EKS Pod Identity를 사용하여 필요한 권한으로 컨트롤러의 Kubernetes Service Account을 구성합니다.

아래 AWS CLI를 사용해서 Pod Identity AddOn을 생성합니다.

```
aws eks create-addon --cluster-name $CLUSTER2_NAME --addon-name eks-pod-identity-agent --addon-version v1.0.0-eksbuild.1

```

아래 명령을 통해 각 노드별로 AddOn이 정상적으로 설치되었는 지 확인합니다.

```
kubectl get pods -n kube-system | grep 'eks-pod-identity-agent'

```

아래와 같이 결과가 출력됩니다.

```
$ kubectl get pods -n kube-system | grep 'eks-pod-identity-agent'
eks-pod-identity-agent-98hrb          1/1     Running   0          68s
eks-pod-identity-agent-mb9gv          1/1     Running   0          68s
eks-pod-identity-agent-mlh7k          1/1     Running   0          68s
```

aws-application-networking-system 네임스페이스를 설정합니다.

```
kubectl apply -f https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/deploy-namesystem.yaml

```

2. Kubernetes Service Account에 Role을 할당.

Service Account를 생성합니다.

```
cat >gateway-api-controller-service-account.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
    name: gateway-api-controller
    namespace: aws-application-networking-system
EOF
kubectl apply -f gateway-api-controller-service-account.yaml

```

aws eks create-pod-identity-association 명령을 사용하여 Service Account와 IAM 역할을 연결합니다.

```
aws eks create-pod-identity-association --cluster-name $CLUSTER2_NAME --role-arn $VPCLatticeControllerIAMRoleArn --namespace aws-application-networking-system --service-account gateway-api-controller

```

### Step4. AWS Gateway Controller 설치 

AWS Gateway API 컨트롤러를 설치합니다.

컨트롤러를 배포하려면 kubectl 또는 Helm 중 하나를 실행합니다.&#x20;

아래와 같이 kubectl 을 통해 설치합니다.

```
kubectl apply -f https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/deploy-v1.0.5.yaml

```

정상적으로 Gateway API Controller가 설치되었는지 확인합니다.

```
kubectl --namespace aws-application-networking-system get pods

```

아래 명령어를 통해서 Gateway Class를 설치합니다.

```
kubectl apply -f https://raw.githubusercontent.com/aws/aws-application-networking-k8s/main/files/controller-installation/gatewayclass.yaml

```

Kubernetes Gateway 'my-hotel'을 생성합니다:

```
cd ~/environment/aws-application-networking-k8s/
kubectl apply -f files/examples/my-hotel-gateway.yaml

```

아래 명령을 통해 정상적으로 Programmed 되었는지 확인합니다.

```
kubectl get gateway

```

아래와 같은 결과가 출력됩니다.

```
$ kubectl get gateway
NAME       CLASS                ADDRESS   PROGRAMMED   AGE
my-hotel   amazon-vpc-lattice             True         11s
```



### Step5. 신규 서비스 설치

Service network에 Associsation 시킵니다.

```
source ~/.bash_profile
aws vpc-lattice create-service-network-vpc-association --service-network-identifier ${my_hotel_sn_id} --vpc-identifier ${CLUSTER2_VPC_ID}
aws vpc-lattice list-service-network-vpc-associations --vpc-id ${CLUSTER2_VPC_ID} | jq -r '.items[].status'

```

c2 클러스터에 "inventory-ver2" 서비스를 배포합니다.

```
cd ~/environment/aws-application-networking-k8s/
kubectl apply -f files/examples/inventory-ver2.yaml

```

"inventory-ver2"는 아래와 같은 menifest 파일로 구성되어 있습니다.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inventory-ver2
  labels:
    app: inventory-ver2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: inventory-ver2
  template:
    metadata:
      labels:
        app: inventory-ver2
    spec:
      containers:
      - name: inventory-ver2
        image: public.ecr.aws/x2j8p8w7/http-server:latest
        env:
        - name: PodName
          value: "Inventory-ver2 handler pod"


---
apiVersion: v1
kind: Service
metadata:
  name: inventory-ver2
spec:
  selector:
    app: inventory-ver2
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8090
```

### Step6.Service Export/Import 구성

#### Service Export 하기

Kubernetes 클러스터 간에 리소스를 공유하기 위해, 두 번째 클러스터(c2)에 존재하는 "`inventory-ver2` " 리소스를 첫 번째 클러스터의 `HTTPRoute`에서 참조할 수 있도록 export 합니다.

```
kubectl apply -f files/examples/inventory-ver2-export.yaml

```

"inventory-ver2-export.yaml" 에 대한 내용은 아래와 같습니다.&#x20;

```
apiVersion: application-networking.k8s.aws/v1alpha1
kind: ServiceExport
metadata:
  name: inventory-ver2
  annotations:
    application-networking.k8s.aws/federation: "amazon-vpc-lattice"
    
```

&#x20;`Kind: ServiceExport`는 Kubernetes 클러스터 간 서비스 공유를 위한 리소스입니다. 이 리소스를 사용하면 한 클러스터의 서비스를 다른 클러스터에서 참조할 수 있습니다.

`ServiceExport`의 주요 사용 방법은 다음과 같습니다:

1. **서비스 내보내기(**`ServiceExport)`:
   * 서비스를 다른 클러스터에서 사용할 수 있도록 `ServiceExport` 리소스를 생성합니다.
   * `ServiceExport`는 내보낼 서비스의 네임스페이스와 이름을 지정합니다.
2. **서비스 참조**:
   * 다른 클러스터에서 `ServiceExport`로 내보낸 서비스를 참조하려면 `ServiceImport` 리소스를 생성합니다.
   * `ServiceImport`에는 참조할 서비스의 네임스페이스, 이름, 클러스터 도메인 등의 정보를 포함합니다.
3. **네트워크 구성**:
   * `ServiceExport`와 `ServiceImport`를 사용하려면 클러스터 간 네트워크 연결이 필요합니다.
   * 이를 위해 VPC 피어링, VPN,또는 Kubernetes 서비스 메시 등의 네트워크 구성이 필요합니다.
   * 이 랩에서는 VPC Lattice를 사용합니다.
4. **동기화 및 상태 관리**:
   * `ServiceExport`로 내보낸 서비스의 상태 변경이 있으면 `ServiceImport`에 자동으로 반영됩니다.
   * 이를 통해 클러스터 간 서비스 정보와 엔드포인트가 일관성 있게 유지됩니다.

사용 사례:

* 마이크로서비스 아키텍처에서 클러스터 간 서비스 공유
* 하이브리드 클라우드 환경에서 온-프레미스 및 클라우드 서비스 연계
* 다중 리전/가용 영역 배포에서 서비스 가용성 향상

이처럼 `ServiceExport`와 `ServiceImport`는 Kubernetes 클러스터 간 서비스 공유와 상호 운용성을 높이는 데 사용됩니다.

#### Service Import 하기

Service Import는 c1 클러스터에서 구성해야 합니다. 아래와 같이 c1 Cluster로 전환합니다.

```
kubectl config use-context c1
kubectl config current-context

```

Kubernetes 클러스터 간에 리소스를 공유하기 위해, 두 번째 클러스터 (c2) 의 "`inventory-ver2` " 리소스를 첫 번째 클러스터의 `HTTPRoute`에서 참조할 수 있도록 import 합니다.

```
kubectl apply -f files/examples/inventory-ver2-import.yaml

```

"inventory-ver2-import"에 대한 mainfest는 아래와 같습니다.

```
apiVersion: application-networking.k8s.aws/v1alpha1
kind: ServiceImport
metadata:
  name: inventory-ver2
spec:
  type: ClusterSetIP
  ports:
  - port: 80
    protocol: TCP
```

HTTPRoute의 인벤토리 규칙을 업데이트하여 트래픽의 10%는 첫 번째 클러스터로, 90%는 두 번째 클러스터로 라우팅합니다.

```
kubectl apply -f files/examples/inventory-route-bluegreen.yaml

```

inventory-route-bluegreen.yaml 의 mainfest 파일 내용은 아래와 같습니다.

첫 번째 클러스터의 HTTPRoute 리소스를 수정했습니다.

* backendRefs 섹션에 두 개의 항목을 추가합니다.
  * inventory-ver2: 가중치 10, 첫 번째 클러스터의 리소스를 참조
  * inventory-ver1: 가중치 90, 두 번째 클러스터의 리소스를 참조
* 이를 통해 트래픽의 10%가 첫 번째 클러스터로, 90%가 두 번째 클러스터로 라우팅됩니다.

```
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: inventory
spec:
  parentRefs:
  - name: my-hotel
    sectionName: http
  rules:
  - backendRefs:
    - name: inventory-ver1
      kind: Service
      port: 80
      weight: 10
    - name: inventory-ver2
      kind: ServiceImport
      weight: 90
      
```

변경 사항을 적용하고 테스트합니다.

* 업데이트된 HTTPRoute 리소스를 첫 번째 클러스터에 적용합니다.
* `/inventory` 엔드포인트로 요청을 보내어 트래픽 분산이 정상적으로 이루어지는지 확인합니다.
* parking, review 모두 확인해 봅니다.

이와 같은 방식으로 HTTPRoute의 인벤토리 규칙을 업데이트하면, 두 클러스터 간 트래픽 분배를 유연하게 조절할 수 있습니다. 이를 통해 점진적인 배포 전략을 구현할 수 있습니다.

아래와 같이 Parking의 트래픽의 10%는 첫 번째 클러스터로, 90%는 두 번째 클러스터로 라우팅 되는 지 확인해 봅니다.

inventoryFQDN을 설정합니다.

```
inventoryFQDN=$(kubectl get httproute inventory -o json | jq -r '.metadata.annotations."application-networking.k8s.aws/lattice-assigned-domain-name"')

```

아래와 같이 Traffic이 분산되는지를 확인해 봅니다.

```
kubectl exec deploy/parking -- sh -c 'for ((i=1; i<=30; i++)); do curl -s "$0"; done' "$inventoryFQDN"
kubectl exec deploy/review -- sh -c 'for ((i=1; i<=30; i++)); do curl -s "$0"; done' "$inventoryFQDN"

```

아래와 같이 결과를 확인 할 수 있습니다.

```
Requsting to Pod(inventory-ver2-7c844f4b89-pnhhh): Inventory-ver2 handler pod
Requsting to Pod(inventory-ver2-7c844f4b89-pnhhh): Inventory-ver2 handler pod
Requsting to Pod(inventory-ver2-7c844f4b89-pnhhh): Inventory-ver2 handler pod
Requsting to Pod(inventory-ver2-7c844f4b89-pnhhh): Inventory-ver2 handler pod
Requsting to Pod(inventory-ver2-7c844f4b89-pnhhh): Inventory-ver2 handler pod
Requsting to Pod(inventory-ver1-57b78749cb-64mtx): Inventory-ver1 handler pod
Requsting to Pod(inventory-ver2-7c844f4b89-pnhhh): Inventory-ver2 handler pod
Requsting to Pod(inventory-ver2-7c844f4b89-pnhhh): Inventory-ver2 handler pod
#Inventory-ver2 와 Inventory-ver1로 Blue/Green 형태로 분산되는 것을 확인
```

아래 파일을 변경하면서 적용해 봅니다.

```
#inventory-route-bluegreen.yaml
#weight를 조절해 봅니다.

apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: inventory
spec:
  parentRefs:
  - name: my-hotel
    sectionName: http
  rules:
  - backendRefs:
    - name: inventory-ver1
      kind: Service
      port: 80
      weight: 10
    - name: inventory-ver2
      kind: ServiceImport
      weight: 90
      
```

생성한 EKS 클러스터의 Kubernetes inventory 서비스를 동일한 AWS 계정에 있는 다른 VPC의 새 클러스터로 마이그레이션할 것입니다.

