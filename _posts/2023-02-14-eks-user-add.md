---
title: EKS를 새로운 IAM 사용자로 관리하기 (You must be logged in to the server (Unauthorized) 에러 해결방법)
author: garit
date: 2023-02-14 14:00:00 +0900
categories: [AWS, EKS]
tags: [AWS, eks]
render_with_liquid: false
---

## EKS를 새로운 IAM 사용자로 관리하기 (You must be logged in to the server (Unauthorized) 에러 해결방법)

### 상황

AWS EKS를 생성한 A라는 사용자를 더이상 사용할 수 없게되어 새로 생성한 IAM 사용자인 B로 로그인하여 kubectl 명령어로 EKS 클러스터를 관리해야 하는 상황이 생겼다. (A 사용자는 아예 삭제하는 상황)  
<br/>

그러나 IAM 사용자 B를 생성할 때 모든 권한을 주는 AdministratorAccess를 주었는데도 kubectl 명령어를 입력하면 다음과 같은 에러가 발생했다.  
<br/>

> error: You must be logged in to the server (Unauthorized)
<br/>

### 원인

AWS 문서에 따르면 kubectl에 구성된 AWS Identity and Access Management(IAM) 개체가 Amazon EKS에서 인증되지 않은 경우 이 오류가 발생한다고 한다.  
<br/> 

### 해결법

먼저 다음 두 가지를 확인해야 한다.  
<br/>

- IAM 사용자 또는 역할을 사용하도록 kubectl 도구를 구성했는가.  
- IAM 개체가 aws-auth ConfigMap에 매핑되어 있는가.  
<br/>

1번 방법은 현재 EKS 클러스터에 접근하려는 사용자가 EKS 클러스터를 생성한 사용자일 때 사용하는 방법이다.  
<br/>

그러나 새로 생성한 사용자 B는 EKS 클러스터를 생성한 사용자가 아니기 때문에 2번 방법으로 해결하고자 한다.  
<br/>

1.  IAM 개체를 사용하여 AWS CLI를 구성했는지 확인한다. 쉘 환경에서 AWS CLI에 대해 구성된 IAM 개체를 보려면 다음 명령을 실행한다.  
> $ aws sts get-caller-identity
<br/>

다음과 같은 결과가 반환된다. 그럼 해당 Credential 정보가 사용자 A의 정보인지 확인한다.  
<br/>

```bash
{
    "UserId": "XXXXXXXXXXXXXXXXXXXXX",
    "Account": "XXXXXXXXXXXX",
    "Arn": "arn:aws:iam::XXXXXXXXXXXX:user/testuser"
}
```

2. EKS 클러스터에 있는 aws-auth ConfigMap에 IAM 사용자 B를 추가하여 EKS 클러스터에 접근할 수 있게 한다.  
<br/>

```bash
$ kubectl edit configmap aws-auth -n kube-system
```

현재 aws-auth ConfigMap의 경우 다음과 같이 IAM 역할만 등록되어 있었다.  
<br/>

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::xxxxxxxxxxx:role/EKS-Role
      username: system:node:{{EC2PrivateDNSName}}
```

그래서 아래와 같이 IAM 사용자를 mapUser 섹션에 추가해주고 저장한다.  
<br/>

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::xxxxxxxxxxx:role/EKS-Role
      username: system:node:{{EC2PrivateDNSName}}
  mapUsers: |
    - userarn: arn:aws:iam::xxxxxxxxxxx:user/user-b
      username: user-b
      groups:
        - system:masters
```

참고로 system:masters 그룹은 슈퍼 사용자 액세스를 통해 모든 리소스에 대한 모든 작업을 수행할 수 있는데 사용자 B는 앞으로 EKS 클러스터를 관리할 관리자이기 때문에 master 권한을 주었다.  
<br/>

위와 같이 aws-auth ConfigMap에 user를 추가하여 권한을 부여하면 kubectl 명령어로 EKS 클러스터를 관리할 수 있다.  
<br/>


참고 
- [https://aws.amazon.com/ko/premiumsupport/knowledge-center/eks-api-server-unauthorized-error/](https://aws.amazon.com/ko/premiumsupport/knowledge-center/eks-api-server-unauthorized-error/)
