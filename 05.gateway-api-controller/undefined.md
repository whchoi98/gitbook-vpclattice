---
description: 'Updtae : 2024.06.07'
---

# 기본 구성하기

## K8s에서 **service-to-service 구성**

이 예제에서는 단일 VPC에 단일 클러스터를 생성한 다음 GW API HTTP Route 2개(rate 및 inventory)와 EKS Service 3개(Parking, Review 및 Inventory)를 구성합니다.

<figure><img src="../.gitbook/assets/image (44).png" alt=""><figcaption></figcaption></figure>

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

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

### Step2. GW API에서 Gateway 생성&#x20;

아래 명령을 통해서 Gateway `my-hotel`  을 생성합니다.&#x20;

```
#cd ~/environment/aws-application-networking-k8s
#kubectl apply -f examples/my-hotel-gateway.yaml
kubectl apply -f ~/environment/aws-application-networking-k8s/files/examples/my-hotel-gateway.yaml 
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

### Step3. GW API에서 HTTP Route 생성

"Parking Service및 "Review Service"를 위한 HTTP Route - "Rate" 를 생성하고, K8s의 Pod와 Service를 생성합니다.

```
cd ~/environment/aws-application-networking-k8s/files/examples/
kubectl apply -f parking.yaml
kubectl apply -f review.yaml
kubectl apply -f rate-route-path.yaml

```

아래 명령으로 정상적으로 배포 되었는지 확인합니다.

```
kubectl get svc,pod,httproute

```

아래와 같이 배포된 것을 확인 할 수 있습니다.

```
$ kubectl get svc,pod,httproute
NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/parking      ClusterIP   172.20.120.122   <none>        80/TCP    6m49s
service/review       ClusterIP   172.20.114.155   <none>        80/TCP    6m47s

NAME                           READY   STATUS    RESTARTS   AGE
pod/parking-59c9fdb57f-5g86b   1/1     Running   0          6m49s
pod/parking-59c9fdb57f-cc9dz   1/1     Running   0          6m49s
pod/review-766b6c9bf5-9wf5m    1/1     Running   0          6m47s
pod/review-766b6c9bf5-sr4cg    1/1     Running   0          6m47s

NAME                                        HOSTNAMES   AGE
httproute.gateway.networking.k8s.io/rates               6m46s
```

아래는 Parking에 대한 manifest 파일 입니다.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: parking
  labels:
    app: parking
spec:
  replicas: 2
  selector:
    matchLabels:
      app: parking
  template:
    metadata:
      labels:
        app: parking
    spec:
      containers:
      - name: parking
        image: public.ecr.aws/x2j8p8w7/http-server:latest
        env:
        - name: PodName
          value: "parking handler pod"

#K8s의 Service를 사용합니다. 아래는 ClusterIP 타입으로 구성됩니다.
---
apiVersion: v1
kind: Service
metadata:
  name: parking
spec:
  selector:
    app: parking
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8090

```

아래는 "review"에 대한 manifest 파일의 내용입니다.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: review
  labels:
    app: review
spec:
  replicas: 2
  selector:
    matchLabels:
      app: review
  template:
    metadata:
      labels:
        app: review
    spec:
      containers:
      - name: aug24-review
        image: public.ecr.aws/x2j8p8w7/http-server:latest
        env:
        - name: PodName
          value: "review handler pod"


---
apiVersion: v1
kind: Service
metadata:
  name: review
spec:
  selector:
    app: review
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8090

```

아래는 "rate-route-path" 라는 HTTPRoute Path 입니다.

```
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: rates
spec:
  parentRefs:
  - name: my-hotel
    sectionName: http
  rules:
  - backendRefs:
    - name: parking
      kind: Service
      port: 80
    matches:
    - path:
        type: PathPrefix
        value: /parking
  - backendRefs:
    - name: review
      kind: Service
      port: 80
    matches:
    - path:
        type: PathPrefix
        value: /review
```

추가로 HTTPRoute `inventory`  manifest 파일을 배포합니다.

```
cd ~/environment/aws-application-networking-k8s/files/examples/
kubectl apply -f inventory-ver1.yaml
kubectl apply -f inventory-route.yaml

```

아래는 inventory-ver1에 대한 manifest 파일 입니다.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inventory-ver1
  labels:
    app: inventory-ver1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: inventory-ver1
  template:
    metadata:
      labels:
        app: inventory-ver1
    spec:
      containers:
      - name: inventory-ver1
        image: public.ecr.aws/x2j8p8w7/http-server:latest
        env:
        - name: PodName
          value: "Inventory-ver1 handler pod"


---
apiVersion: v1
kind: Service
metadata:
  name: inventory-ver1
spec:
  selector:
    app: inventory-ver1
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8090

```

아래는 "inventory-route" 라는 HTTPRoute Path 입니다.

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

```

아래 명령으로 정상적으로 배포되었는지 확인합니다.

```
kubectl get svc,pod,httproute

```

아래 명령과 같이 출력됩니다.

```
$ kubectl get svc,pod,httproute
NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/inventory-ver1   ClusterIP   172.20.172.83    <none>        80/TCP    2m26s
service/parking          ClusterIP   172.20.120.122   <none>        80/TCP    12m
service/review           ClusterIP   172.20.114.155   <none>        80/TCP    12m

