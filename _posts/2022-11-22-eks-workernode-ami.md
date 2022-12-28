---
title: EKS 기존 노드로 AMI를 생성하여 새로운 노드그룹 생성하기 
author: garit
date: 2022-11-22 10:00:00 +0900
categories: [AWS, EKS]
tags: [AWS, eks]
render_with_liquid: false
---

## EKS 기존 노드로 AMI를 생성하여 새로운 노드그룹 생성하기 

### 상황

AWS EKS를 사용하던 중 기존 노드(인스턴스)에 설치된 솔루션 및 설정들을 그대로 새 노드에 적용해야 하는 일이 생겼다.  
<br/>

애초에 노드 그룹을 생성할 때 템플릿에 필요한 설정들을 다 넣어놨으면 좋았겠지만, 중간에 추가된 것들이라 기존 노드로 이미지(ami)를 만들어서 새 노드그룹을 생성해야 한다.  
<br/>

### EKS 노드(인스턴스) 이미지(AMI) 생성하기

**1.아래의 경로로 인스턴스 상세 페이지로 접속한다.**  
> EKS 콘솔 -> 클러스터 -> 컴퓨팅 -> 복사하고자 하는 노드 선택 -> 인스턴스 ID 클릭 -> 인스턴스 상세 페이지 진입  
<br/>

**2.인스턴스로 이미지(AMI)를 생성한다.**  
> 우측 상단의 작업 버튼 클릭 -> 이미지 및 템플릿 클릭 -> 이미지 생성 클릭 -> 이미지 이름, 설명, 태그 등 추가하고 생성하기 클릭  
<br/>

![ami1](https://user-images.githubusercontent.com/67899732/209769079-c8ea707b-d25b-46bc-b0f5-ca3831bec6f8.png)


**3.생성된 이미지(AMI)를 확인한다.**  
> EC2 콘솔 -> 이미지 -> AMI 접속 -> 위에서 설정한 AMI 이름으로 검색하여 생성된 이미지 확인  
<br/>


### 기존 노드의 AMI로 새 시작템플릿 생성하기

**1.다음과 같은 경로에서 시작템플릿을 생성할 수 있다.**  
> EC2 콘솔 -> 인스턴스 -> 시작템플릿 -> 시작템플릿 생성   


**2.시작템플릿 이름 및 설명 등을 작성하고 '애플리케이션 및 OS 이미지(Amazon Machine Image)'에서 위에서 생성한 AMI를 선택한다.**  
> 내 AMI 클릭 -> 위에서 생성한 AMI 이름 검색 -> 선택 -> 나머지 사항 설정 -> 시작템플릿 생성 버튼 클릭  

![ami2](https://user-images.githubusercontent.com/67899732/209770402-c66a8753-94e1-4ec3-acf4-31f7f8e80978.png)


이렇게 EKS 노드그룹을 생성할 때 사용할 시작템플릿을 만들었으니 이제 이 시작템플릿을 가지고 새 노드그룹만 만들면 된다.  
<br/>


### 새 노드그룹 생성하기

**1.다음과 같은 경로에서 노드그룹을 생성할 수 있다.**  
> EKS 콘솔 -> 클러스터 -> 컴퓨팅 -> 노드그룹 -> 우측 상단 노드그룹 추가 클릭  
<br/>

**2.시작템플릿 사용을 활성화하여 위에서 만든 시작템플릿을 선택한다.**  
> 노드그룹 이름, 역할을 설정한 뒤 시작템플릿을 선택한다. 그 후 여러 세부사항들을 설정하고 노드그룹을 생성한다.

![ami3](https://user-images.githubusercontent.com/67899732/209772530-cdee5a1f-d009-4379-8b02-4eeccbe85908.png)

**3.노드 생성 후 노드에 접속하여 원하는 설정들이 포함됐는지 확인한다.**  
> $ ssh -i EC2-key.pem workernode-ip  
<br/>


### 주의사항

새 노드를 생성한 후 해당 노드에 Pod를 배포하면 다음과 같은 에러가 발생한다.  
> Error from server: no preferred addresses found; known addresses: []

해당 시작템플릿은 기존 노드의 모든 것들을 가져왔기 때문에 kubelet 설정도 기존 노드의 IP로 되어있어 발생하는 에러다.  

그러므로 새 노드에 접속하여 아래 파일을 변경해줘야 한다.  
> $ ssh -i EC2-key.pem workernode-ip
> $ vi /etc/systemd/system/kubelet.service.d/10-kubelet-args.conf
> Enviroment='KUBELET_ARGS=--node-ip="이 부분에 적혀있는 IP를 새 노드 IP로 변경해줘야 한다"  
<br/> 

변경 후 ```systemctl restart kubelet```을 해준다.  
<br/>

그 후 Pod를 재배포 해보면 에러가 발생하지 않고 제대로 배포된 것을 확인할 수 있다.  
<br/>

