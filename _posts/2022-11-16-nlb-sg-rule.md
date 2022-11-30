---
title: EKS Node Group Security Group에 NLB health check용 ingress rule 생성 방지하기
author: garit
date: 2022-11-16 10:40:00 +0900
categories: [AWS, EKS]
tags: [aws, eks, nlb]
render_with_liquid: false
---

## EKS Node Group Security Group에 NLB health check용 ingress rule 생성 방지하기

### 상황

EKS Node Group의 Security Group을 살펴보던 도중 AWS NLB로 생성한 k8s Service의 NodePort들이 여러 개 생성되어 있는 것을 발견했다.  
심지어는 service를 새로 생성했을 때 sg의 ingress rule이 모자른다는 에러 메세지가 뜨기도 했다. (rule 개수 limit에 도달해서) 
그래서 k8s NodePort 범위인 30000 ~ 32767 포트 대역으로 열어주는 rule을 생성했음에도 불구하고 30000번대 포트의 ingress rule은 지속적으로 생성되었다.

### 해결 방안
AWS Load balancer controller를 사용하면 annotation을 추가하여 sg에 rule 추가되는 것을 방지할 수 있다.
> AWS에서는 ALB/NLB에 대한 오퍼레이션을 유연하게 지원하고자 AWS Load balancer controller라는 오픈소스 프로젝트를 통해 Kubernetes의 기본 service controller보다 AWS 리소스에 대해 더 다양한 기능을 제공하고 있다.
> AWS Load balancer controller 설치하기
> [https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/aws-load-balancer-controller.html](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/aws-load-balancer-controller.html)

 AWS Load balancer controller를 설치한 후 Service를 생성할 때 다음과 같이 'service.beta.kubernetes.io/aws-load-balancer-type: external' annotation을 추가하면 AWS Load balancer controller를 이용하여 NLB를 생성할 수 있다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nlb-sample-service
  namespace: nlb-sample-app
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: LoadBalancer
  selector:
    app: nginx
```

* AWS Load balancer controller를 사용하지 않으면 기본적으로 Kubernetes의 In-tree Service controller를 사용하여 AWS NLB를 생성한 것이어서 해당 annotation은 사용할 수 없다.

### 추가할 annotation
>   service.beta.kubernetes.io/aws-load-balancer-manage-backend-security-group-rules: "false"

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nlb-sample-service
  namespace: nlb-sample-app
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
	service.beta.kubernetes.io/aws-load-balancer-manage-backend-security-group-rules: "false"
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: LoadBalancer
  selector:
    app: nginx
```

위의 annotation을 추가 시 Controller가 Security group rule을 자동으로 추가/삭제하지 않기 때문에 꼭 Worker node의 Security group에 직접 Ingress rule을 넣어주어야 한다.

### 정리
Kubernetes의 In-tree service controller는 NLB와 관련된 Security group rule을 자동으로 추가/삭제한다. 자동으로 Rule이 추가/삭제되지 않도록 하기 위해서는 AWS Load balancer controller를 이용하여 NLB타입의 서비스를 생성해야 하고, 이 때 'service.beta.kubernetes.io/aws-load-balancer-manage-backend-security-group-rules: false' annotation을 추가하여 자동 제어 기능을 diable 할 수 있다.


**참고:** 
- [https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/service/nlb/](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/service/nlb/)
- [https://docs.aws.amazon.com/eks/latest/userguide/network-load-balancing.html](https://docs.aws.amazon.com/eks/latest/userguide/network-load-balancing.html)
- [https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/service/annotations/#manage-backend-sg-rules](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/service/annotations/#manage-backend-sg-rules)
