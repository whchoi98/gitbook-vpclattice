---
description: 'Updtae : 2024.01.19'
---

# 기본 구성하기

## K8s에서 **service-to-service 구성**

이 예제에서는 단일 VPC에 단일 클러스터를 생성한 다음 HTTP Route 2개(rate 및 inventory)와 Service 3개(Parking, Review 및 Inventory-1)를 구성합니다.



### Step1. VPC Lattice Service Network 생성



AWS Gateway API Controller가 동작하려면 VPC Lattice Service Network가 필요합니다. DEFAULT\_SERVICE\_NETWORK 환경 변수가 지정되면 GW API Controller가 자동으로 서비스 네트워크를 구성합니다.&#x20;

AWS CLI를 사용하여 "my-hotel"이라는 이름으로 VPC Lattice 서비스 네트워크를 수동으로 생성할 수 있습니다.

```
# my-hotel service network 생성
aws vpc-lattice create-service-network --name my-hotel

# my-hotel service network sn id 환경 변수 저장
aws vpc-lattice list-service-networks | jq -r '.items[]| select(.name=="my-hotel") | .id'
echo "export my_hotel_sn_id=$(aws vpc-lattice list-service-networks | jq -r '.items[]| select(.name=="my-hotel") | .id')" | tee -a ~/.bash_profile
source ~/.bash_profile

# my-hotel service network과 EKS Cluster Association
aws vpc-lattice create-service-network-vpc-association --service-network-identifier ${my_hotel_sn_id} --vpc-identifier ${CLUSTER_VPC_ID}

```



{% hint style="warning" %}
VPC는 1개의 Service Network에만 Associaation 이 가능합니다. 앞서 LAB에서 Superapp에 Client VPC를 Assocation 시켰다면, 삭제 후 Association 해야 합니다.
{% endhint %}



```
aws vpc-lattice list-service-network-vpc-associations --vpc-id ${CLUSTER_VPC_ID} | jq -r '.items[].status'

```

"Key:status" 의  값이 "ACTIVE" 인 것을 확인하고 , 다음 단계를 확인합니다.

"VPC lattice" - "Service Networks" - "my-hotel" 에서도 확인이 가능합니다.

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

### Step2. GW API에서 Gateway 생성&#x20;

아래 명령을 통해서 Gateway `my-hotel`  을 생성합니다.&#x20;

```
cd ~/environment/aws-application-networking-k8s
kubectl apply -f examples/my-hotel-gateway.yaml

```

아래와 같은 manifest 파일 형식입니다.

```
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: my-hotel
spec:
  gatewayClassName: amazon-vpc-lattice
  listeners:
  - name: http
    protocol: HTTP
    port: 80
```

my-hotel Gateway가 PROGRAMMED 상태가 True인 상태로 생성되었는지 확인합니다.

```
kubectl get gateway

```

아래와 같이 출력됩니다.

```
$ kubectl get gateway
NAME       CLASS                ADDRESS   PROGRAMMED   AGE
my-hotel   amazon-vpc-lattice             True         99s
```

""P r및 "검토 서비스"에 대한 경로 일치 라우팅을 가질 수 있는 Kubernetes HTTPRoute "속도"를 만듭니다
