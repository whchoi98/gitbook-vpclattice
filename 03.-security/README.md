---
description: 'Update : 2024.01.10'
---

# 03. Security

VPC Lattice는 기본적으로 안전한 구조입니다. 공유하려는 서비스와 액세스를 제공하려는 VPC에 대해 명시해야 하기 때문입니다. Multi-Account 시나리오의 경우 AWS RAM(Resource Access Manager)를 사용하여 Account 경계를 넘어 Resource를 공유해야 합니다. VPC Lattice는 네트워크의 여러 계층에서 심층 방어 전략을 구현할 수 있는 프레임워크를 제공합니다.

* 첫 번째 계층은 서비스 네트워크와의 서비스 및 VPC 연결입니다. VPC 또는 특정 서비스가 서비스 네트워크에 연결되지 않은 경우 VPC의 클라이언트는 서비스에 액세스할 수 없습니다.
* 두 번째 계층은 서비스 네트워크에 대한 선택적 네트워크 수준 보안 보호를 제공합니다. 네트워크 수준의 기본 요소들로 시스템을 보호하려면 네트워크 ACL을 사용하거나 VPC에 보안 그룹을 배치하여 네트워크 연결을 처리하도록 선택할 수 있습니다. 이렇게 하면 VPC의 모든 리소스를 허용하는 대신 허용할 VPC의 리소스 그룹을 참조할 수 있습니다.
* 세 번째 계층은 서비스 네트워크 및 개별 서비스에 적용할 수 있는 VPC Lattice 인증 정책입니다. 일반적으로 서비스 네트워크의 인증 정책은 네트워크 또는 클라우드 관리자가 운영하며, 예를 들어 AWS Organizations에서 지정된 조직의 인증된 요청만 허용하는 등 엄격한 권한 부여를 구현합니다. 서비스의 인증 정책을 통해 서비스 소유자는 네트워크 또는 클라우드 관리자가 서비스 네트워크 수준에서 적용한 엄격한 권한 부여 규칙보다 더 제한적일 수 있는 세분화된 제어를 설정할 수 있습니다. VPC Lattice 인증 정책은 지정된 보안 주체가 서비스 그룹 또는 특정 서비스에 액세스할 수 있는지 여부를 제어하기 위해 서비스 네트워크 또는 서비스에 첨부하는 IAM 정책 문서입니다. 액세스를 제어하려는 각 서비스 네트워크 또는 서비스에 하나의 인증 정책을 연결할 수 있습니다.
* 인증 정책은 IAM identity-based 정책과 다릅니다. IAM identity-based 정책은 IAM 사용자, 그룹 또는 역할에 연결되며 해당 ID가 어떤 리소스에서 수행할 수 있는 작업을 정의합니다. 인증 정책은 서비스 및 서비스 네트워크에 연결됩니다. 승인이 성공하려면 인증 정책과 ID 기반 정책 모두에 명시적인 allow 명령문이 있어야 합니다.
* 서비스 네트워크 연결에 대한 VPC의 보안 그룹과 인증 정책은 모두 선택 사항입니다. 보안 그룹을 구성하지 않고 서비스 네트워크를 VPC에 연결하고 인증 유형을 NONE으로 설정하여 인증 정책을 사용하지 않을 수 있습니다.

이 실습에서는 다음과 같은 방법으로 인증 정책을 사용하여 서비스 네트워크 및 서비스에 대한 액세스를 보호하는 방법을 살펴보겠습니다.

1. Service Network에 엄격한 인증 정책을 적용하여 인증된 사용자만 허용
2. Service (Reservation, Parking)에 보다 세부적인 인증 정책을 적용합니다. 'LatticeWorkshop InstanceClient1'에서만 액세스를 허용하도록 "Reservation Service"를 구성하겠습니다. 그런 다음 'LatticeWorkshop InstanceClient2'에서 "Rate Target Group"에 대한 GET 및 "Payments Target Group"에 대한 POST를 허용하도록 Parking Service를 구성합니다.
3. AWS SigV4를 사용하여 'reservation' 서비스에 대한 http 요청에 서명.

<figure><img src="../.gitbook/assets/image (37).png" alt=""><figcaption></figcaption></figure>
