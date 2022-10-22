---
title: AWS EKS Worker Node에 Customized된 설정 적용하기
author: garit
date: 2022-10-20 10:00:00 +0900
categories: [AWS, EKS]
tags: [aws, eks]
render_with_liquid: false
---

## AWS EKS Worker Node에 Customized된 설정 적용하기

### 상황

push 발송 등으로 인해 동접자 수 증가 대응을 위해 Nignx 기반의 서비스의 worker_conneections 값과 OS 커널 파라미터 수정이 필요하다.
>> 즉, EKS Worker Node의 파라미터를 변경해야 한다.

[변경할 파라미터 값]
- /etc/sysctl.conf
>> fs.file-max = 3244429
- etc/security/limits.conf
>> ec2-user  soft  nofile  4096
>> ec2-user  hard  nofile  10240



### EKS Node Group 시작템플릿 변경하기

- 시작템플릿을 변경하는 이유?
  : Worker Node 내에 Customized된 설정을 적용하기 위해서는 AWS EC2에서 제공하는 UserData라는 기능을 사용하여 인스턴스 시작 시 원하는 명령어가 실행될 수 있도록 해야 한다.
    그렇기 때문에 현재 Node Group의 시작템플릿에 UserData를 적용한 새로운 버전의 템플릿으로 업데이트하거나 템플릿 자체를 새로 생성하여 Worker Node를 재생성해야 한다.
    (1)시작 템플릿 버전 업데이트는 사용자 지정 시작 템플릿을 사용 하였을때만 가능
    (2)EKS Optimized AMI를 사용하는 경우 새로운 시작 템플릿을 생성하여 적용

- 나의 경우 (2)에 해당하므로 새로 UserData를 적용한 새로운 시작 템플릿을 생성하였다.

[과정]

1. 새로운 시작 템플릿을 생성한다.
- EC2 대쉬보드 -> 인스턴스 -> 시작 템플릿 -> 시작 템플릿 생성
- AMI는 현재 사용 중인 EKS Optimized AMI를 선택 및 나머지 세부 사항은 기존 설정 그대로
- 고급세부정보 -> 사용자 데이터(UserData)에 bootstrap.sh  스크립트가 실행될 수 있도록 필요한 명령어와 함께 입력 -> 생성
- bootstrap.sh 스크립트가 실행될 수 있도록 입력하는 이유는 생성되는 인스턴스가 EKS 클러스터에 조인되도록 하기 위함이다.

```bash
MIME-Version: 1.0
Content-Type: multipart/mixed; boundary="==MYBOUNDARY=="

--==MYBOUNDARY==
Content-Type: text/x-shellscript; charset="us-ascii"

#!/bin/bash

echo 'fs.file-max = 3244429' >> /etc/sysctl.conf
echo 'ec2-user  soft  nofile  4096' >> /etc/security/limits.conf
echo 'ec2-user  hard  nofile  10240' >> /etc/security/limits.conf

set -ex
/etc/eks/bootstrap.sh {클러스터 이름} --apiserver-endpoint {EKS API server endpoint}  --b64-cluster-ca {Certificate authority}

--==MYBOUNDARY==--
```

2. 방금 새로 생성한 시작 템플릿을 사용하는 새로운 노드 그룹을 생성한다.
- EKS 대쉬보드 -> 클러스터 -> 컴퓨팅 -> 노드 그룹 -> 노드 그룹 추가
- 시작 템플릿 사용 -> 위에서 생성한 시작 템플릿 지정 및 세부 사항 입력 -> 생성

3. 새로운 노드 그룹의 노드가 모두 Ready 상태가 된 것을 확인한 후 필요하면 이전 노드 그룹을 삭제한다.
4. 적용된 커널 파라미터 확인
```bash
cat /etc/sysctl.conf
cat /etc/security/limits.conf
```


### 주의할 점

Node Group에 적용된 시작 템플릿과 해당 Node Groupd의 AutoScalingGroup(이하 ASG)이 사용하는 시작 템플릿이 별개로 있는데,
변경해야 할 시작 템플릿은 Node Group 대쉬보드에서 바로 보이는 시작 템플릿이고 ASG에 할당된 시작 템플릿은 변경하지 않아야 한다.
왜냐하면 ASG는 Managed Node Group 설정에 따라 EKS에서 자동으로 생성/변경하므로 사용자의 Custom한 변경이 이루어지지 않도록 AWS측에서 권장하기 때문이다.


- 참고
  - [https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/eks-optimized-ami.html](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/eks-optimized-ami.html)
  - [https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/launch-templates.html#launch-template-custom-ami](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/launch-templates.html#launch-template-custom-ami)
  - [https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/create-managed-node-group.html](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/create-managed-node-group.html)
  - [https://aws.amazon.com/ko/premiumsupport/knowledge-center/eks-resolve-node-group-errors-in-cluster/](https://aws.amazon.com/ko/premiumsupport/knowledge-center/eks-resolve-node-group-errors-in-cluster/)
  