NAME                                  READY   STATUS    RESTARTS   AGE
pod/inventory-ver1-5df5f5c976-4pnhq   1/1     Running   0          2m26s
pod/inventory-ver1-5df5f5c976-f46qs   1/1     Running   0          2m26s
pod/parking-59c9fdb57f-5g86b          1/1     Running   0          12m
pod/parking-59c9fdb57f-cc9dz          1/1     Running   0          12m
pod/review-766b6c9bf5-9wf5m           1/1     Running   0          12m
pod/review-766b6c9bf5-sr4cg           1/1     Running   0          12m

NAME                                            HOSTNAMES   AGE
httproute.gateway.networking.k8s.io/inventory               2m24s
httproute.gateway.networking.k8s.io/rates                   12m
```

httproute "rate"의 실제 Lattice Service DNS 주소를 확인해 봅니다.

```
kubectl get httproute rates -o yaml
k8s_rates_svc_dns=$(kubectl get httproute rates -o json | jq -r '.metadata.annotations."application-networking.k8s.aws/lattice-assigned-domain-name"')
echo "export k8s_rates_svc_dns=${k8s_rates_svc_dns}" | tee -a ~/.bash_profile
source ~/.bash_profile

```

httproute "inventory"의 실제 Lattice Service DNS 주소를 확인해 봅니다.

```
kubectl get httproute inventory -o yaml
k8s_inventory_svc_dns=$(kubectl get httproute inventory -o json | jq -r '.metadata.annotations."application-networking.k8s.aws/lattice-assigned-domain-name"')
echo "export k8s_inventory_svc_dns=${k8s_inventory_svc_dns}" | tee -a ~/.bash_profile
source ~/.bash_profile

```

inventory-ver1 에서 rate 로 서비스 요청을 해봅니다.

```
kubectl exec deploy/inventory-ver1 -- curl $k8s_rates_svc_dns/parking
kubectl exec deploy/inventory-ver1 -- curl $k8s_rates_svc_dns/review 

```

parking 에서 inventory-ver1 로 서비스 요청을 해봅니다.

```
kubectl exec deploy/parking -- curl $k8s_inventory_svc_dns

```

"parking', "rate", "inventory-ver1"의 실제 Lattice Service DNS 주소들을 복사해 둡니다.



1개의 HTTPRoute, K8s Service를 아래에서 처럼 manifest 파일을 생성해서 배포해 봅니다.

```
cat <<EoF > ~/environment/lattice-test-01.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lattice-test-01
  labels:
    app: lattice-test-01
spec:
  replicas: 2
  selector:
    matchLabels:
      app: lattice-test-01
  template:
    metadata:
      labels:
        app: lattice-test-01
    spec:
      containers:
      - image: whchoi98/network-multitool
        imagePullPolicy: Always
        name: lattice-test-01
        ports:
        - containerPort: 80
          protocol: TCP
        readinessProbe:
          tcpSocket:
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          tcpSocket:
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
---
apiVersion: v1
kind: Service
metadata:
  name: lattice-test-01
spec:
  selector:
    app: lattice-test-01
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: lattice-test-01
spec:
  parentRefs:
  - name: my-hotel
    sectionName: http
  rules:
  - backendRefs:
    - name: lattice-test-01
      kind: Service
      port: 80
      weight: 10
EoF

kubectl apply -f ~/environment/lattice-test-01.yaml
```



```
kubectl get httproute inventory -o yaml
k8s_lattice_test_01_svc_dns=$(kubectl get httproute lattice-test-01 -o json | jq -r '.metadata.annotations."application-networking.k8s.aws/lattice-assigned-domain-name"')
echo "export k8s_lattice_test_01_svc_dns=${k8s_lattice_test_01_svc_dns}" | tee -a ~/.bash_profile
source ~/.bash_profile

```

아래와 같이 다시 검증해 봅니다.

<pre><code>kubectl exec deploy/inventory-ver1 -- curl $k8s_rates_svc_dns/parking
kubectl exec deploy/inventory-ver1 -- curl $k8s_rates_svc_dns/review 
kubectl exec deploy/parking -- curl $k8s_inventory_svc_dns
<strong>
</strong></code></pre>

Client VPC의 EC2에서 확인을 해 봅니다. Cloud9에서 새로운 터미널을 Open하고 아래와 같이 접속합니다.

```
aws ssm start-session --target $InstanceClient1

```

앞서 복사해둔 "$k8s\_rates\_svc\_dns/parking" , "$k8s\_rates\_svc\_dns/review", "$k8s\_inventory\_svc\_dns" DNS 주소를 "InstanceClient1"에서 확인하고 접속해 봅니다.

```
nslookup $k8s_rates_svc_dns/parking
nslookup $k8s_rates_svc_dns/review
nslookup $k8s_inventory_svc_dns
nslookup $k8s_inventory_svc_dns
curl $k8s_rates_svc_dns/parking
curl $k8s_rates_svc_dns/review
curl $k8s_inventory_svc_dns
curl $k8s_lattice_test_01_svc_dns
```

### Step4. VPC Lattice Console 에서 확인

VPC Lattice Console에서 결과를 확인해 봅니다.

* "VPC Lattice" - "Service Network" - "my-hotel"

<figure><img src="../.gitbook/assets/image (45).png" alt=""><figcaption></figcaption></figure>

* "VPC Lattice" - "Service" - "Parking" - "Routing"

<figure><img src="../.gitbook/assets/image (46).png" alt=""><figcaption></figcaption></figure>

* "VPC Lattice" - "Target Groups"

<figure><img src="../.gitbook/assets/image (47).png" alt=""><figcaption></figcaption></figure>

* "VPC Lattice" - "Target Groups"-"k8s-default-parking-xxxx"

<figure><img src="../.gitbook/assets/image (48).png" alt=""><figcaption></figcaption></figure>
