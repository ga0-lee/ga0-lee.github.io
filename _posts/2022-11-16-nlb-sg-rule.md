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
그래서 k8s NodePort 범위인 30000 ~ 32767 포트 대역으로 열어주는 rule을 생성했음에도 불구하고 ingress rule은 지속적으로 생성되었다.

### 해결 방안
