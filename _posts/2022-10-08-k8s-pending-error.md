---
title: AWS NLB로 배포한 K8s Service가 Pending 상태에서 지워지지 않는 문제
author: garit
date: 2022-10-08 11:00:00 +0900
categories: [Kubernetes, Errors]
tags: [k8s, aws]
render_with_liquid: false
---

## AWS NLB로 배포한 K8s Service가 Pending 상태에서 지워지지 않는 문제 해결 

### 상황

AWS Load Balancer Controller를 사용하기 위해 Service를 LB 타입(AWS NLB)으로 배포하였으나, External IP가 생성되지 않으면서 Pending 상태에서 멈춰있음

### 시도

```bash
kubectl delete svc -n kube-system aws-load-balancer-service --force
```
> 결과: 지워지지 않음

```bash
kubectl edit svc -n kube-system aws-load-balancer-service
> Type을 NodePort, ClusterIP로 변경
> 지정된 온갖 Annotation을 지움
> 그 후 delete 시도
```
> 결과: 지워지지 않음


### 해결법

```bash
kubectl edit svc -n kube-system aws-load-balancer-service
> finalizer를 지움
> 그 후 delete 시도
```
> 결과: 드디어 지워짐!!!      


- 참고: [https://stackoverflow.com/questions/61931602/cannot-delete-kubernetes-service-with-no-deployment](https://stackoverflow.com/questions/61931602/cannot-delete-kubernetes-service-with-no-deployment)
