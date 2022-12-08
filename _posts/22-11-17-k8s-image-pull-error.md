---
title: K8s Pod 이미지 Pull 에러 해결방안
author: garit
date: 2022-11-17 10:40:00 +0900
categories: [AWS, EKS]
tags: [aws, eks, docker]
render_with_liquid: false
---

## K8s Pod 이미지 Pull 에러 해결방안

### 상황
Jenkins 파이프라인을 지우고 다시 생성하면서 버전이 같은 이미지가 생성되었다. ECR에서는 같은 버전의 이미지가 push 되면 기존에 있던 이미지를 untagged로 바꾸고 새로 push된 이미지를 해당 버전으로 표기하여 저장한다.  
그러나 EKS에서 해당 버전으로 배포를 다시 했더니 새로 빌드한 이미지가 아닌 예전에 생성된 이미지로 Pod가 배포되었다.  
ex) 22.11.01에 빌드한 ecr-test:0.1.2은 spring v2.2.2 -> 22.11.05에 빌드한 ecr-test:0.1.2는 spring v2.6.0  
    -> 그러나 22.11.05에 배포한 ecr-test:0.1.2 pod는 spring v2.2.2로 실행되고 있는 문제를 발견했다.  
	
### 원인
EKS Worker Node에 기존 ecr-test:0.1.2 버전의 이미지가 남아있었으며, Pod의 imagePullPolicy가 IfNotPresent로 되어 있었다.  
그래서 Pod는 ecr-test:0.1.2 버전의 이미지가 worker node의 docker에 있으므로 새로 ECR에서 pull 해오지 않고 예전에 pull 해왔던 기존 이미지를 사용하여 Pod를 배포했던 것이다.  

### 해결방안
1. 먼저 Pod(또는 Deployment)의 yaml에 imagePullPolicy를 Always로 바꿔준다.  

```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: test-pod
spec:
  containers:
  - name: spring
    image: ecr-test:0.1.2
    imagePullPolicy: Always
```
<br/>

그러나 imagePullPolicy를 Always로 해준다고 해도 docker 이미지의 Digest 값이 같으면 다운로드를 시도는 하나 실제로 다운로드 하지 않는 현상이 있다고 한다.  
그러므로 worker node에 pull 해온 이미지들을 주기적으로 지워줌으로써 이전 이미지를 계속 쓰는 일이 없도록 해야 할 것이다.  
<br/>
2. docker prune을 실행시키는 cron shell script를 사용하여 docker image를 주기적으로 지워준다.  
<br/>
EKS의 경우 Worker Node가 생성될 때 Node Group의 시작템플릿에 cron shell을 생성하는 명령어를 추가해놓으면 worker node가 생성될 때마다 해당 shell도 함께 적용된다.  

이미 사용하고 있는 시작템플릿이 있는 경우 새로운 버전을 생성하여 적용한다.  

- EKS 콘솔 접속 -> 클러스터 -> 노드그룹 -> 시작템플릿 -> 작업 -> 템플릿 수정(새 버전 생성) -> 이미지 및 기본 설정들은 기존 버전과 동일하게 -> 제일 하단의 사용자데이터(User data)에 아래와 같이 입력 -> 템플릿 생성  
```bash
MIME-Version: 1.0
Content-Type: multipart/mixed; boundary="==MYBOUNDARY=="

--==MYBOUNDARY==

#!/bin/bash -xe
EKS_REGION='ap-northeast-2'
EKS_CLUSTER_NAME='cluster-name'
EKS_API_SERVER_ENDPOINT='https://cluster-api-server-endpoint.gr7.ap-northeast-2.eks.amazonaws.com'
EKS_B64_CLUSTER_CA='Certificate authority'

/etc/eks/bootstrap.sh \
  --apiserver-endpoint $EKS_API_SERVER_ENDPOINT \
  --b64-cluster-ca $EKS_B64_CLUSTER_CA \
  --kubelet-extra-args '--node-labels=node-type=self,ng=kbland-ekcl-stg-ap' \
  $EKS_CLUSTER_NAME

# docker prune
sudo echo "0 7 * * 1 bash docker image prune" > /etc/cron.weekly/cleaner_image.sh && chmod +x /etc/cron.weekly/cleaner_image.sh

--==MYBOUNDARY==--
```   
<br/>

템플릿을 새로 생성한 후 EKS 콘솔에서 시작템플릿 버전을 위에서 생성한 버전으로 변경한다.  

그 후 ssh -i ec2-key.pem worker-node-ip 으로 worker node에 접속하여 해당 파일이 잘 생성됐나 확인한다.  
> cat /etc/cron.weekly/cleaner_image.sh

위와 같은 작업을 통해 해당 현상을 해결하였다.   

<br/><br/>

참고
- [https://kubernetes.io/ko/docs/concepts/containers/images/#%EC%9D%B4%EB%AF%B8%EC%A7%80-%ED%92%80-pull-%EC%A0%95%EC%B1%85](https://kubernetes.io/ko/docs/concepts/containers/images/#%EC%9D%B4%EB%AF%B8%EC%A7%80-%ED%92%80-pull-%EC%A0%95%EC%B1%85)
- [https://docs.docker.com/storage/storagedriver/](https://docs.docker.com/storage/storagedriver/)
