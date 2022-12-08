---
title: EKS Autoscaling Group 세부 설정하기
author: garit
date: 2022-11-18 10:40:00 +0900
categories: [AWS, EKS]
tags: [aws, eks]
render_with_liquid: false
---

## EKS Autoscaling Group 세부 설정하기

### 상황
EKS 노드그룹을 생성하면 해당 노드그룹의 Autoscaling을 담당하는 Autoscaling Group이 함께 생성되며 수시로 Workernode를 체크해서 Node의 수를 조절한다. (노드그룹에 지정한 desired, min, max를 가지고)  
<br/>
그러나 어떠한 이유로 이미 생성된 노드가 변경되면 안 되는 상황이 있다. (ex. 보안 심의, 컷오버 등등)  
<br/>
그런 상황에서도 테스트는 이뤄져야 하고 테스트 도중 노드가 새로 생기고 사라지기도 하는데 그때마다 이미 결재를 다 받아놓은 노드가 삭제되면 아주 난감한 상황이 발생한다.  
<br/>
그래서 이미 결재 받은 노드 말고, 테스트 때문에 생긴 노드만 삭제되게 해야 했다.  
<br/>

### 해결방안
1.Autoscaling Group의 종료 정책을 변경한다.  
<br/>
기본적으로 Autoscaling Group의 종료 정책은 제일 먼저 생긴 즉, 제일 오래된 인스턴스부터 종료하게 되어있다.    
<br/>
그래서 이것을 최신 인스턴스로 변경하면 제일 나중에 생긴 인스턴스부터 종료한다.    

> EKS 콘솔 -> Cluster 선택 -> Node Group 선택 -> Autoscaling Group 선택 -> 세부정보 -> 제일 하단의 고급 구성 편집 클릭 -> 종료 정책에서 '가장 오래된 인스턴스' 또는 '가장 오래된 시작템플릿'을 삭제 -> '최신 인스턴스' 추가 후 저장  

<br/>

2.Autoscaling Group의 인스턴스 축소 보호를 설정한다.
- 인스턴스 축소 보호란
: Auto Scaling 그룹에 대한 인스턴스 축소 보호 설정을 변경하여 축소 시 해당 Amazon EC2 Auto Scaling이 새 인스턴스를 종료할 수 있는지 여부를 제어할 수 있다.  
  축소 보호가 활성화된 경우 새로 시작된 인스턴스는 기본적으로 축소 보호되지만 이미 생성된 인스턴스들은 따로 설정을 해야한다.  
  
- 기생성된 인스턴스에 축소 보호 활성화 하기
> EKS 콘솔 -> Cluster 선택 -> Node Group 선택 -> Autoscaling Group 선택 -> 인스턴스 관리 선택 -> 축소 보호를 설정하려는 인스턴스 선택 -> 우측 상단의 작업 버튼 클릭 -> '축소 보호 설정' 클릭
*만약 이미 축소 보호가 설정된 인스턴스의 경우에는 '축소 보호 설정' 버튼이 막혀있고, '축소 보호 제거' 버튼만 활성화 되어있다.  


<br/><br/>

참고
- [https://docs.aws.amazon.com/ko_kr/autoscaling/ec2/userguide/ec2-auto-scaling-instance-protection.html](https://docs.aws.amazon.com/ko_kr/autoscaling/ec2/userguide/ec2-auto-scaling-instance-protection.html)
