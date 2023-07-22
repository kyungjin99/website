---
reviewers:
- timstclair
- deads2k
- liggitt
title: 노드 인가 사용하기
content_type: concept
weight: 90
---

<!-- overview -->
노드 인가는 kubelet에 의해 만들어진 API 요청을 특별히 인가하는 특수한 목적의 인가 모드이다.


<!-- body -->
## 개요

노드 인가자(authorizer)는 kubelet이 API 동작을 수행할 수 있도록 허가한다. 이는 다음을 포함한다.

읽기 작업:

* 서비스
* 엔드포인트
* 노드
* 파드
* kubelet의 노드에 바인딩 된 파드와 연관된 시크릿, 컨피그맵, 퍼시스턴트 볼륨 클레임과 퍼시스턴트 볼륨

쓰기 작업:

* 노드와 노드의 상태 (`NodeRestriction` 승인 플러그인을 사용하여 kubelet이 자신의 노드만 수정하도록 제한한다)
* 파드와 파드의 상태 (`NodeRestriction` 승인 플러그인을 사용하여 kubelet이 자신과 바인딩된 파드만 수정하도록 제한한다)
* 이벤트

인증 관련 작업:

* TLS 부트스트래핑을 위한 [CertificateSigningRequests API](/docs/reference/access-authn-authz/certificate-signing-requests/)에 대한 읽기/쓰기 권한
* 위임된 인증/권한 확인에 대해 TokenReviews 및 SubjectAccessReviews을 생성하는 기능

향후 릴리스에서 노드 인가자는 kubelet이 올바르게 작동하는 데 필요한
최소한의 권한 집합을 갖도록 권한을 추가하거나 제거할 수 있다.

노드 인증자에 의해 인증되기 위해 kubelet은 사용자 이름이 `system:node:<nodeName>`인
`system:node` 그룹에 속해 있음을 식별하는 자격증명을 사용해야 한다.
이 그룹 및 사용자 이름 형식은 [Kubelet TLS 부트스트래핑](/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/)의
일부로 각 kubelet에 대해 생성된 아이덴티티와 일치한다.

`nodeName` 값은 kubelet에 등록된 노드의 이름과 **정확하게 일치해야 한다**. 기본적으로 호스트 이름은 `hostname`으로 제공되거나 [kubelet 옵션](/docs/reference/command-line-tools-reference/kubelet/) `--hostname-override`를 통해 오버라이딩 된다. 그러나 `--cloud-provider` kubelet 옵션을 사용하는 경우, 특정 호스트 이름은 로컬의 `hostname` 과 `--hostname-override` 옵션을 무시한 채 클라우드 제공자에 의해 결정될 수 있다.
kubelet이 호스트 이름을 결정하는 방법에 대한 자세한 내용은 [kubelet options reference](/docs/reference/command-line-tools-reference/kubelet/)을 참조한다.

노드 인증자를 활성화하려면 `--authorization-mode=Node`로 apiserver를 시작한다.

kubelet이 작성할 수 있는 API 오브젝트를 제한하려면 `--enable-admission-plugins=...,NodeRestriction,...`로 API 서버를 시작해 [NodeRestriction](/docs/reference/access-authn-authz/admission-controllers#noderestriction) 승인 플러그인을 사용하도록 설정한다.

## 마이그레이션 시 고려 사항

### `system:nodes` 그룹 외부의 kubelet

`system:nodes` 그룹 외부의 kubelet은 `Node` 권한 부여 모드에 의해 인가되지 않으며,
현재 권한을 부여하는 어떠한 메커니즘을 통해서라도 계속해서 권한을 부여받아야 할 것이다.
노드 승인 플러그인은 이러한 큐블릿의 요청을 제한하지 않는다.

### 구별되지 않는 사용자 이름을 갖는 kubelet

일부 배포에서 kubelet은 `system:nodes` 그룹에 자신을 배치하는 자격증명을 가지고 있지만
`system:node:...` 형식의 사용자 이름이 없기 때문에
연관된 특정 노드를 식별하지 못한다.
이러한 kubelet은 `Node` 인가 모드에 의해 승인되지 않으며,
현재 권한을 부여하는 어떠한 메커니즘을 통해서라도 계속해서 권한을 부여받아야 할 것이다.

기본 노드 식별자 구현에서는 노드의 아이덴티티를 고려하지 않기 때문에
`NodeRestriction` 승인 플러그인은 이러한 kubelet의 요청을 무시할 것이다.

### RBAC를 사용하여 이전 버전에서 업그레이드하기

[RBAC](/docs/reference/access-authn-authz/rbac/)을 사용하여 업그레이드된 1.7 버전 이전의 클러스터에는 `system:nodes` 그룹 바인딩이 이미 존재하기 때문에 현재 상태 그대로 작동할 것이다.

클러스터 관리자가 API에 대한 노드 액세스를 제한하기 위해 `Node` 인가자와 `NodeRestriction` 승인 플러그인을 사용하기 시작하려는 경우,
다음 작업을 중단 없이 수행할 수 있다:

1. `Node` 인가 모드 (`--authorization-mode=Node,RBAC`)와 `NodeRestriction` 승인 플러그인을 활성화한다
2. 모든 kubelet의 자격증명이 그룹/사용자 이름 요구 사항을 준수하는지 확인한다
3. apiserver 로그를 감사(auditing)하여 `Node` 인가자가 kubelet의 요청을 거부하지 않는지 확인한다 (`NODE DENY` 메시지가 지속적으로 기록되지 않는지를 확인)
4. `system:node` 클러스터 역할 바인딩을 삭제한다

### RBAC 노드 허가

1.6 버전에서 `system:node` 클러스터 역할은 [RBAC 인가 모드](/docs/reference/access-authn-authz/rbac/)를 사용할 때 `system:node` 그룹에 자동으로 바인딩되었다.

1.7 버전에서는 노드 인가자가 시크릿 및 컨피그맵 액세스에 대한 추가적인 제한을 통해 동일한 작업을 수행할 수 있게 되면서
`system:nodes` 그룹을 `system:node` 역할에 자동으로 바인딩하는 것이 사용 중단(deprecated)되었다.

`Node`와 `RBAC` 인가 모드를 모두 사용하도록 설정한 경우, 1.7 버전에서는 `system:nodes` 그룹의 `system:node` 역할에 대한 자동 바인딩이 생성되지 않는다.

1.8 버전에서는 바인딩이 전혀 생성되지 않는다.

RBAC를 사용할 때는 다른 사용자 또는 그룹을 해당 역할에 바인딩하는 배포 방법과의 호환성을 위해
`system:node` 클러스터 역할이 계속 생성된다.

